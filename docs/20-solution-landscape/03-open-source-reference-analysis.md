# Open-Source Reference Analysis for Guardian

**Scope:** Phân tích 3 repo guardrail open-source để quyết định pattern nào adopt/adapt/avoid khi xây Guardian.

**Repos phân tích:**
1. [protectai/llm-guard](https://github.com/protectai/llm-guard) — Apache 2.0, FastAPI server, scanner pipeline
2. [guardrails-ai/guardrails](https://github.com/guardrails-ai/guardrails) — Apache 2.0, validator framework, Hub registry
3. [NVIDIA-NeMo/Guardrails](https://github.com/NVIDIA-NeMo/Guardrails) — Apache 2.0, Colang DSL, IORails parallel engine

---

## TL;DR — Kết luận trước

**Primary reference: `llm-guard`** (kiến trúc giống nhất với Guardian)
**Secondary: `guardrails-ai`** (concurrent execution pattern)
**Tertiary: `NeMo`** (IORails parallel pattern, observability hooks)

**Chiến lược:**
- **KHÔNG import cả 3 như library** — có conflict với vLLM backend, thêm dependency không cần thiết
- **ADOPT patterns và interface design** — rebuild lightweight cho Guardian
- **Fork-and-modify `llm-guard`'s FastAPI scaffold** — tiết kiệm 1-2 tuần boilerplate

---

## 1. So sánh tổng quan

| Tiêu chí | llm-guard | guardrails-ai | NeMo Guardrails |
|---|---|---|---|
| **Use case gần Guardian** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **FastAPI server sẵn sàng** | ✅ (llm_guard_api) | ⚠️ Flask legacy | ✅ (nemoguardrails serve) |
| **vLLM integration** | ❌ Dùng transformers trực tiếp | ❌ Dùng HF pipeline | ⚠️ Qua OpenAI-compatible API |
| **Async/concurrent execution** | ✅ asyncio.to_thread | ✅ AsyncGuard native | ✅ IORails parallel |
| **Streaming support** | ❌ Limited | ✅ StreamRunner + chunking | ✅ LLMRails (IORails chưa support) |
| **PII detection built-in** | ✅ Anonymize scanner | ✅ Hub validator | ⚠️ Qua Nemotron |
| **Info Classification (hierarchical)** | ❌ Entity-based | ❌ Không có | ⚠️ Custom action needed |
| **Config-driven (YAML)** | ✅ scanners.yml | ✅ RAIL/Pydantic | ⚠️ Colang + YAML |
| **License** | Apache 2.0 | Apache 2.0 | Apache 2.0 |
| **Production users** | Nhiều startup, enterprise | Guardrails Pro customers | Amdocs, Lowe's, Tech Mahindra |
| **Maturity** | High | High | High |

---

## 2. Ý tưởng ADOPT (đem về Guardian)

### 2.1 Từ llm-guard — Scanner Protocol Interface ⭐ CRITICAL

**Source:** [`llm_guard/input_scanners/base.py`](https://github.com/protectai/llm-guard/blob/main/llm_guard/input_scanners/base.py)

Protocol-based design thay vì class inheritance — pluggable, type-safe, stateless:

```python
from typing import Protocol

class GuardianScanner(Protocol):
    async def scan(self, text: str, context: dict) -> ScanResult:
        ...

class ScanResult(BaseModel):
    sanitized_text: str           # masked hoặc original
    is_safe: bool
    risk_score: float              # 0.0–1.0
    metadata: dict                 # entities, level, confidence
```

**Tại sao tốt:**
- Thêm scanner mới không cần sửa core engine
- Test từng scanner độc lập
- Swap models không ảnh hưởng contract
- Threshold tuning per-deployment qua config

**Áp dụng:** Guardian định nghĩa `PIIScanner` và `InfoClassificationScanner` implement cùng Protocol. Future: thêm `PromptInjectionScanner`, `JailbreakScanner` không breaking changes.

---

### 2.2 Từ llm-guard — FastAPI Server Scaffold ⭐ CRITICAL

**Source:** [`llm_guard_api/app/app.py`](https://github.com/protectai/llm-guard/blob/main/llm_guard_api/app/app.py)

Cấu trúc đáng copy:

```python
# Pseudo-structure từ llm-guard adapt cho Guardian
app/
├── main.py                    # FastAPI app, routing
├── scanners/
│   ├── base.py                # Protocol definition
│   ├── pii_scanner.py         # PII detection + masking
│   └── info_class_scanner.py  # 5-level classification
├── clients/
│   └── vllm_client.py         # Replace transformers ← KHÁC llm-guard
├── config/
│   ├── loader.py              # YAML config parser
│   └── schemas.py             # Pydantic models
├── middleware/
│   ├── auth.py                # Bearer token
│   ├── rate_limit.py          # slowapi integration
│   └── metrics.py             # Prometheus
└── schemas.py                 # Request/Response Pydantic
```

**Patterns cần copy:**
- `asyncio.to_thread()` wrapper cho CPU-bound work (mặc dù Guardian dùng vLLM network call, pattern vẫn hữu ích cho Presidio regex)
- `scan_fail_fast` config — dừng ở scanner đầu tiên fail
- Per-endpoint timeout (`scan_prompt_timeout`)
- Lazy model loading để K8s probe không timeout

---

### 2.3 Từ guardrails-ai — AsyncGuard Concurrent Pattern ⭐ CRITICAL

**Source:** [`guardrails/async_guard.py`](https://github.com/guardrails-ai/guardrails/blob/main/guardrails/async_guard.py)

**Insight quan trọng:** Guardrails mặc định chạy validators tuần tự → chậm. AsyncGuard chạy song song theo JSON path.

Guardian phải học từ lỗi này: **không chạy PII + InfoClass tuần tự:**

```python
# ❌ SAI (sequential) — +100% latency
pii_result = await pii_scanner.scan(text)
info_result = await info_scanner.scan(text)

# ✅ ĐÚNG (parallel) — +0% latency so với scanner chậm nhất
pii_result, info_result = await asyncio.gather(
    pii_scanner.scan(text),
    info_scanner.scan(text),
)
```

**Source docs:** [Concurrency docs](https://www.guardrailsai.com/docs/concepts/concurrency)

---

### 2.4 Từ guardrails-ai — Per-Scanner `on_fail` Action

**Source:** [`validator_on_fail_actions`](https://www.guardrailsai.com/docs/concepts/validator_on_fail_actions)

Mỗi scanner declare hành vi khi fail:

```yaml
# Guardian scanner config
scanners:
  pii:
    model: vllm://pii-distilbert
    threshold: 0.85
    on_fail: mask           # mask | reject | log_only | escalate

  info_classification:
    model: vllm://infocls-lora
    threshold: 0.80
    on_fail_by_level:
      public: allow
      internal_use_only: allow
      restricted: log_only
      secret: reject
      top_secret: reject_and_alert
```

**Giá trị:** Policy thay đổi không cần sửa code. CISO update YAML → rollout qua config map.

---

### 2.5 Từ NeMo — IORails Parallel I/O Pattern

**Source:** NeMo v0.21+ IORails engine — claimed 33% latency improvement

**Insight áp dụng cho Guardian:**

```
Input phase (parallel):
  ├── PII scan (vllm-pii)        ┐
  └── InfoClass scan (vllm-cls)  ┘→ gather → merged verdict

Output phase (parallel):
  ├── PII scan (vllm-pii)        ┐
  └── InfoClass scan (vllm-cls)  ┘→ gather → merged verdict
```

**⚠️ Cảnh báo từ NeMo's experience:**
- IORails không support streaming (early release)
- Rail mutations gây race condition trong parallel mode → **chỉ dùng read-only rails khi parallel**
- Nếu Guardian cần masking (mutation), masking phải serial SAU detection parallel

**Áp dụng cho Guardian:**
```python
# Stage 1: Detection (parallel, read-only)
detections = await asyncio.gather(
    pii_detect(text),       # trả về entities, không masking
    info_classify(text),    # trả về level + confidence
)

# Stage 2: Transformation (serial, mutation)
if detections[0].is_unsafe:
    masked = apply_mask(text, detections[0].entities)
```

---

### 2.6 Từ NeMo — OpenTelemetry Hooks

NeMo đã có OTel integration từ đầu. Guardian adopt ngay từ Phase 1, không để observability sau:

- Per-rail latency breakdown (PII vs InfoClass)
- Policy violation metrics (rejected count, masked count)
- Model inference time vs orchestration overhead
- vLLM queue depth

---

## 3. Ý tưởng AVOID (đừng học theo)

### 3.1 AVOID: `llm-guard` — Transformers trực tiếp

**Vấn đề:** llm-guard gọi `transformers.pipeline()` cho mỗi request → load model vào memory mỗi lần, không reuse.

**Guardian:** Models chạy trên vLLM (continuous batching, shared GPU), Guardian chỉ gọi HTTP/gRPC qua client.

**Lý do quan trọng:** Với 20K engineers, load model inline = memory blow-up và không thể scale horizontally.

### 3.2 AVOID: `llm-guard` — Presidio cho Classification

**Vấn đề:** Presidio tốt cho entity detection (PERSON, EMAIL), KHÔNG phù hợp cho hierarchical classification (public → top_secret).

**Guardian:** Xây custom classifier (DistilBERT + LoRA) cho Info Classification. Giữ Presidio chỉ cho PII extension (CJK).

### 3.3 AVOID: `guardrails-ai` — Sequential Default Runner

**Vấn đề:** Default `Runner` chạy validators tuần tự — sẽ vỡ latency budget.

**Guardian:** Parallel từ day 1 (asyncio.gather). Không bao giờ chạy tuần tự 2+ scanners cho cùng một request.

### 3.4 AVOID: `guardrails-ai` — Naive Reask Loop

**Vấn đề:** Reask mechanism re-prompt LLM với error context → sau 3 lần reask, thêm 1K-3K tokens vào context, phá vỡ token economy.

**Guardian:** Không bao giờ re-prompt LLM khi scanner fail. Chỉ: mask, reject, hoặc log.

### 3.5 AVOID: `NeMo` — Colang 2.0 DSL cho stateless validation

**Vấn đề:** Colang designed cho dialogue flows (multi-turn conversations). Guardian là stateless I/O validation — Colang interpreter = unnecessary overhead.

**Guardian:** YAML + Python functions cho custom logic. Đơn giản, debug dễ, Git-friendly.

### 3.6 AVOID: `NeMo` — LLMRails dialogue engine

**Vấn đề:** LLMRails duy trì conversation state, message history. Guardian không cần.

**Guardian:** Mỗi request độc lập, stateless. Session-level tracking (nếu cần) để phase 2, tách biệt với core validation engine.

---

## 4. Key Files để team nghiên cứu

### 4.1 llm-guard (ưu tiên cao nhất — study kỹ)

| File | Purpose | Giá trị cho Guardian |
|---|---|---|
| `llm_guard_api/app/app.py` | FastAPI entry point | Scaffold structure, request handling |
| `llm_guard_api/app/config.py` | YAML config loader | Copy logic, adapt schema |
| `llm_guard/input_scanners/base.py` | Scanner Protocol | **CRITICAL** — copy interface design |
| `llm_guard/input_scanners/anonymize.py` | PII detection + masking | Adapt masking logic (thay transformers bằng vLLM client) |
| `llm_guard/output_scanners/sensitive.py` | Output PII validation | Pattern cho output check |
| `llm_guard/vault.py` | Reversible masking vault | Reference cho Guardian's masking strategy |
| `llm_guard_api/config/scanners.yml` | Example config | Reference format |

### 4.2 guardrails-ai (study vừa phải)

| File | Purpose | Giá trị cho Guardian |
|---|---|---|
| `guardrails/async_guard.py` | AsyncGuard concurrent pattern | **CRITICAL** — concurrent execution logic |
| `guardrails/validators/validator_base.py` | Validator base class | Reference cho Protocol design |
| `guardrails/runner.py` (StreamRunner) | Streaming chunking strategy | Reference cho streaming output validation |
| [guardrails_pii](https://github.com/guardrails-ai/guardrails_pii) | Presidio + GLiNER hybrid | Model selection reference |

### 4.3 NeMo Guardrails (study nhẹ)

| File | Purpose | Giá trị cho Guardian |
|---|---|---|
| `nemoguardrails/server/app.py` | FastAPI server pattern | Reference cho serve command |
| `nemoguardrails/rails/llm/iorails.py` | IORails parallel executor | Pattern cho parallel rail execution |
| `nemoguardrails/actions/actions_server.py` | Custom action registration | Reference cho plugin mechanism |

---

## 5. Đề xuất cụ thể cho Guardian Phase 1

Dựa trên phân tích trên, cập nhật Phase 1 implementation plan:

### 5.1 Week 1: Scaffold & Interface

- [ ] **Clone llm-guard locally** để tham khảo code
- [ ] **Copy FastAPI structure** từ `llm_guard_api/app/` (KHÔNG fork — rebuild cleanly)
- [ ] **Define `GuardianScanner` Protocol** dựa trên `llm_guard/input_scanners/base.py`
- [ ] **Define ScanResult, ScanRequest, ScanResponse** Pydantic models
- [ ] **Implement config loader** — YAML → Pydantic models
- [ ] **Setup Prometheus metrics + OTel skeleton**

### 5.2 Week 2: vLLM Client + Scanners

- [ ] **Build `VLLMClient`** (async HTTP pool, retry logic, circuit breaker)
- [ ] **Implement `PIIScanner`**:
  - Gọi vLLM PII model qua client
  - Masking logic (tham khảo `llm_guard/input_scanners/anonymize.py`)
  - Return `ScanResult` với `sanitized_text` + `entities`
- [ ] **Implement `InfoClassificationScanner`**:
  - Gọi vLLM InfoClass model
  - Map softmax probabilities → level + confidence
  - Return `ScanResult` với `metadata.level`
- [ ] **Parallel orchestrator** — `asyncio.gather(pii, info)` trong handler

### 5.3 Week 3: Integration + Testing

- [ ] **POST /v1/check/input, /v1/check/output** endpoints
- [ ] **Redis cache integration** (exact-match, TTL 1h)
- [ ] **Unit tests per scanner** (mock vLLM)
- [ ] **Integration test với real vLLM** (local dev instance)
- [ ] **K8s manifests** (với lazy-load + probe tuning từ llm-guard)

---

## 6. Risk assessment sau khi study 3 repos

| Risk | Phát hiện từ repo | Mitigation cho Guardian |
|---|---|---|
| Race condition khi parallel masking | NeMo IORails warning | Split detection (parallel) vs mutation (serial) |
| Sequential latency blow-up | guardrails-ai default runner | asyncio.gather từ day 1 |
| Reask loop phá token economy | guardrails-ai reask | Hard rule: never re-prompt LLM |
| Colang overhead cho stateless validation | NeMo experience | YAML + Python only, no DSL |
| Transformers memory blow-up ở scale | llm-guard pattern | vLLM backend, Guardian là client only |
| Model load time vs K8s probe timeout | llm-guard docs warning | Lazy-load + `initialDelaySeconds: 120s` |
| Presidio không fit hierarchical classification | llm-guard Sensitive scanner | Custom classifier, không dùng Presidio cho InfoClass |

---

## 7. Câu hỏi còn mở

1. **Vault cho reversible masking:** llm-guard dùng in-memory Vault — Guardian cần persistent (Redis)? Ai deprotect được masked content? (Privacy implication)
2. **Scanner governance:** Team ML/Security tự thêm custom scanner không? Nếu có, cần review process nào?
3. **Cache key includes policy version?** Khi update scanner threshold, cache có invalidate không?
4. **vLLM routing strategy:** 1 vLLM instance serve cả 2 models (memory efficient nhưng single point of failure) hay 2 instances riêng (isolation + scale độc lập)?
5. **Có nên fork llm-guard branch hay build from scratch?** Fork tiết kiệm thời gian nhưng gánh nặng maintain upstream sync. Build from scratch sạch hơn nhưng chậm 1-2 tuần.

---

## Sources

- [protectai/llm-guard](https://github.com/protectai/llm-guard)
- [llm-guard documentation](https://llm-guard.com/)
- [guardrails-ai/guardrails](https://github.com/guardrails-ai/guardrails)
- [Guardrails AI Concurrency docs](https://www.guardrailsai.com/docs/concepts/concurrency)
- [Guardrails Hub](https://guardrailsai.com/hub)
- [NVIDIA-NeMo/Guardrails](https://github.com/NVIDIA-NeMo/Guardrails)
- [NeMo Parallel Rails Tutorial](https://docs.nvidia.com/nemo/microservices/latest/guardrails/tutorials/parallel-rails.html)
- [NeMo Streaming Guide](https://docs.nvidia.com/nemo/guardrails/latest/user-guides/advanced/streaming.html)
- Detailed research reports:
  - `plans/reports/researcher-260415-2228-llmguard-adoption-analysis.md`
  - `plans/reports/researcher-260415-2228-guardrails-ai-analysis.md`
  - `plans/reports/researcher-260415-2228-nemo-guardrails-deep-dive.md`
