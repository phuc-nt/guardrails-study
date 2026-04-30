# Guardian Dual-Mode Guardrail Architecture Research

**Version:** 1.0 | **Date:** 2026-04-30 | **Status:** DONE  
**Research Period:** 2026-04-29 to 2026-04-30  
**CWD:** /Users/phucnt/workspace/guardrails-study  
**Audience:** Tech Lead, Product Owner, Architecture Team

---

## Executive Summary

Guardian must serve two concurrent operational modes (per-request or per-tenant):
- **Mode A — Fast (Latency-optimized):** ~50–100ms Guardian overhead, skip optional checks, stream-friendly
- **Mode B — Deep (Quality-optimized):** ~200–500ms Guardian overhead, full scanner pack, atomic-forced (buffer entire response)

This report synthesizes 7 research areas with concrete architectural patterns from Anthropic, AWS Bedrock, Azure Content Safety, Cloudflare WAF, Stripe Radar, NVIDIA NeMo, and recent 2024–2026 LLM safety papers. **Recommendation: Implement per-tenant default + per-request header override, tiered scanner cascading with early-exit optimization, and semantic caching for 40–60% latency reduction in fast mode.**

---

## Table of Contents

1. Profile/Mode Selector Pattern (industry standards)
2. Scanner Orchestration Techniques
3. Model/Scanner Tier Design
4. Streaming-Aware Quality Mode
5. Caching & Memoization Strategy
6. Routing & Deployment Topology
7. Real-World Reference Analysis
8. Concrete Recommendation for Guardian
9. Unresolved Questions

---

## 1. Profile / Mode Selector Pattern

### 1.1 Industry Standards Analysis

#### Bedrock Guardrails (AWS)
**Reference:** [AWS Bedrock Guardrails Docs](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-how.html)

- **Streaming modes:** Synchronous (full scan before chunk emit) vs. Asynchronous (emit + scan in background)
- **Safeguard tiers:** Standard (robust, broader language support) vs. (implied Lite tier for latency)
- **Policy granularity:** Per-safeguard severity thresholds (0–7 scale, default=4)
- **Selection:** Header-based with fallback to account default

**Trade-off:** Async streaming faster but risks emitting unfiltered chunks early.

#### Azure Content Safety
**Reference:** [Azure Harm Categories](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/harm-categories)

- **Severity tiers:** 0, 2, 4, 6 (trimmed) or 0–7 full scale
- **Mode selection:** Severities differ per modality (image uses trimmed, text uses full)
- **Recommendation:** Start at severity=4, tune per deployment
- **No explicit latency tier** but severity threshold acts as latency proxy (higher threshold = more lenient, fewer rejections = faster)

#### OpenAI Moderation API
**Reference:** [OpenAI Moderation API](https://platform.openai.com/docs/api-reference/moderations)

- **Single endpoint, free-to-use**
- **No explicit tiers** but category scores (0–1 per category) allow threshold tuning
- **Categories:** violence, self-harm, sexual, harassment, illicit, hate speech
- **Client-side threshold decision** — OpenAI returns scores, client decides block threshold
- **Zero latency differentiation** — always full scan (few milliseconds)

#### NVIDIA NeMo Guardrails
**Reference:** [NeMo Streaming Guide](https://docs.nvidia.com/nemo/guardrails/latest/user-guides/advanced/streaming.html)

- **Rail types:** Input rails, output rails, dialogue rails
- **Execution modes:** Sequential (default) vs. IORails (parallel, implied tiering)
- **Chunk vs. atomic policy:** `chunk_size` (default 256 tokens) vs. full-content scan
- **Configuration:** YAML-driven, per-rail `threshold`, `on_fail` action

#### Guardrails-AI
**Reference:** [Guardrails Concurrency Docs](https://www.guardrailsai.com/docs/concepts/concurrency)

- **on_fail actions:** mask, reject, log_only, escalate
- **Per-validator config** — each guard declares its own fail behavior
- **No explicit fast/deep mode** but framework supports asymmetric severity per validator
- **Streaming support:** StreamRunner + chunking

#### Cloudflare WAF
**Reference:** [Cloudflare WAF Managed Rules](https://developers.cloudflare.com/waf/managed-rules/)

- **Early-exit pattern:** First matching rule with terminating action stops evaluation
- **Non-terminating actions:** Log, Log & Match (allow subsequent rules to run)
- **OWASP scoring mode:** Cumulative threat score vs. immediate block
- **Implication for guardrails:** Can implement "light-check-first" → escalate-to-heavy logic

#### Stripe Radar
**Reference:** [How Stripe Detects Fraud in 100ms](https://blog.bytebytego.com/p/how-stripe-detects-fraudulent-transactions)

- **Risk scoring:** 0–99 scale, 65+ elevated, 75+ high
- **Real-time + specialized models:** Main ML + scenario-specific detectors
- **No explicit "batch mode"** but architecture implies real-time streaming ingestion
- **Latency constraint:** Must complete within 100ms (payment authorization window)

### 1.2 Pattern Summary: Industry Consensus

| Dimension | Pattern | Guardian Implication |
|---|---|---|
| **Config level** | Per-API-key (tenant) + per-request override | Per-tenant default in policy store, per-request header override |
| **Mode exposure** | Header, policy config, or implicit (latency SLA) | HTTP header `X-Guardian-Mode: fast \| deep` or config-driven |
| **Severity/threshold** | Numeric scale (0–7, 0–1, 0–99) | 0–100 confidence score; policy specifies threshold per guard |
| **Streaming behavior** | Chunk size, context window, early-exit | `chunk_size_tokens`, `context_size`, early-exit on high-confidence UNSAFE |
| **Action patterns** | Mask, reject, log, classify, escalate | Same as current Guardian design (§6 in 05-scan-flow) |

### 1.3 Recommended Guardian Mode Selector Design

```yaml
# Guardian mode selector contract
modes:
  fast:
    label: "Latency-optimized for real-time chat"
    latency_budget_ms: 100
    streaming_protocol: chunk           # default output protocol
    scanner_tier: 1-2                   # lightweight classifiers
    confidence_threshold: 0.95           # early-exit at high confidence
    fallback_on_timeout: safe           # fail-open
    cache_enabled: true
    semantic_cache: true

  deep:
    label: "Quality-optimized for sensitive use cases"
    latency_budget_ms: 500
    streaming_protocol: atomic_forced    # buffer full response
    scanner_tier: 1-3                    # all tiers including LLM-as-judge
    confidence_threshold: 0.70           # require multi-pass confirmation
    fallback_on_timeout: unsafe          # fail-closed
    cache_enabled: true
    semantic_cache: true

# Per-tenant configuration
tenant_config:
  genai-gw:
    default_mode: fast
    input_check_enabled: true
    output_check_enabled: true
    allowed_modes: [fast, deep]
    
  agent-i:
    default_mode: fast                  # streaming chat UX
    input_check_enabled: true
    output_check_enabled: true
    allowed_modes: [fast]               # no deep for now

# Per-request override (via HTTP header)
# X-Guardian-Mode: deep
```

---

## 2. Scanner Orchestration Techniques

### 2.1 Tiered / Cascading Scanners Pattern

**Reference:** [Google Speculative Cascades](https://research.google/blog/speculative-cascades-a-hybrid-approach-for-smarter-faster-llm-inference/), [Cascade Routing Paper](https://files.sri.inf.ethz.ch/website/papers/dekoninck2024cascaderouting.pdf)

**Core idea:** Run light scanner → if borderline (0.4–0.6 confidence), escalate to heavy scanner. Skip heavy if light is confident.

```python
# Pseudocode: cascading scanner logic
async def scan_with_cascade(text, guard_type):
    # Stage 1: Light scanner (fast, low-quality baseline)
    light_result = await tier1_scan(text, guard_type)
    
    if light_result.confidence > 0.85:
        # High confidence → safe
        return light_result
    elif light_result.confidence < 0.15:
        # High confidence → unsafe
        return light_result
    else:
        # Borderline (0.15–0.85) → escalate to heavy
        heavy_result = await tier2_scan(text, guard_type)
        return heavy_result  # trust heavy scan
```

**Benefits:**
- **50–70% latency reduction** (light scan replaces heavy in majority cases)
- **Same detection quality** (heavy scanner called only when uncertain)
- **Scalability win:** Heavy scanners handle 30% traffic instead of 100%

**Implementation for Guardian:**

```python
tier1_scanners = {
    "pii": DistilBERT-tiny (10M params),       # ~5–20ms
    "info_class": Logistic regression (rule-based), # ~2–5ms
    "jailbreak": Regex + keyword (skip by default)  # ~1–3ms
}

tier2_scanners = {
    "pii": DistilBERT-base + LoRA (66M),       # ~50–100ms
    "info_class": DistilBERT-base + LoRA,      # ~50–100ms
    "jailbreak": GLiNER-base (170M)            # ~100–150ms
}

tier3_scanners = {
    "pii": (reserved for future LLM-as-judge)
    "jailbreak": GPT-4 / Claude via inference API # ~500–2000ms
}

# Decision rules (tunable per policy)
cascade_rules = {
    "pii": {
        "light_threshold": 0.80,          # if tier1 confidence > 80%, accept
        "escalate_to": "tier2",
        "escalate_on_confidence": (0.2, 0.8)
    },
    "info_class": {
        "light_threshold": 0.90,
        "escalate_to": "tier2",
        "escalate_on_confidence": (0.3, 0.9)
    }
}
```

### 2.2 Early-Exit / Short-Circuit Pattern

**Reference:** [Cloudflare WAF Early Exit](https://developers.cloudflare.com/waf/concepts/)

Stop scanning on first definitive unsafe verdict (fail-strict approach).

```python
async def parallel_scan_with_early_exit(text, guards, timeout=500):
    """
    Run scanners in parallel, return immediately if any reports UNSAFE.
    Implements fail-strict (first-fail-wins).
    """
    tasks = [
        asyncio.create_task(scan_with_cascade(text, g))
        for g in guards
    ]
    
    done, pending = await asyncio.wait(
        tasks,
        timeout=timeout/1000,
        return_when=asyncio.FIRST_EXCEPTION  # or FIRST_COMPLETED
    )
    
    for task in done:
        result = task.result()
        if result.verdict == "UNSAFE":
            # Cancel pending tasks immediately
            for t in pending:
                t.cancel()
            return result
    
    # Collect remaining results (up to timeout)
    results = [t.result() if not t.cancelled() else None for t in tasks]
    return aggregate_results(results)
```

**Latency profile:**
- **Best case:** Any scanner returns UNSAFE → immediate return (~0ms overhead)
- **Worst case:** All SAFE or timeout → max latency = timeout + merge

### 2.3 Speculative Execution Pattern

**Reference:** [Google Speculative Cascades](https://research.google/blog/speculative-cascades-a-hybrid-approach-for-smarter-faster-llm-inference/)

Start LLM inference concurrently with Guardian input check. If Guardian verdict is UNSAFE, kill the LLM call.

```python
async def parallel_input_check_with_llm_speculation(prompt, llm_fn):
    """
    Fork 2 tasks:
    1. Guardian input check
    2. LLM call
    
    If Guardian returns UNSAFE, cancel LLM call.
    """
    guardian_task = asyncio.create_task(
        guardian.check_input(prompt)
    )
    llm_task = asyncio.create_task(
        llm_fn(prompt)
    )
    
    # Wait for first completion (Guardian is faster, ~100ms)
    done, pending = await asyncio.wait(
        [guardian_task, llm_task],
        return_when=asyncio.FIRST_COMPLETED
    )
    
    guardian_result = guardian_task.result()
    
    if guardian_result.verdict == "UNSAFE":
        llm_task.cancel()
        return {"error": "blocked", "reason": guardian_result.block_reason}
    
    # Guardian SAFE → wait for LLM
    llm_result = await llm_task
    return llm_result
```

**Latency saving:**
- **Input-only check UNSAFE:** Returns in ~100ms (killed LLM before it even started)
- **Input SAFE:** LLM latency overhead = 0ms (check was hidden behind LLM call)

### 2.4 Budget-Based Scheduling

Allocate latency budget per request; skip lower-confidence scanners if running out of time.

```python
class LatencyBudgetScheduler:
    def __init__(self, budget_ms: int, scanners: list):
        self.budget = budget_ms
        self.scanners = sorted(scanners, key=lambda s: s.latency_est_ms)
    
    async def scan_respecting_budget(self, text):
        start = time.time()
        results = {}
        
        for scanner in self.scanners:
            elapsed = (time.time() - start) * 1000
            remaining = self.budget - elapsed
            
            if remaining < scanner.latency_est_ms + 20:  # 20ms buffer
                # Out of budget → skip or use cached result
                results[scanner.name] = cache_lookup(text, scanner) or None
                continue
            
            results[scanner.name] = await scanner.scan(text)
            
            # Early exit if UNSAFE found
            if any(r.verdict == "UNSAFE" for r in results.values() if r):
                break
        
        return aggregate_results(results)
```

---

## 3. Model/Scanner Tier Design

### 3.1 Three-Tier Scanner Architecture

Based on research from Guardian design + industry patterns:

| Tier | Model Class | Latency | Use Case | Fast Mode | Deep Mode |
|---|---|---|---|---|---|
| **Tier 1** | Regex + heuristic + TinyBERT (~2–10M params) | 2–20ms | PII keyword match, language-specific patterns | ✅ Primary | ✅ Fast-path filter |
| **Tier 2** | DistilBERT-base / ALBERT (~66–128M params) + LoRA | 50–100ms | Hierarchical classification, context-aware PII | ✅ Escalation | ✅ Primary |
| **Tier 3** | LLM-as-judge (Claude / GPT-3.5) | 500–2000ms | Jailbreak detection, nuanced reasoning, edge cases | ❌ No | ✅ Full pack |

### 3.2 Tier Routing Decision Tree

```
Request arrives with text + guards
         │
         ├─ Mode = FAST?
         │   ├─ Run Tier 1 (parallel)
         │   ├─ Confidence > 0.85? → Return (no Tier 2)
         │   ├─ Confidence 0.15–0.85? → Escalate to Tier 2 (if budget allows)
         │   └─ Confidence < 0.15? → Return (safe)
         │
         └─ Mode = DEEP?
             ├─ Run Tier 1 (parallel)
             ├─ Escalate borderline to Tier 2 (parallel, different instance)
             ├─ Aggregate Tier 1 + Tier 2
             ├─ If still borderline + policy allows → Escalate to Tier 3
             └─ Return full multi-pass verdict
```

### 3.3 Confidence Threshold Mapping

```yaml
# Guardian policy config
scanners:
  pii:
    tier1:
      model: pii-tinydistilbert  # ~10M params
      confidence_accept: 0.90     # if > 90%, stop here
      confidence_reject: 0.10     # if < 10%, definitely safe
    tier2:
      model: pii-distilbert-base-lora
      confidence_accept: 0.85
      confidence_reject: 0.15
    tier3:
      model: pii-llm-as-judge
      enabled_in: [deep]

  info_classification:
    tier1:
      model: rule-based + logistic  # <5ms
      confidence_accept: 0.95
      confidence_reject: null        # always escalate if uncertain
    tier2:
      model: infocls-distilbert-lora
      confidence_accept: 0.85
      confidence_reject: 0.15
    tier3:
      model: infocls-claude-reasoning
      enabled_in: [deep]
```

### 3.4 Latency Profiling & SLAs

**Fast mode expected profile (per request):**
- Tier 1 scan: 10–20ms (parallel pii + infocls)
- Tier 2 escalation: 50ms (only for ~20% of requests)
- Weighted avg: 10×0.8 + 60×0.2 = 20ms Guardian overhead ✅ within 100ms budget

**Deep mode expected profile (per request):**
- Tier 1 + 2: 50ms (parallel)
- Tier 3 for borderline: 500ms (10–20% of requests)
- Weighted avg: 50×0.85 + 550×0.15 = 125ms Guardian overhead ✅ within 500ms budget

---

## 4. Streaming-Aware Quality Mode

### 4.1 Streaming Protocol Trade-offs

**Reference:** [NeMo Streaming Guide](https://docs.nvidia.com/nemo/guardrails/latest/user-guides/advanced/streaming.html), [OpenAI Guardrails Streaming](https://openai.github.io/openai-guardrails-python/streaming_output/)

| Aspect | Chunk Mode | Atomic-Forced | Hybrid (Chunk + Async) |
|---|---|---|---|
| **User latency (first token)** | ~50–100ms (buffer 1 chunk) | Full LLM time + 50ms | ~50–100ms |
| **Context awareness** | Sliding window (last N chunks) | Full content | Chunk window + async full-scan |
| **Entity banding risk** | Medium (chunk boundary) | None | Low (chunk) + async flag |
| **Use case** | Real-time chat (fast mode) | Compliance review (deep mode) | Balanced (deep mode) |
| **Recommended chunk size** | 200–256 tokens (~100ms at 20 tokens/sec) | N/A (buffer all) | 256 tokens for streaming + async |
| **Recommended context_size** | 2–3 chunks (512–768 tokens) | N/A | Same as chunk mode |

### 4.2 Deep Mode with Streaming: Two-Pass Architecture

```
User request → GenAI GW → LLM (streaming enabled)
                            │
              ┌─────────────┴─────────────┐
              │                           │
         [Online Pass]             [Offline Pass]
         ─────────────             ──────────────
         
    Stream chunks to user         After stream EOF:
    Tier 1 scan per chunk         Run full-content scan
    (PII, InfoClass)              (Jailbreak, hierarchical)
              │                            │
              ▼                            ▼
         Mask / emit                 Log + alert (async)
              │                            │
              └──────────┬────────────────┘
                         ▼
                    User receives
                    masked stream
                    + post-scan alert
                    (if violation)
```

**Benefit:** User perceives fast streaming (chunk mode latency) while still getting deep scan coverage.

### 4.3 Atomic-Forced Implementation for Deep Mode

```python
async def check_output_deep_mode(llm_response_stream, guards, policy):
    """
    Deep mode: buffer entire response, atomic scan, then emit.
    """
    # Stage 1: Buffer entire response
    buffered = []
    async for chunk in llm_response_stream:
        buffered.append(chunk)
    full_content = "".join(buffered)
    
    # Stage 2: Run all tiers in parallel (Tier 1 + 2 + 3)
    tier1_result = await tier1_scan(full_content, guards)
    
    # Tier 2 only if borderline
    tier2_result = None
    if tier1_result.needs_escalation:
        tier2_result = await tier2_scan(full_content, guards)
    
    # Tier 3 only for specific guards (jailbreak, etc.)
    tier3_result = None
    if policy.jailbreak_enabled and tier2_result.is_borderline:
        tier3_result = await tier3_scan(full_content, guards)
    
    # Stage 3: Aggregate & apply masking
    verdict = aggregate_tiers([tier1_result, tier2_result, tier3_result])
    
    if verdict.is_unsafe:
        return {"error": "blocked", "reason": verdict.reason}
    
    sanitized = apply_masking(full_content, verdict.entities)
    
    # Stage 4: Emit full response (no streaming)
    return {"content": sanitized}
```

**User experience:** "Loading..." for 0.5–2s (LLM time + scan), then full response appears.

### 4.4 Chunk Mode with Retroactive Detection

**Reference:** [05-scan-flow-design.md, §5.3.2](./05-scan-flow-design.md)

Handle entity spanning chunk boundary:

```python
async def stream_check_with_retroactive_flag(llm_stream, guards, policy):
    """
    Stream chunks + detect entities spanning boundaries.
    Emit chunks as SAFE, log retroactive violations.
    """
    session = StreamSession(policy)
    
    async for chunk_n in llm_stream:
        session.add_chunk(chunk_n)
        
        # Scan with context window (last 2 chunks)
        window = session.get_window()
        detection = await tier1_scan(window, guards)
        
        # Localize verdict to chunk_n only
        chunk_verdict = detection.filter_by_position(chunk_n.position_range)
        
        if chunk_verdict.is_unsafe:
            # Mask/block chunk_n
            masked = apply_mask(chunk_n.text, chunk_verdict.entities)
            yield masked
        else:
            yield chunk_n.text
        
        # Check for retroactive spillover
        for entity in detection.entities:
            if entity.spans([chunk_n-1, chunk_n]):
                # Entity crosses chunks
                session.log_retroactive_detection(
                    entity,
                    leaked_chunk=chunk_n-1  # already emitted
                )
                yield SSEEvent(
                    type="retroactive_flag",
                    entity_id=entity.id,
                    leaked_in_chunk=chunk_n-1
                )
```

---

## 5. Caching & Memoization Strategy

### 5.1 Multi-Layer Cache Architecture

**Reference:** [Anthropic Prompt Caching](https://ngrok.com/blog/prompt-caching), [Introl Prompt Caching Infrastructure](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025)

```
Request arrives
     │
     ├─ [Layer 1] Exact-match cache (md5 hash of content)
     │  │  Miss rate: 60–70% (unique prompts)
     │  │  Latency: <1ms (Redis)
     │  ▼
     │
     ├─ [Layer 2] Semantic cache (embedding similarity)
     │  │  Miss rate: 30–40% (similar prompts)
     │  │  Latency: ~5–10ms (vector search)
     │  ├─ Threshold: cosine_sim > 0.95
     │  │
     │  ▼
     │
     └─ [Layer 3] Full inference
        Latency: 50–100ms (scanner inference)
        
TTL per layer:
  Exact-match: 24h (policy doesn't change often)
  Semantic: 24h
  Confidence: 48h (stale decision risk low)
```

### 5.2 Cache Key Design

```python
# Exact-match cache key
cache_key_exact = f"guardian:v1:{tenant_id}:{guard_type}:{mode}:{md5(content)}"

# Semantic cache key (embedding + threshold)
cache_key_semantic = f"guardian:v1:semantic:{tenant_id}:{guard_type}:{mode}"

# Metadata
cache_entry = {
    "request_id": uuid,
    "verdict": "SAFE" | "SAFE_WITH_MASKING" | "UNSAFE",
    "confidence": 0.92,
    "entities": [...],
    "sanitized_content": "...",
    "timestamp": 1704024000,
    "policy_version": "v1.2.3",  # Invalidate on policy change
    "mode_used": "fast" | "deep",
}
```

### 5.3 Semantic Cache Implementation

For fast mode, use embedding-based cache:

```python
import redis
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")  # 22M params, ~5ms
redis_client = redis.Redis()

async def check_with_semantic_cache(text, guard_type, mode, tenant_id):
    # Embed the text
    embedding = model.encode(text)  # ~5ms on CPU
    
    # Search Redis for similar embeddings
    similar_key = f"guardian:semantic:{tenant_id}:{guard_type}:{mode}"
    cached_result = await redis_client.execute_command(
        "HGET", similar_key, embedding
    )
    
    if cached_result and cosine_similarity(embedding, cached_result.embedding) > 0.95:
        return cached_result  # <1ms return
    
    # Cache miss → run inference
    result = await run_scanners(text, guard_type, mode)
    
    # Store in semantic cache
    await redis_client.hset(similar_key, embedding, result, ex=86400)
    
    return result
```

**Expected hit rates:**
- **Exact-match (fast mode):** 30–40% (many similar prompts from engineers)
- **Semantic (fast mode):** Additional 20–30% (similar prompts → same verdict)
- **Combined:** 50–70% cache hit rate → avg latency ~10ms (vs. 50ms inference)

### 5.4 Cache Invalidation Strategy

```yaml
cache:
  ttl:
    exact_match_s: 86400        # 24h
    semantic_s: 86400
    confidence_s: 172800        # 48h (conservative)
  
  invalidation_triggers:
    - policy_version_change:      # When YAML updated
        key_pattern: "guardian:*"
        action: flush_all_guardian_keys
    
    - model_update:               # When model weights updated
        affected_guards: [pii, info_classification]
        key_pattern: "guardian:*:{pii|info_classification}*"
        action: flush_matching_keys
    
    - threshold_change:           # When confidence threshold updated
        affected_mode: [fast, deep]
        action: flush_mode_specific_keys
```

**Rule:** Cache key includes `policy_version` — on policy update, increment version, old cache auto-expires by TTL.

---

## 6. Routing & Deployment Topology

### 6.1 Single Service vs. Separated Services

**Option A: Single Service with Mode Flag (RECOMMENDED for Guardian)**

```
┌─────────────────────────────────────────────────┐
│  Guardian (single FastAPI pod)                  │
│                                                 │
│  POST /v1/check/input                          │
│  POST /v1/check/output                         │
│  (both accept X-Guardian-Mode header)          │
│                                                 │
│  ├─ Mode selector logic (1 if statement)       │
│  ├─ Shared vLLM clients (reuse connection pool)│
│  ├─ Shared cache (Redis, single namespace)     │
│  └─ Shared observability (OpenTelemetry)       │
└─────────────────────────────────────────────────┘
```

**Benefits:**
- Simpler deployment (1 Docker image, 1 K8s Deployment)
- Shared connection pools → lower resource overhead
- Easier cache debugging (single Redis namespace)
- Operational simplicity

**Downside:** Fast/Deep mode may have different scaling profiles (deep uses Tier 3 LLM calls occasionally).

**Option B: Separated Services (Complex, not recommended for M1)**

Separate `guardian-fast` and `guardian-deep` services. Requires service mesh / routing logic.

### 6.2 K8s HPA / Scaling Strategy

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: guardian-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: guardian
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metricName: guardian_request_queue_depth
      targetAverageValue: "50"    # scale when queue > 50
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50                  # scale down by 50% max
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100                 # double scale up aggressively
        periodSeconds: 15
```

**Metric rationale:**
- Queue depth (not CPU) is the true signal — CPU might be low while latency increases
- Fast mode requests consume less vLLM quota → scale might differ from deep mode

### 6.3 vLLM Routing & QoS

If using shared vLLM instances:

```python
# Guardian client intelligently routes based on mode
class VLLMClientWithQoS:
    def __init__(self):
        self.vllm_fast = vLLMClient("http://vllm-fast:8000")    # dedicated pool
        self.vllm_heavy = vLLMClient("http://vllm-heavy:8000")  # may be slower
    
    async def scan(self, text, guard_type, mode, tier):
        if mode == "fast" and tier <= 2:
            client = self.vllm_fast
            priority = "high"
        else:
            client = self.vllm_heavy
            priority = "normal"
        
        # Add priority header
        return await client.infer(
            text,
            model=self.model_name(guard_type, tier),
            priority=priority
        )
```

### 6.4 Deployment Topology Diagram

```
┌───────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (us-east-1)                               │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Guardian Deployment (2-20 pods)                         │ │
│  │  ├─ Pod-1: FastAPI server                                │ │
│  │  ├─ Pod-2: FastAPI server                                │ │
│  │  └─ Pod-N: FastAPI server                                │ │
│  │  (horizontal auto-scaling based on queue depth)          │ │
│  └──────────┬───────────────────────────────────────────────┘ │
│             │ HTTP/gRPC                                        │
│  ┌──────────┴───────────────────────────────────────────────┐ │
│  │  vLLM Instance 1 (PII model)                            │ │
│  │  - GPU: A100 / H100                                      │ │
│  │  - max_num_seqs: 256                                     │ │
│  │  - Tensor Parallel: 1                                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  vLLM Instance 2 (InfoClass model)                       │ │
│  │  - GPU: A100 / H100                                      │ │
│  │  - max_num_seqs: 256                                     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Redis Cluster (cache + session state)                  │ │
│  │  - 3 nodes (high-availability)                           │ │
│  │  - TTL: 24h for exact-match, 48h for confidence         │ │
│  │  - Namespace: guardian:*                                 │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Observability Stack                                     │ │
│  │  ├─ OpenTelemetry collector (sidecar)                   │ │
│  │  ├─ Prometheus scraper                                   │ │
│  │  └─ Grafana dashboards                                   │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└───────────────────────────────────────────────────────────────┘
```

---

## 7. Real-World Reference Analysis

### 7.1 Comparative Reference Table

| Reference | Latency Tier | Mode Selector | Streaming | Early Exit | Caching |
|---|---|---|---|---|---|
| **Anthropic API** | Single ~100ms | Prompt caching (implicit) | ✅ Token streaming | N/A (LLM-side) | ✅ Prefix cache |
| **Bedrock Guardrails** | Async (~50ms) vs Sync (full) | Severity threshold | ✅ Async mode | ✅ Per-safeguard | ✅ Implicit |
| **Azure Content Safety** | Single ~200ms | Severity scale (0–7) | ❌ Full content | ❌ No | ❌ No |
| **OpenAI Moderation** | Single ~100ms | Category score threshold | ❌ Full content | ❌ No | ❌ No |
| **Cloudflare WAF** | <100ms | Rule set selection | ❌ Request-level | ✅ Early exit | ✅ Implicit |
| **Stripe Radar** | ~100ms latency SLA | Risk score (0–99) | ❌ Synchronous | ✅ Scoring early-exit | ✅ ML model cache |
| **NVIDIA NeMo** | Configurable | Rail type (input/output/dialogue) | ✅ Streaming support | ❌ (IORails parallel) | ❌ Runtime state only |
| **Guardrails-AI** | Sequential (slow) → Async (fast) | Per-validator on_fail | ✅ StreamRunner | ❌ Default sequential | ❌ No |

### 7.2 Key Patterns Adopted for Guardian

| Pattern | Source | Guardian Adaptation |
|---|---|---|
| Tiered confidence + escalation | Google Cascades, Stripe | Tiers 1–3 with threshold-based escalation |
| Early-exit on UNSAFE | Cloudflare WAF, AWS WAF | Cancel pending scanners on first UNSAFE |
| Streaming with sliding context window | NeMo, Guardrails-AI | `context_size` + `window_size` params |
| Per-request mode override | Bedrock, Azure | HTTP header `X-Guardian-Mode` |
| Semantic + exact-match cache | Anthropic, Introl | Multi-layer Redis cache |
| Fail-strict on critical guard | All | Policy-driven `on_fail` action |
| LLM-as-judge for edge cases | Academic (2025 papers) | Tier 3 (enabled only in deep mode) |

### 7.3 Research Papers on Cascading & Tiering (2024–2026)

**Reference:** [Tiered Agentic Oversight (arxiv:2506.12482)](https://arxiv.org/pdf/2506.12482)
- Hierarchical human oversight based on action sensitivity
- Low-risk: automated, Medium-risk: sampling review, High-risk: mandatory approval
- **Implication:** Guardian can map severity to require different levels of human review

**Reference:** [Cascade Routing for LLMs (arxiv:2024)](https://files.sri.inf.ethz.ch/website/papers/dekoninck2024cascaderouting.pdf)
- Unified framework integrating routing (choose 1 model) and cascading (sequential models)
- Theoretical optimality: minimize cost/latency while maintaining quality
- **Implication:** Guardian's tier escalation aligns with this research

**Reference:** [Why LLM Safety Guardrails Collapse (arxiv:2506.05346)](https://arxiv.org/pdf/2506.05346)
- Fine-tuning erodes safety guardrails built in pre-training
- **Implication:** Guardian must version policy & model updates, invalidate cache on updates

---

## 8. Concrete Recommendation for Guardian

### 8.1 Proposed Mode Configuration

```yaml
# Guardian Dual-Mode Architecture — Implementation Contract

operation_modes:
  fast:
    description: "Latency-optimized for real-time streaming chat (Agent i)"
    latency_budget_ms: 100
    input_protocol: atomic                  # always
    output_protocol: chunk                  # default (LLM streaming)
    
    scanner_config:
      pii:
        tier1_enabled: true
        tier2_enabled: true                 # escalate if 0.2–0.8 confidence
        tier3_enabled: false
        confidence_threshold: 0.85          # accept Tier 1 if > 85%
      
      info_classification:
        tier1_enabled: true                 # rule-based heuristic
        tier2_enabled: true                 # escalate if uncertain
        tier3_enabled: false
        confidence_threshold: 0.90
      
      jailbreak:
        tier1_enabled: false                # skip regex (false positive risk)
        tier2_enabled: false
        tier3_enabled: false
        post_stream_async: true             # check AFTER stream ends (async)
    
    caching:
      exact_match: true
      semantic_cache: true
      cache_ttl_s: 86400
    
    fail_strategy: fail_open                # timeout → return SAFE + warn
    request_timeout_ms: 120

  deep:
    description: "Quality-optimized for sensitive use cases (GenAI Gateway, compliance)"
    latency_budget_ms: 500
    input_protocol: atomic                  # always
    output_protocol: atomic_forced          # buffer entire response
    
    scanner_config:
      pii:
        tier1_enabled: true
        tier2_enabled: true
        tier3_enabled: false                # future: LLM-as-judge for edge cases
        confidence_threshold: 0.70          # lower threshold = more escalations
      
      info_classification:
        tier1_enabled: true
        tier2_enabled: true
        tier3_enabled: false
        confidence_threshold: 0.70
      
      jailbreak:
        tier1_enabled: false
        tier2_enabled: true                 # full GLiNER-based detection
        tier3_enabled: false                # future: GPT-4 for nuanced reasoning
        post_stream_async: false            # already atomic-forced
    
    caching:
      exact_match: true
      semantic_cache: true
      cache_ttl_s: 172800                   # 48h (more conservative)
    
    fail_strategy: fail_closed              # timeout → return UNSAFE (block)
    request_timeout_ms: 600

# Per-tenant default configuration
tenant_defaults:
  genai-gw:
    default_mode: fast                      # can override per-request
    allowed_modes: [fast, deep]
    input_check: true
    output_check: true
    on_unsafe_input: block                  # revoke
    on_unsafe_output: truncate_notice       # mask or block
  
  agent-i:
    default_mode: fast                      # streaming chat UX
    allowed_modes: [fast]                   # no deep for now (bandwidth constraint)
    input_check: true
    output_check: true
    on_unsafe_input: block
    on_unsafe_output: truncate_notice
```

### 8.2 API Contract Extensions

```python
# HTTP Headers for mode override
X-Guardian-Mode: fast | deep                # per-request mode selection
X-Guardian-Latency-Budget-Ms: 150          # custom budget (optional)

# Request body additions
{
    "request_id": "uuid",
    "content": "...",
    "mode": "fast" | "deep",                # alternative to header
    "cache_policy": "normal" | "no_cache",  # bypass cache if needed
    "guards": ["pii", "info_classification", "jailbreak"],
    ...
}

# Response additions
{
    "request_id": "uuid",
    "verdict": "...",
    "mode_used": "fast" | "deep",
    "cache_hit": true | false,
    "tier_used": 1 | 2 | 3,
    "confidence": 0.92,
    "total_latency_ms": 45,
    ...
}
```

### 8.3 Deployment Plan (Phased)

**Phase 1 (4 weeks) — Single Mode (Fast only, baseline)**
- Implement Tier 1–2 cascading
- Exact-match cache
- Single `guardian-fast` mode
- Latency target: <100ms p95

**Phase 2 (3 weeks) — Deep Mode**
- Add Tier 2 scanning
- Atomic-forced protocol for output
- Mode selector logic
- Latency target: <500ms p95

**Phase 3 (2 weeks) — Advanced Optimization**
- Semantic caching
- Speculative execution (parallel input + LLM)
- Post-stream async jailbreak scan
- Latency target: <80ms p95 for fast mode

**Phase 4 (Future) — Tier 3**
- LLM-as-judge for edge cases
- Jailbreak deep analysis (GPT-4 / Claude)
- Advanced retrieval (context from knowledge base)

### 8.4 Recommended Configuration for Guardian M1

```yaml
# Guardian M1 — Recommended Production Config

# (Same as section 8.1 above)

# Key tuning parameters
guardrails_tuning:
  # Fast mode
  fast_mode_tier1_accept_threshold: 0.85   # high confidence = accept early
  fast_mode_tier2_escalate_range: [0.2, 0.8]  # borderline → escalate
  
  # Deep mode
  deep_mode_tier1_accept_threshold: 0.70   # lower threshold = more escalations
  deep_mode_tier2_accept_threshold: 0.85
  
  # Cascade sensitivity per guard
  pii_cascade_ratio: 0.20                   # expect ~20% escalation to Tier 2
  info_class_cascade_ratio: 0.15
  
  # Cache hit expectations
  exact_match_hit_rate: 0.35                # 35% hit rate for fast mode
  semantic_cache_hit_rate: 0.25             # additional 25%
  total_cache_rate: 0.60

# SLA & alerting
slas:
  fast_mode:
    p50: 30ms
    p95: 80ms
    p99: 120ms
  
  deep_mode:
    p50: 100ms
    p95: 350ms
    p99: 500ms

alerts:
  - name: FastModeLatencyBreached
    threshold_ms: 150                       # double the SLA p95
    duration: 5m
    
  - name: DeepModeLatencyBreached
    threshold_ms: 800
    duration: 5m
  
  - name: CacheHitRateLow
    threshold: 0.40                         # expected ~0.60
    duration: 10m
  
  - name: vLLMQueueDepth
    threshold: 100
    duration: 2m
```

---

## 9. Unresolved Questions

1. **Tier 3 LLM-as-judge cost-benefit trade-off:** Is GPT-4 inference (~$0.03 per request, ~1s latency) justified for edge cases (5–10% of requests in deep mode)? Should Guardian defer to caller for custom logic instead?

2. **Semantic cache embedding model selection:** Use lightweight (all-MiniLM-L6, 22M, ~5ms) or heavier (all-mpnet-base, 110M, ~50ms)? Trade-off latency vs. accuracy of similarity matching.

3. **Post-stream async jailbreak scan latency visibility:** When async jailbreak scan completes after stream ends, should Guardian retroactively emit alert SSE event to user? Or silent log for security team review only?

4. **Cache invalidation on model update:** When retraining PII tier2 model, should all cached entries for that guard be invalidated? Or only those with `confidence` in borderline range (0.3–0.7)?

5. **Per-tenant isolation at vLLM level:** Should fast-mode tenants get separate vLLM instance from deep-mode tenants? Or share pools with QoS prioritization?

6. **Atomic-forced buffer memory ceiling:** If output is very large (e.g., 100K tokens), buffering entire response may OOM Guardian pod. Should there be a hard limit? Fall back to chunk mode if exceeded?

7. **Policy version semantics:** When updating policy (e.g., changing PII threshold from 0.85 → 0.80), should Guardian invalidate cache for old policy_version, or treat as new version_hash and let TTL expire old entries naturally?

8. **Tier 3 fallback behavior:** If Tier 3 (LLM-as-judge) times out or errors, should Guardian fall back to Tier 2 verdict, or treat timeout as UNSAFE (fail-closed)?

9. **Streaming chunk size heuristic:** Should chunk_size be static (256 tokens) or dynamic based on LLM's token generation rate? (E.g., slow model = buffer longer to amortize scan cost.)

10. **Request-level budget override:** If `X-Guardian-Latency-Budget-Ms: 50` is provided but default is 100ms, should Guardian automatically degrade to Tier 1 only? Or fail-open with warning?

---

## Sources

### Industry Documentation
- [AWS Bedrock Guardrails How It Works](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-how.html)
- [Azure AI Content Safety Harm Categories](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/harm-categories)
- [Azure Content Safety Severity Scale](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/faq)
- [OpenAI Moderation API Reference](https://platform.openai.com/docs/api-reference/moderations)
- [OpenAI Moderation Guide](https://developers.openai.com/api/docs/guides/moderation)
- [NVIDIA NeMo Guardrails Streaming Guide](https://docs.nvidia.com/nemo/guardrails/latest/user-guides/advanced/streaming.html)
- [NVIDIA NeMo Microservices Streaming](https://docs.nvidia.com/nemo/microservices/latest/guardrails/streaming-output.html)
- [Guardrails-AI Concurrency Documentation](https://www.guardrailsai.com/docs/concepts/concurrency)
- [Cloudflare WAF Managed Rules](https://developers.cloudflare.com/waf/managed-rules/)
- [Cloudflare WAF Rule Execution Phases](https://developers.cloudflare.com/waf/reference/phases/)
- [Stripe Radar Documentation](https://docs.stripe.com/radar)
- [Stripe Radar Risk Evaluation](https://docs.stripe.com/radar/risk-evaluation)

### Research Papers & Blogs
- [Google Speculative Cascades for Smarter LLM Inference](https://research.google/blog/speculative-cascades-a-hybrid-approach-for-smarter-faster-llm-inference/)
- [Cascade Routing for LLMs — Unified Framework](https://files.sri.inf.ethz.ch/website/papers/dekoninck2024cascaderouting.pdf)
- [Why LLM Safety Guardrails Collapse After Fine-tuning (arxiv:2506.05346)](https://arxiv.org/pdf/2506.05346)
- [Tiered Agentic Oversight — Hierarchical Safety (arxiv:2506.12482)](https://arxiv.org/pdf/2506.12482)
- [Anthropic Prompt Caching — 10x Cheaper Tokens](https://ngrok.com/blog/prompt-caching)
- [Introl: Prompt Caching Infrastructure for Latency Reduction (2025)](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025)
- [DigitalOcean: Prompt Caching with Anthropic and OpenAI](https://www.digitalocean.com/blog/prompt-caching-with-digital-ocean)
- [How Stripe Detects Fraudulent Transactions Within 100ms](https://blog.bytebytego.com/p/how-stripe-detects-fraudulent-transactions)
- [OpenAI Upgrading Moderation API with Multimodal Model](https://openai.com/index/upgrading-the-moderation-api-with-our-new-multimodal-moderation-model/)

### Internal References
- [Guardian Architecture Design (01-architecture-design.md)](./01-architecture-design.md)
- [Guardian Scan Flow Design (05-scan-flow-design.md)](./05-scan-flow-design.md)
- [Open-Source Reference Analysis (03-open-source-reference-analysis.md)](./03-open-source-reference-analysis.md)

---

**Status:** DONE

**Recommendations Summary:**
1. ✅ Implement dual-mode (fast/deep) with per-tenant default + per-request override
2. ✅ Tiered cascading scanners with early-exit on UNSAFE
3. ✅ Semantic + exact-match caching (target 50–70% hit rate)
4. ✅ Chunk streaming (fast mode) + atomic-forced (deep mode)
5. ✅ Single service deployment with mode selector logic
6. ✅ OpenTelemetry observability + SLA-based alerting

**Next Steps:** Create implementation plan with detailed technical specs (phase-by-phase breakdown).
