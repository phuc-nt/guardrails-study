# Guardian — Architecture Design

**Version:** 0.2 (Draft) | **Date:** 2026-04-30
**Scope:** Milestone 1 của **Guardrail Platform** toàn tập đoàn LY Corp — phục vụ 2 consumer: GenAI Gateway (B2E) + Agent i (B2C)

---

## 1. Tổng quan hệ thống

### 1.1 Context

**Guardian** là API server cung cấp guardrail service, là **nhóm tính năng đầu tiên** của Guardrail Platform toàn tập đoàn. Milestone 1 phục vụ song song 2 consumer:

- **GenAI Gateway** — proxy nội bộ giữa agent của ~20,000 engineer ↔ external LLM (Claude, GPT). Threat chính: PII / secret egress.
- **Agent i** — unified AI agent brand của LY Corp (Yahoo! JAPAN AI Assistant + LINE AI hợp nhất, 7 Domain Agents, có memory + task execution). Threat chính: PII end-user, harmful output, task-execution safety. Ref: https://www.lycorp.co.jp/en/news/release/020398/

→ Guardian thiết kế **multi-tenant** ngay từ Milestone 1: config / policy / scanner pack / audit log tách theo `tenant_id` (`genai-gw` | `agent-i` | …future).

**Hai guardrail models:**

| Model | Input | Output | Serving |
|---|---|---|---|
| **PII Guard** | Text | `SAFE` / `UNSAFE` + masked text | vLLM |
| **Info Classifier** | Text | `public` / `internal_use_only` / `restricted` / `secret` / `top_secret` + confidence score | vLLM |

**Scale target:**
- GenAI Gateway: 20,000 engineer peak (~340 req/sec sau ×2 input+output check)
- Agent i: consumer-facing, traffic forecast TBD; yêu cầu **streaming support** (SSE/chunk protocol) cho chat UX

---

### 1.2 Full System Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  Engineer / AI Agent                                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTP/SSE
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GenAI Gateway                                                       │
│                                                                      │
│  ┌─────────────────────┐    ┌──────────────────────────────────┐    │
│  │  Request Handler    │    │  Response Handler                 │    │
│  │  (pre-LLM hook)     │    │  (post-LLM hook)                  │    │
│  └──────────┬──────────┘    └──────────────┬─────────────────── ┘   │
└─────────────┼──────────────────────────────┼─────────────────────── ┘
              │                              │
    ┌─────────┴──────────┐        ┌──────────┴──────────┐
    │  [INPUT CHECK]      │        │  [OUTPUT CHECK]      │
    │  Guardian API       │        │  Guardian API        │
    └─────────┬──────────┘        └──────────┬──────────┘
              │                              │
    ┌─────────┴──────────────────────────────┴──────────┐
    │  Guardian (API Server)                             │
    │                                                    │
    │  ┌──────────────────┐  ┌───────────────────────┐  │
    │  │  PII Guard       │  │  Info Classifier       │  │
    │  │  endpoint        │  │  endpoint              │  │
    │  └────────┬─────────┘  └────────────┬───────────┘  │
    └───────────┼────────────────────────-┼──────────────┘
                │  gRPC / HTTP            │
                ▼                         ▼
    ┌───────────────────────────────────────────────────┐
    │  Model Runner (vLLM)                              │
    │                                                   │
    │  ┌──────────────────┐  ┌──────────────────────┐  │
    │  │  PII Model       │  │  InfoClass Model      │  │
    │  │  (DistilBERT/    │  │  (DistilBERT + LoRA   │  │
    │  │   GLiNER-based)  │  │   fine-tuned)         │  │
    │  └──────────────────┘  └──────────────────────┘  │
    └───────────────────────────────────────────────────┘

              │ (nếu input SAFE)
              ▼
    ┌─────────────────────┐
    │  External LLM       │
    │  (Claude / GPT)     │
    └─────────────────────┘
```

---

## 2. Thiết kế chi tiết: Input Check Flow

### 2.1 Parallel Pattern — Ẩn latency của Guardian sau LLM call

Kỹ thuật quan trọng nhất để không làm tăng latency người dùng: **chạy Guardian input check song song với việc gửi request tới external LLM**.

```
GenAI Gateway nhận request từ user
         │
         ├──────────────────────────────────────────┐
         │                                          │
         ▼                                          ▼
[Gửi request → External LLM]          [Guardian input check]
(Claude/GPT, 500ms–5s)                (PII + InfoClass, ~100ms)
         │                                          │
         │                              ┌───────────┴──────────┐
         │                              │  UNSAFE?             │
         │                              │  → Cancel LLM call   │
         │                              │  → Return block msg  │
         │                              │                      │
         │                              │  SAFE?               │
         │                              │  → Continue          │
         │                              └───────────┬──────────┘
         │                                          │
         └──────────────── join ────────────────────┘
                                │
                           LLM Response
```

**Kết quả:** Latency overhead = `max(guardian_latency, 0)` — Guardian xong trước LLM call → zero overhead.

**Xử lý edge case:** Nếu Guardian phát hiện UNSAFE sau khi LLM đã bắt đầu trả về → cancel LLM call, return block message.

---

### 2.2 Output Check Flow — Streaming-compatible

Với non-streaming response:
```
LLM Response → Guardian output check → SAFE → Trả về user
                                      → UNSAFE → Block, return safe message
```

Với streaming response (SSE / chunked):
```
LLM bắt đầu stream tokens
         │
         ▼
Buffer đủ N tokens (256 tokens / ~100ms)
         │
         ▼
Guardian validate chunk đầu tiên (~50ms)
         │
    SAFE?─────────────────────→ Release tokens to user
         │                             │
         │                    Async validate remaining chunks
         │                             │
         │                    Nếu vi phạm trong chunk sau:
         │                    → Block thêm output
         │                    → Send "response truncated" message
    UNSAFE?──────────────────→ Block toàn bộ, trả safe message
```

**First-token latency overhead:** ~50–100ms (chấp nhận được, so với LLM call 500ms–5s).

---

## 3. Guardian API Design

### 3.1 Endpoints

```
POST /v1/check/input
POST /v1/check/output
GET  /v1/health
```

### 3.2 Request/Response Schema

**POST /v1/check/input**

```json
// Request
{
  "request_id": "uuid",
  "content": "string (prompt text)",
  "user_id": "string (for session tracking & caching)",
  "guards": ["pii", "info_classification"],  // chọn guard cần check
  "context": {                                // optional
    "service": "genai-gateway",
    "model": "claude-3-5-sonnet"
  }
}

// Response — SAFE
{
  "request_id": "uuid",
  "verdict": "SAFE",
  "guards": {
    "pii": {
      "result": "SAFE",
      "latency_ms": 45
    },
    "info_classification": {
      "result": "SAFE",
      "level": "internal_use_only",
      "confidence": 0.92,
      "latency_ms": 52
    }
  },
  "total_latency_ms": 98
}

// Response — UNSAFE (PII detected)
{
  "request_id": "uuid",
  "verdict": "UNSAFE",
  "guards": {
    "pii": {
      "result": "UNSAFE",
      "masked_content": "My name is [PERSON] and my email is [EMAIL]",
      "entities": [
        {"type": "PERSON", "start": 11, "end": 19},
        {"type": "EMAIL", "start": 33, "end": 52}
      ],
      "latency_ms": 87
    },
    "info_classification": {
      "result": "UNSAFE",
      "level": "secret",
      "confidence": 0.88,
      "latency_ms": 54
    }
  },
  "total_latency_ms": 141
}
```

**POST /v1/check/output** — cùng schema, field `content` là LLM response text.

### 3.3 Behavior khi UNSAFE

| Guard | Verdict | GenAI Gateway action |
|---|---|---|
| PII UNSAFE | Masked content available | Có thể forward masked content hoặc block — do policy của GW quyết định |
| Info Class ≥ Secret | UNSAFE | Block, không forward tới external LLM |
| Info Class ≤ Restricted | UNSAFE | Log + alert, forward với warning header |
| Cả 2 UNSAFE | UNSAFE | Block, ưu tiên rule nghiêm ngặt hơn |

---

## 4. Guardian Internal Architecture

### 4.1 Component Diagram

```
Guardian (FastAPI)
├── /v1/check/input
│     ├── Request validation (Pydantic)
│     ├── Cache lookup (Redis) ──→ hit: return cached result
│     ├── Guard orchestrator
│     │     ├── PII Guard client ──→ vLLM /v1/completions
│     │     └── InfoClass client ──→ vLLM /v1/completions
│     │           (run in parallel via asyncio.gather)
│     ├── Result aggregation
│     ├── Cache write (Redis)
│     └── Response serialization
│
├── /v1/check/output  (same flow)
│
├── Redis Client (semantic cache + rate limiting)
├── vLLM Client pool (connection pooling, gRPC hoặc HTTP/2)
└── Observability (OpenTelemetry → Prometheus)
```

### 4.2 Concurrency Model

Guardian sử dụng **async Python (FastAPI + asyncio)**:
- Mỗi request: `asyncio.gather(pii_check, info_class_check)` — 2 model calls song song
- Connection pool tới vLLM: `aiohttp.ClientSession` với pool size = 50–100
- Không blocking I/O; throughput giới hạn bởi vLLM capacity, không phải Guardian

### 4.3 Caching Strategy

```
Request → Redis lookup (md5(content) key)
                │
         Cache HIT?─────────────→ Return cached result (< 1ms)
                │
         Cache MISS
                │
         Run vLLM inference (~50–150ms)
                │
         Write to Redis (TTL = 1 hour)
                │
         Return result
```

**Cache key:** `guardian:v1:{guard_type}:{md5(content)}`
**TTL:** 1 hour (guardrail kết quả không phụ thuộc vào thời gian)
**Expected hit rate:** 30–50% cho typical enterprise workload (nhiều engineer hỏi câu tương tự)

---

## 5. Model Runner (vLLM) Configuration

### 5.1 Deployment

Mỗi model chạy trên instance vLLM riêng (isolation tốt hơn, scale độc lập):

```
vllm-pii-guard:
  - model: pii-distilbert-guardian (fine-tuned)
  - tensor_parallel_size: 1  (không cần TP cho <3B)
  - max_num_seqs: 2048
  - max_num_batched_tokens: 16384
  - gpu_memory_utilization: 0.90

vllm-info-classifier:
  - model: infocls-distilbert-lora (fine-tuned LoRA)
  - tensor_parallel_size: 1
  - max_num_seqs: 2048
  - max_num_batched_tokens: 16384
  - gpu_memory_utilization: 0.90
```

### 5.2 Scale Estimation (20K peak users)

**Load calculation:**
- 20K users × 10 LLM calls/hour = 200K calls/hour = ~56 calls/sec average
- Burst factor 3x = ~170 LLM calls/sec at peak
- Each call = 2 Guardian checks (input + output) = **~340 req/sec peak to Guardian**
- 2 guard types per check, chạy song song = 340 req/sec per model

**GPU sizing per model:**
- vLLM throughput: ~150 req/sec per A100 (DistilBERT ~66M params, thực tế nhanh hơn nhiều)
- DistilBERT là classifier nhỏ, thực tế có thể đạt 500–1000 req/sec/GPU
- **Với cache 40% hit rate:** 340 × 0.6 = ~204 req/sec effective
- **GPUs needed (conservative):** 2–3 A100 per model = **4–6 GPU total**

### 5.3 Kubernetes HPA

Scale dựa trên queue depth (không phải CPU):

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: vllm_num_requests_in_queue
    target:
      type: AverageValue
      averageValue: "50"   # scale khi queue > 50 requests
minReplicas: 2
maxReplicas: 10
```

---

## 6. Reliability & Error Handling

### 6.1 Circuit Breaker

```
Guardian timeout strategy:
- Timeout per guard check: 500ms
- Nếu timeout: fail-open (SAFE) + log warning
- Exception: Info Classification ≥ Secret → fail-closed (block)

GenAI Gateway timeout strategy:
- Guardian call timeout: 800ms total
- Nếu Guardian unreachable (503/timeout):
  - Non-sensitive requests (heuristic): fail-open + flag
  - Always-on compliance mode: fail-closed
```

**Heuristic cho fail-open/fail-closed:** GenAI Gateway có thể detect service type từ API key scopes hoặc header để quyết định policy.

### 6.2 Degraded Mode

Khi Guardian không available:
1. **Shadow mode:** Request vẫn đi qua, Guardian check async sau (không block user)
2. **Log & alert:** Toàn bộ requests trong window "unguarded" được log
3. **Rate limit:** Giảm throughput xuống 10% bình thường để reduce risk khi không có guardrail

---

## 7. Observability

### 7.1 Metrics (OpenTelemetry → Prometheus)

| Metric | Type | Alert threshold |
|---|---|---|
| `guardian_check_latency_ms` (p50/p95/p99) | Histogram | p99 > 500ms |
| `guardian_unsafe_rate` (per guard type) | Counter | Spike > 2x baseline |
| `guardian_cache_hit_rate` | Gauge | < 20% (cache issue) |
| `guardian_circuit_open` | Gauge | > 0 (Guardian down) |
| `vllm_queue_depth` | Gauge | > 100 |
| `guardian_error_rate` | Counter | > 1% |

### 7.2 Tracing

Mỗi request có `trace_id` xuyên suốt từ GenAI Gateway → Guardian → vLLM:
```
GenAI Gateway [trace_id: abc123]
  └── Guardian /check/input [span: input_check]
        ├── Redis lookup [span: cache_lookup, hit: false]
        ├── vLLM PII check [span: pii_inference, latency: 87ms]
        └── vLLM InfoClass check [span: infocls_inference, latency: 54ms]
```

---

## 8. Deployment Topology

```
┌─────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                  │
│                                                      │
│  ┌───────────────┐    ┌──────────────────────────┐  │
│  │ GenAI Gateway │    │ Guardian (FastAPI)        │  │
│  │  Deployment   │───→│  Deployment (2+ pods)    │  │
│  └───────────────┘    └──────────────┬───────────┘  │
│                                      │               │
│                          ┌───────────┼─────────┐    │
│                          ▼           ▼         │    │
│                   ┌──────────┐ ┌──────────┐   │    │
│                   │ vLLM PII │ │ vLLM     │   │    │
│                   │ (GPU pod)│ │ InfoClass│   │    │
│                   └──────────┘ │ (GPU pod)│   │    │
│                                └──────────┘   │    │
│                                               │    │
│                   ┌───────────────────────────┘    │
│                   ▼                                 │
│            ┌────────────┐                          │
│            │   Redis    │  (cache + session)        │
│            │  Cluster   │                           │
│            └────────────┘                          │
│                                                      │
│  ┌────────────────────────────────────────────┐     │
│  │  Observability: Prometheus + Grafana        │     │
│  └────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

---

## 9. Latency Budget

Toàn bộ flow với parallel input check:

```
Thời điểm T=0: User gửi request tới GenAI Gateway

T=0ms    : GenAI Gateway nhận request
T=1ms    : Fork 2 tasks song song:
           ├── Task A: Gửi request tới External LLM (Claude/GPT)
           └── Task B: Gửi tới Guardian /check/input

T=100ms  : Guardian /check/input hoàn thành (PII + InfoClass)
           └── Nếu UNSAFE → cancel Task A, return block message (T=101ms total)

T=500ms–5s: External LLM trả về response (Task A)

T=LLM+50ms : Guardian /check/output (buffer 256 tokens, validate)

T=LLM+150ms: Response tới user (streaming: first token ở T=LLM+100ms)

────────────────────────────────────────────────────
User-perceived latency overhead từ Guardian:
  Input check:  ~0ms (ẩn sau LLM call)
  Output check: ~50–150ms (buffer trước khi stream)
  Total: +50–150ms / một LLM call = ~3–5% overhead
```

---

## Câu hỏi còn mở

1. **Policy quyết định:** Khi Info Classification trả về `secret` cho input prompt — block hay chỉ log? (cần CISO sign-off)
2. **PII trong output:** Nếu LLM output chứa PII — block toàn bộ hay masked và forward? Ai quyết định?
3. **Rate limiting:** Guardian có áp rate limit per-user/per-service không? Hay để GenAI Gateway handle?
4. **Audit log retention:** Log các request UNSAFE giữ bao lâu? (GDPR / privacy compliance)
5. **Model update rollout:** Khi update model weights — rolling deploy hay blue/green? Behavior thay đổi có thể gây ra policy gap?
