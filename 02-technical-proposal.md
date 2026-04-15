# Guardian — Technical Proposal
## Tích hợp với GenAI Gateway (Milestone 1)

**Version:** 0.1 | **Date:** 2026-04-15
**Audience:** Engineering team, Tech Lead, CISO

---

## 1. Bài toán cần giải quyết

GenAI Gateway hiện là proxy cho phép 20,000 engineer nội bộ truy cập external LLM (Claude, GPT). Hiện tại không có lớp bảo vệ nào kiểm tra:
- **PII**: engineer vô tình gửi thông tin cá nhân (tên, email, số điện thoại) tới external LLM
- **Thông tin nhạy cảm**: prompt hoặc response chứa nội dung được phân loại Secret/Top Secret theo policy CISO

**Guardian** sẽ giải quyết bằng cách cung cấp 2 guardrail checks cho mỗi LLM call.

### Constraints chính

| Constraint | Giá trị | Lý do |
|---|---|---|
| Max latency overhead | < 5% total LLM call time | Không làm degraded UX của 20K engineers |
| Peak throughput | ~340 req/sec | 20K users × 10 calls/hr × 3x burst × 2 checks |
| Availability | 99.9% | Đảm bảo GenAI Gateway không bị blocked khi Guardian down |
| PII coverage | JP / KR / TH / EN tối thiểu | Majority của user base |
| Info classification levels | 5: public → top_secret | Theo CISO policy |

---

## 2. Giải pháp đề xuất

### 2.1 Tổng quan

Guardian là **FastAPI server** đứng giữa GenAI Gateway và vLLM model runner. Cung cấp 2 endpoints:

```
POST /v1/check/input   — kiểm tra trước khi gửi tới LLM
POST /v1/check/output  — kiểm tra response từ LLM
```

Mỗi endpoint chạy **2 guards song song** (asyncio.gather):
- `pii_guard`: phát hiện và mask PII → SAFE / UNSAFE + masked_content
- `info_classifier`: phân loại mức độ bảo mật → public / internal_use_only / restricted / secret / top_secret

### 2.2 Kỹ thuật giải quyết latency

**Vấn đề:** External LLM call mất 500ms–5s. Thêm Guardian check làm tăng latency.

**Giải pháp: Parallel Input Check**

GenAI Gateway gửi request tới Guardian input check **đồng thời** với gửi prompt tới External LLM:

```
T=0ms   User request → GenAI Gateway
T=1ms   ├── Fork: Gửi → External LLM (500ms–5s)
        └── Fork: Gửi → Guardian /check/input (~100ms)

T=100ms Guardian trả kết quả
        ├── UNSAFE → Cancel LLM call, return block (user thấy phản hồi T=101ms)
        └── SAFE   → Chờ LLM response

T=500ms+ LLM response → Guardian /check/output (~50ms buffer)
T=550ms+ Response → User
```

**Kết quả:**
- Input check latency: **0ms** (ẩn hoàn toàn sau LLM call)
- Output check latency: **+50–100ms** (buffer stream trước khi release)
- **Tổng overhead: ~2–3% cho một 2s LLM call** — đạt constraint <5%

### 2.3 Streaming support

Với SSE streaming responses:
1. Buffer 256 tokens đầu tiên (~100ms)
2. Guardian validate chunk đầu → nếu SAFE, release stream cho user
3. Tiếp tục validate các chunk sau (async, không block stream)
4. Nếu vi phạm ở chunk sau → truncate stream, gửi thông báo

---

## 3. Tech Stack & Lý do lựa chọn

| Component | Lựa chọn | Alternatives đã xem xét | Lý do |
|---|---|---|---|
| **Guardian API** | FastAPI (Python) | Go, Node.js | Async native, ecosystem ML tốt, team familiarity |
| **Model Serving** | vLLM | Triton, TGI | Battle-tested, continuous batching, TGI đã maintenance mode |
| **PII Model** | DistilBERT fine-tuned + Presidio CJK | GLiNER, Llama Guard | 40–60ms CPU inference, Presidio mở rộng tốt cho JP/KR |
| **Info Class Model** | DistilBERT + LoRA fine-tune | Full fine-tune, GPT-based | LoRA: 10-20x less memory, zero inference overhead, 40–60ms |
| **Cache** | Redis | Memcached, local cache | Semantic cache support, distributed, team familiarity |
| **Internal comm.** | HTTP/2 (gRPC-compatible) | gRPC, REST HTTP/1 | Multiplexing, connection pooling, low overhead |
| **Observability** | OpenTelemetry + Prometheus | Datadog (cost), custom | Industry standard 2026, integrates với MLflow |
| **Orchestration** | Kubernetes + HPA | VM-based | Team đã có K8s, GPU autoscaling cần thiết |

### 3.1 Lý do chọn DistilBERT + LoRA cho cả 2 models

- **Latency:** 40–60ms trên CPU → đáp ứng budget tốt
- **Size:** 66M params → fit trên 1 GPU, không cần tensor parallelism
- **LoRA:** Fine-tune với 500–2000 samples/class (5K–10K total) — khả thi với data nội bộ hiện có
- **Inference cost:** LoRA adapters merge vào weights → **zero overhead so với base model**
- **Flexibility:** Có thể retrain khi policy thay đổi mà không cần full fine-tune

---

## 4. Model Details

### 4.1 PII Guard

**Architecture:** DistilBERT-base fine-tuned cho NER + Microsoft Presidio cho CJK extension

**Supported entity types (minimum):**

| Loại PII | Ngôn ngữ | Method |
|---|---|---|
| PERSON, EMAIL, PHONE | EN, JP, KR, TH | DistilBERT NER |
| ADDRESS, GPS | EN | DistilBERT NER |
| Japanese name (kanji/kana) | JP | Presidio + spaCy ja |
| Korean RRN (주민등록번호) | KR | Presidio + regex + checksum |
| Thai name | TH | Presidio + regex (no pre-trained model available) |
| CREDIT_CARD, IBAN | All | Presidio regex |

**Output:**
```json
{
  "result": "UNSAFE",
  "masked_content": "My name is [PERSON] and email is [EMAIL_ADDRESS]",
  "entities": [
    {"type": "PERSON", "start": 11, "end": 18, "score": 0.98},
    {"type": "EMAIL_ADDRESS", "start": 32, "end": 51, "score": 0.99}
  ]
}
```

**Training data cần thiết:** Labeled PII dataset nội bộ (JP/KR/TH focus). Có thể bootstrap từ synthetic data + validation.

### 4.2 Info Classifier

**Architecture:** DistilBERT-base + LoRA fine-tuned cho 5-class text classification

**5 classes theo CISO policy:**

| Level | Ví dụ nội dung |
|---|---|
| `public` | Thông tin đã công bố, docs public |
| `internal_use_only` | Tài liệu nội bộ, operational data |
| `restricted` | MID/UID, dữ liệu hành vi, tài chính chung |
| `secret` | Tên đầy đủ + email, địa chỉ, GPS, LINE ID |
| `top_secret` | System credentials, PAN+CVV, dữ liệu di truyền |

**Output:**
```json
{
  "result": "UNSAFE",
  "level": "secret",
  "confidence": 0.88,
  "probabilities": {
    "public": 0.01,
    "internal_use_only": 0.04,
    "restricted": 0.07,
    "secret": 0.88,
    "top_secret": 0.00
  }
}
```

**Training data cần thiết:** 500–2,000 labeled examples per class. Có thể dùng GPT-4 để generate synthetic data với validation từ CISO team.

---

## 5. Integration với GenAI Gateway

### 5.1 Changes cần thiết ở GenAI Gateway

GenAI Gateway cần thêm 2 hooks:

**Pre-LLM hook (input check):**
```python
# Pseudo-code
async def pre_llm_hook(request):
    # Gửi song song: LLM call + Guardian input check
    llm_task = asyncio.create_task(send_to_llm(request))
    guardian_task = asyncio.create_task(
        guardian_client.check_input(request.prompt, request.user_id)
    )

    guardian_result = await guardian_task

    if guardian_result.verdict == "UNSAFE":
        llm_task.cancel()
        return block_response(guardian_result)

    llm_response = await llm_task
    return llm_response
```

**Post-LLM hook (output check):**
```python
async def post_llm_hook(response):
    guardian_result = await guardian_client.check_output(response.content)

    if guardian_result.verdict == "UNSAFE":
        return block_response(guardian_result)

    return response
```

### 5.2 Configuration per API Key / Service

GenAI Gateway có thể configure Guardian behavior per consumer:

```yaml
guardian_policy:
  default:
    input_guards: ["pii", "info_classification"]
    output_guards: ["pii"]
    on_unsafe: "block"           # block | log_only | mask_and_forward
    timeout_ms: 800
    on_timeout: "fail_open"      # fail_open | fail_closed

  high_compliance_service:
    input_guards: ["pii", "info_classification"]
    output_guards: ["pii", "info_classification"]
    on_unsafe: "block"
    on_timeout: "fail_closed"
```

---

## 6. Reliability Design

### 6.1 Availability targets

| Component | Target | Strategy |
|---|---|---|
| Guardian API | 99.9% | 2+ pods, K8s restart policy |
| vLLM (PII) | 99.5% | 2+ replicas, HPA |
| vLLM (InfoClass) | 99.5% | 2+ replicas, HPA |
| Redis cache | 99.9% | 3-node cluster |

### 6.2 Graceful degradation

```
Guardian DOWN → GenAI GW fail-open + log all requests as "unguarded"
vLLM DOWN     → Guardian trả 503 → GW fail-open (tùy policy)
Redis DOWN    → Guardian bypass cache, direct vLLM call (latency tăng, service tiếp tục)
```

### 6.3 Guardian timeout policy

```
Per-guard timeout: 400ms
Total /check/input timeout: 600ms (hai guards song song)
GenAI GW → Guardian timeout: 800ms

Nếu timeout:
- PII guard: fail-open + warn log
- Info classification (level ≥ secret): fail-closed + block
```

---

## 7. Roadmap triển khai

### Phase 1 — Foundation (2 tuần)

- [ ] Scaffold Guardian FastAPI service (health check, basic routing)
- [ ] Integrate vLLM client (connection pool, retry logic)
- [ ] Implement `/v1/check/input` và `/v1/check/output` endpoints
- [ ] Redis cache integration (exact-match, TTL 1h)
- [ ] Unit tests + integration tests với vLLM mock
- [ ] Docker image + K8s deployment manifests

**Deliverable:** Guardian chạy được, kết nối vLLM, pass basic test suite.

### Phase 2 — GenAI Gateway Integration (1 tuần)

- [ ] Implement pre/post LLM hooks trong GenAI Gateway
- [ ] Parallel input check pattern
- [ ] Streaming output buffer + validate
- [ ] Circuit breaker + fail-open/fail-closed
- [ ] E2E testing với real LLM calls

**Deliverable:** Flow hoàn chỉnh: User → GW → Guardian → LLM → Guardian → User.

### Phase 3 — Observability & Hardening (1 tuần)

- [ ] OpenTelemetry instrumentation (traces + metrics)
- [ ] Prometheus metrics export
- [ ] Grafana dashboard (latency, cache hit rate, unsafe rate)
- [ ] Alert rules
- [ ] Load test (340 req/sec peak simulation)
- [ ] Shadow mode rollout (log-only trước khi enforce block)

**Deliverable:** Production-ready, observability hoàn chỉnh, đã load-tested.

### Phase 4 — Model Fine-tuning (song song với Phase 1–3)

- [ ] Thu thập + label training data (PII: 2K samples/lang; InfoClass: 1K/class)
- [ ] Fine-tune DistilBERT PII Guard
- [ ] Fine-tune DistilBERT + LoRA InfoClass
- [ ] Evaluation + CISO validation
- [ ] Deploy fine-tuned models vào vLLM

**Deliverable:** Models fine-tuned theo data nội bộ, CISO approved.

---

## 8. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Model accuracy thấp (false positive block engineers) | Medium | High | Shadow mode trước, threshold tuning Week 1 |
| PII coverage thiếu cho CJK | Medium | Medium | Presidio CJK extension + regex fallback |
| Training data không đủ cho InfoClass | Medium | Medium | GPT-4 synthetic data generation + CISO validation |
| vLLM cold start làm tăng latency | Low | Medium | Warm pods luôn chạy (minReplicas: 2) |
| Guardian down làm block GenAI GW | Low | High | Fail-open default + circuit breaker |
| Cache poisoning (wrong verdict cached) | Low | High | Short TTL (1h) + cache key hash content + policy version |

---

## 9. Câu hỏi cần CISO / Stakeholder trả lời trước Phase 1

1. **Block hay mask-and-forward?** Khi PII detected trong input — block request hay mask và vẫn forward tới LLM?
2. **Secret threshold:** Khi InfoClass trả về `secret` — block hay log-only? Cho tất cả services hay chỉ một số?
3. **Audit log:** Log content của UNSAFE requests không? (Chứa PII → GDPR implications)
4. **Training data approval:** Team có thể sử dụng production prompts (anonymized) để train không?
5. **Fail-open policy:** Khi Guardian down — có service nào bắt buộc fail-closed (block all) không?

---

## Câu hỏi kỹ thuật còn mở

1. GenAI Gateway hiện được viết bằng gì? (Python? Go?) — ảnh hưởng đến design của hooks
2. vLLM cluster đã có sẵn chưa, hay cần provision mới?
3. Redis cluster đã có trong infra không?
4. K8s cluster có GPU nodes chưa? GPU type gì (A100/L4/A10)?
5. Có requirement về data residency (model weights, logs phải ở region nào)?
