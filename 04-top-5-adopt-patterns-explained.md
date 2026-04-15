# 5 Ý tưởng ADOPT quan trọng nhất — Diễn giải đơn giản

**Mục đích:** Giải thích cho team cách hiểu, cách dùng, và tại sao cần 5 pattern này khi xây Guardian.
**Đối tượng:** Engineer chưa quen với guardrail framework, cần ví dụ code trực quan.

---

## Pattern 1 — Scanner Protocol Interface

**Nguồn:** [llm-guard](https://github.com/protectai/llm-guard) — file `llm_guard/input_scanners/base.py`

### Vấn đề nếu không làm

Nếu viết PII check và InfoClass check trực tiếp trong handler:

```python
# ❌ Cách xấu — tightly coupled
@app.post("/check/input")
async def check_input(req: Request):
    text = req.prompt

    # PII check inline
    pii_model = load_pii_model()
    entities = pii_model.predict(text)
    if entities:
        text = mask_entities(text, entities)

    # InfoClass check inline
    cls_model = load_cls_model()
    level = cls_model.predict(text)
    if level in ["secret", "top_secret"]:
        return {"verdict": "UNSAFE"}

    return {"verdict": "SAFE"}
```

Vấn đề:
- Thêm scanner mới (ví dụ: jailbreak detection) → phải sửa handler
- Không test được từng scanner độc lập
- Không swap được model khi cần
- Không config threshold từ bên ngoài

### Giải pháp: Protocol-based design

Định nghĩa một **interface chung** mà mọi scanner phải implement:

```python
# app/scanners/base.py
from typing import Protocol
from pydantic import BaseModel

class ScanResult(BaseModel):
    sanitized_text: str       # text sau khi masking (nếu có)
    is_safe: bool              # True = pass
    risk_score: float          # 0.0 – 1.0
    metadata: dict             # thông tin bổ sung (entities, level, confidence)

class GuardianScanner(Protocol):
    """Mọi scanner phải tuân theo interface này."""

    name: str

    async def scan(self, text: str, context: dict) -> ScanResult:
        ...
```

Mỗi scanner là một class riêng implement Protocol:

```python
# app/scanners/pii_scanner.py
class PIIScanner:
    name = "pii"

    def __init__(self, vllm_client, threshold: float):
        self.client = vllm_client
        self.threshold = threshold

    async def scan(self, text: str, context: dict) -> ScanResult:
        response = await self.client.predict("pii-model", text)
        entities = response.entities
        is_safe = len(entities) == 0
        sanitized = mask_entities(text, entities) if not is_safe else text
        return ScanResult(
            sanitized_text=sanitized,
            is_safe=is_safe,
            risk_score=response.max_confidence,
            metadata={"entities": entities},
        )

# app/scanners/info_class_scanner.py
class InfoClassificationScanner:
    name = "info_classification"

    async def scan(self, text: str, context: dict) -> ScanResult:
        response = await self.client.predict("infocls-model", text)
        level = response.level
        is_safe = level in ["public", "internal_use_only", "restricted"]
        return ScanResult(
            sanitized_text=text,
            is_safe=is_safe,
            risk_score={"public": 0.0, "internal_use_only": 0.2,
                        "restricted": 0.5, "secret": 0.8, "top_secret": 1.0}[level],
            metadata={"level": level, "confidence": response.confidence},
        )
```

Handler trở nên sạch:

```python
# app/main.py
scanners: list[GuardianScanner] = [
    PIIScanner(vllm_client, threshold=0.85),
    InfoClassificationScanner(vllm_client, threshold=0.80),
]

@app.post("/v1/check/input")
async def check_input(req: CheckRequest):
    results = await run_scanners(scanners, req.content, req.context)
    return aggregate(results)
```

### Lợi ích

1. **Thêm scanner mới** chỉ cần tạo file mới + thêm vào list, không sửa handler
2. **Test độc lập** — mock `vllm_client`, test logic của từng scanner
3. **Swap model** — thay URL vLLM trong config, không sửa code
4. **Config-driven** — threshold, enabled/disabled từ YAML

---

## Pattern 2 — FastAPI Server Scaffold

**Nguồn:** [llm-guard](https://github.com/protectai/llm-guard) — thư mục `llm_guard_api/app/`

### Vấn đề nếu không làm

Tự xây từ zero:
- Mất 1–2 tuần viết boilerplate (auth, rate limit, metrics, config loader, health check)
- Dễ bỏ sót detail production (probe tuning, lazy load, graceful shutdown)
- Không có reference để debug khi vấn đề xảy ra

### Giải pháp: Copy structure đã proven

Từ llm-guard, cấu trúc này đã battle-tested ở production:

```
guardian/
├── app/
│   ├── main.py                 # FastAPI app, lifespan events
│   ├── scanners/
│   │   ├── base.py             # Protocol definition
│   │   ├── pii_scanner.py
│   │   └── info_class_scanner.py
│   ├── clients/
│   │   └── vllm_client.py      # HTTP pool, retry, circuit breaker
│   ├── config/
│   │   ├── loader.py           # YAML → Pydantic
│   │   └── schemas.py
│   ├── middleware/
│   │   ├── auth.py             # Bearer token
│   │   ├── rate_limit.py       # slowapi
│   │   └── metrics.py          # Prometheus exporter
│   ├── schemas.py              # Request/Response models
│   └── orchestrator.py         # Run scanners in parallel
├── config/
│   └── guardian.yaml           # Scanner config
├── tests/
├── Dockerfile
└── k8s/
    ├── deployment.yaml         # với lazy-load + probe tuning
    └── hpa.yaml
```

**Các pattern cụ thể cần copy:**

#### 2.1 Lazy model loading

```python
# app/main.py
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: không load model ngay, chỉ khởi tạo client
    app.state.vllm_client = VLLMClient(config.vllm_url)
    app.state.scanners = build_scanners(config, app.state.vllm_client)
    print("Guardian ready — models sẽ lazy-load ở request đầu tiên")
    yield
    # Shutdown
    await app.state.vllm_client.close()

app = FastAPI(lifespan=lifespan)
```

#### 2.2 K8s probe tuning (từ llm-guard docs)

```yaml
# k8s/deployment.yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 60    # đợi vLLM connection ready
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 120   # model warmup có thể mất vài phút
  periodSeconds: 10
```

#### 2.3 Fail-fast với timeout

```python
# app/orchestrator.py
async def run_scanners(scanners, text, context, fail_fast=True, timeout_ms=500):
    try:
        results = await asyncio.wait_for(
            asyncio.gather(*[s.scan(text, context) for s in scanners]),
            timeout=timeout_ms / 1000,
        )
    except asyncio.TimeoutError:
        return [ScanResult(sanitized_text=text, is_safe=False,
                           risk_score=1.0, metadata={"error": "timeout"})]

    if fail_fast and any(not r.is_safe for r in results):
        return results  # dừng, không chờ scanner chậm hơn

    return results
```

### Lợi ích

- **Tiết kiệm 1–2 tuần** so với viết từ đầu
- **Tránh bug production** — probe tuning, lazy load đã được llm-guard học từ real incidents
- **Có reference** khi bug — so sánh với llm-guard behavior

---

## Pattern 3 — `asyncio.gather` Concurrent Execution

**Nguồn:** [guardrails-ai](https://github.com/guardrails-ai/guardrails) — `AsyncGuard` class

### Vấn đề nếu không làm

Chạy tuần tự các scanner → **latency cộng dồn**:

```python
# ❌ Sequential — latency = PII + InfoClass
pii_result = await pii_scanner.scan(text, ctx)       # 80ms
info_result = await info_scanner.scan(text, ctx)     # 60ms
# Total: 80 + 60 = 140ms
```

Với constraint <5% overhead trên 2s LLM call = 100ms budget → 140ms đã vỡ budget.

**Tệ hơn:** Khi thêm scanner thứ 3 (jailbreak detection 70ms) → 210ms. Linear growth.

### Giải pháp: Chạy song song với `asyncio.gather`

```python
# ✅ Parallel — latency = max(PII, InfoClass)
pii_result, info_result = await asyncio.gather(
    pii_scanner.scan(text, ctx),        # 80ms
    info_scanner.scan(text, ctx),       # 60ms
)
# Total: max(80, 60) = 80ms
```

Thêm scanner thứ 3 không làm tăng latency nếu model thứ 3 ≤ scanner chậm nhất:

```python
pii, info, jailbreak = await asyncio.gather(
    pii_scanner.scan(text, ctx),        # 80ms
    info_scanner.scan(text, ctx),       # 60ms
    jailbreak_scanner.scan(text, ctx),  # 70ms
)
# Total: max(80, 60, 70) = 80ms  ← vẫn 80ms!
```

### Tại sao `asyncio.gather` hoạt động

Vì các scanner gọi vLLM qua HTTP — **I/O-bound**, không blocking CPU:

```
Time →
Scanner PII     : [==== HTTP call 80ms ====]
Scanner InfoCls : [=== HTTP call 60ms ===]
                 ↑ cả 2 cùng start
                                          ↑ PII xong
                                    ↑ InfoCls xong
                                          ↑ gather() unblock
```

Event loop switch giữa các task khi một task đang chờ I/O → không có task nào chờ task khác.

### Cảnh báo từ NeMo experience (Pattern 5)

Nếu một scanner **mutate** text (ví dụ masking) và scanner khác đọc text đó:

```python
# ❌ Race condition
pii_result, info_result = await asyncio.gather(
    pii_scanner.scan(text, ctx),          # mask text → "My name is [PERSON]"
    info_scanner.scan(text, ctx),         # classify original hay masked?
)
```

Giải quyết: **Split detection (parallel) và mutation (serial)** — xem Pattern 5.

### Lợi ích

- **Latency = max, không phải sum** — thêm scanner không phá budget
- **Scale horizontal tốt** — vLLM xử lý song song, CPU của Guardian idle
- **Đơn giản** — 1 dòng code thay đổi, không cần framework phức tạp

---

## Pattern 4 — Per-Scanner `on_fail` Action (YAML-driven)

**Nguồn:** [guardrails-ai](https://www.guardrailsai.com/docs/concepts/validator_on_fail_actions)

### Vấn đề nếu không làm

Hard-code policy trong code:

```python
# ❌ Policy trong code
if pii_result.is_unsafe:
    return block_response()   # block luôn
if info_result.level == "secret":
    log_and_alert()
    return block_response()
if info_result.level == "restricted":
    log_only()
    continue  # vẫn pass
```

Vấn đề:
- CISO thay đổi policy → deploy lại code
- Service A cần strict, service B cần permissive → phải duplicate code
- Không audit được policy history (cần Git log code, lẫn với code changes khác)

### Giải pháp: Declarative policy trong YAML

```yaml
# config/guardian.yaml
scanners:
  pii:
    enabled: true
    model: "vllm://pii-distilbert"
    threshold: 0.85
    on_fail: mask              # mask | reject | log_only | escalate

  info_classification:
    enabled: true
    model: "vllm://infocls-lora"
    threshold: 0.80
    on_fail_by_level:
      public: allow
      internal_use_only: allow
      restricted: log_only      # pass nhưng log
      secret: reject             # block request
      top_secret: reject_and_alert   # block + gửi alert cho CISO

# Policy per consumer (API key scope)
policies:
  default:
    scanners: ["pii", "info_classification"]
    fail_mode: fail_open         # Guardian down → pass

  high_compliance:
    scanners: ["pii", "info_classification"]
    fail_mode: fail_closed       # Guardian down → block
    override:
      info_classification.on_fail_by_level.restricted: reject
```

Code handler trở nên generic:

```python
# app/orchestrator.py
def apply_action(result: ScanResult, action: str) -> FinalVerdict:
    match action:
        case "allow":
            return FinalVerdict(pass_through=True)
        case "mask":
            return FinalVerdict(pass_through=True, text=result.sanitized_text)
        case "log_only":
            logger.warning("Scanner failed", extra=result.metadata)
            return FinalVerdict(pass_through=True)
        case "reject":
            return FinalVerdict(pass_through=False, reason="policy_violation")
        case "reject_and_alert":
            send_alert(result)
            return FinalVerdict(pass_through=False, reason="policy_violation")
        case "escalate":
            queue_for_human_review(result)
            return FinalVerdict(pass_through=False, reason="requires_review")
```

### Lợi ích

1. **Policy change = config change** — CISO sửa YAML, rollout qua K8s configmap, không deploy code
2. **Per-consumer policy** — service A dùng `default`, service B dùng `high_compliance`
3. **Audit trail** — Git history của `guardian.yaml` = lịch sử policy thay đổi
4. **Testing dễ** — unit test từng action độc lập, không cần spin up full service

---

## Pattern 5 — Detection Parallel + Mutation Serial

**Nguồn:** NVIDIA NeMo IORails warning — race condition khi rail mutate data trong parallel mode

### Vấn đề khi kết hợp Pattern 3 (parallel) với masking

```python
# ❌ Race condition
async def check_input(text: str):
    pii_result, info_result = await asyncio.gather(
        pii_scanner.scan(text, ctx),      # MUTATE: mask text
        info_scanner.scan(text, ctx),     # READ: classify
    )
    # Câu hỏi: info_scanner đã classify text gốc hay text đã mask?
    # Trả lời: text GỐC — vì cả 2 nhận `text` làm parameter, không chia sẻ state
    # Nhưng nếu muốn info_scanner classify text đã mask thì sao?
```

Câu hỏi thực sự: **Muốn classify text gốc hay text đã mask?**

**Use case 1:** Classify text gốc (để biết "prompt này chứa thông tin Secret không?")
→ Parallel OK, vì InfoClass không phụ thuộc PII masking.

**Use case 2:** Classify text đã mask (để biết "sau khi mask thì còn Secret không?")
→ Cần chạy tuần tự: PII mask trước, InfoClass sau.

Guardian cần **cả 2 verdict** — prompt gốc có vi phạm? Và nếu mask thì có còn vi phạm?

### Giải pháp: Split thành 2 stages

```python
async def check_input(text: str):
    # Stage 1: Detection (parallel, read-only, không mutate)
    pii_detection, info_classification = await asyncio.gather(
        pii_scanner.detect(text),              # trả entities, KHÔNG mask
        info_scanner.classify(text),           # trả level
    )

    # Stage 2: Masking (serial, chỉ khi cần)
    masked_text = text
    if pii_detection.has_pii:
        masked_text = apply_mask(text, pii_detection.entities)

    # Stage 3: Re-classify sau mask (serial, chỉ khi cần)
    post_mask_info = None
    if pii_detection.has_pii and info_classification.level in ["secret", "top_secret"]:
        # Mask có giảm được level không?
        post_mask_info = await info_scanner.classify(masked_text)

    return CheckResult(
        original_level=info_classification.level,
        pii_entities=pii_detection.entities,
        masked_text=masked_text,
        post_mask_level=post_mask_info.level if post_mask_info else None,
    )
```

### Flow minh họa

```
text = "My SSN is 123-45-6789 and I want to discuss project Alpha."

Stage 1 (parallel, ~80ms):
  ├── PII detect    → entities: [SSN: 123-45-6789]
  └── InfoClass     → level: secret (vì SSN)

Stage 2 (serial, ~5ms):
  masked_text = "My SSN is [REDACTED_SSN] and I want to discuss project Alpha."

Stage 3 (serial, ~60ms, optional):
  InfoClass (masked) → level: internal_use_only (SSN đã mask)

Final verdict:
  - Original level: secret
  - Masked level: internal_use_only
  - Action: tùy policy
    • Nếu policy allow mask-and-forward → forward masked_text
    • Nếu policy strict → block (original là secret)
```

### Tại sao pattern này quan trọng

1. **Tránh race condition** — mỗi stage có input/output rõ ràng
2. **Tối ưu cost** — Stage 3 chỉ chạy khi cần (có PII + level cao)
3. **Flexible policy** — GenAI Gateway có thể quyết định dùng `original_level` hay `post_mask_level` tùy service
4. **Debug dễ** — log từng stage độc lập, biết chính xác cái nào chậm

---

## Tổng kết — Cách 5 patterns tương tác

```
┌──────────────────────────────────────────────────────────┐
│  Request: POST /v1/check/input                            │
│                                                           │
│  [Pattern 2] FastAPI scaffold                             │
│    ├── Middleware (auth, rate limit, metrics)             │
│    ├── Load [Pattern 4] policy from YAML                  │
│    │                                                       │
│    └── Orchestrator                                        │
│          │                                                 │
│          ├── [Pattern 1] Scanner Protocol                  │
│          │    ├── PIIScanner.detect()                      │
│          │    └── InfoClassificationScanner.classify()     │
│          │                                                 │
│          ├── [Pattern 3] asyncio.gather                    │
│          │    └── Stage 1: Parallel detection              │
│          │                                                 │
│          ├── [Pattern 5] Split stages                      │
│          │    ├── Stage 2: Serial masking                  │
│          │    └── Stage 3: Serial re-classify (optional)   │
│          │                                                 │
│          └── [Pattern 4] Apply on_fail action              │
│               └── allow | mask | reject | escalate         │
│                                                             │
│  Response: verdict + metadata                              │
└──────────────────────────────────────────────────────────┘
```

### Thứ tự học cho team mới

1. **Pattern 1** trước — hiểu interface design, làm nền cho tất cả
2. **Pattern 2** — có skeleton để chạy thử được
3. **Pattern 3** — thêm 2 scanners, chạy song song, đo latency
4. **Pattern 5** — khi bắt đầu làm masking, split stages để tránh race
5. **Pattern 4** cuối — khi có 2–3 consumers với policy khác nhau

---

## Câu hỏi thực hành cho team

Sau khi đọc xong, mỗi người tự trả lời:

1. Viết `GuardianScanner` Protocol definition trong Python, có `name` và method `scan()`
2. Giải thích tại sao `asyncio.gather` không chạy trên multi-thread nhưng vẫn parallel được
3. Cho ví dụ 2 use case cần fail_closed và 2 use case cần fail_open
4. Khi nào policy thay đổi cần deploy lại code, khi nào chỉ cần update YAML?
5. Một request có 3 scanners: A (100ms), B (50ms), C (80ms). Chạy tuần tự mất bao nhiêu? Chạy song song mất bao nhiêu?

**Đáp án câu 5:** Tuần tự 230ms, song song 100ms (= max).
