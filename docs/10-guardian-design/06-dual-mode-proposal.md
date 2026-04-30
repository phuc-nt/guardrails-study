# Guardian — Dual-Mode Proposal (Fast vs Deep)

**Version:** 0.1 | **Date:** 2026-04-30
**Scope:** Đề xuất Guardian hỗ trợ 2 chế độ vận hành trên cùng một service: **fast** (tối ưu latency) và **deep** (tối ưu chất lượng).
**Audience:** Tech Lead, Product Owner GenAI Gateway / Agent i, CISO

> Nguồn research đầy đủ: [`plans/reports/researcher-260430-1706-dual-mode-guardrail-architecture.md`](../../plans/reports/researcher-260430-1706-dual-mode-guardrail-architecture.md)
> Bổ sung cho: [01-architecture-design.md](./01-architecture-design.md), [05-scan-flow-design.md](./05-scan-flow-design.md)

---

## 1. Vì sao cần 2 mode

| Use case | Đặc tính | Mode phù hợp |
|---|---|---|
| Agent i consumer chat (streaming, B2C) | Latency-critical, traffic lớn, content phổ thông | **fast** |
| GenAI GW: code review, sensitive doc, compliance prompt | Chấp nhận chờ vài trăm ms, cần đầy đủ check | **deep** |
| GenAI GW: chat thường của engineer | Cân bằng | **fast** (default) |

Một mức scanner cố định **không phục vụ tốt cả 2** — quá chậm cho consumer, quá nông cho compliance.

---

## 2. Định nghĩa 2 mode

| Khía cạnh | **fast** | **deep** |
|---|---|---|
| Latency budget (Guardian overhead) | ≤ 100ms p95 | ≤ 500ms p95 |
| Output protocol | `chunk` (sliding window) | `atomic_forced` (buffer full response) |
| Scanner tier dùng | T1 + T2 | T1 + T2 (T3 reserve) |
| Confidence threshold escalate T1→T2 | 0.85 (early-exit cao) | 0.70 (escalate nhiều hơn) |
| Cache | exact + semantic, TTL 24h | exact + semantic, TTL 48h |
| Fail strategy on timeout | **fail-open** + warn header | **fail-closed** (block) |
| Async post-stream jailbreak scan | có | không cần (đã atomic) |

**Scanner tier:**
- **T1** — regex / keyword / TinyBERT, ~5-20ms
- **T2** — full classifier (DistilBERT-base + LoRA), ~50-100ms
- **T3** — LLM-as-judge (GPT-4 / Claude), ~500-2000ms — **reserve cho future, không bật ở M1**

---

## 3. Cách chọn mode

**Layered selection:**

1. **Per-tenant default** ở policy store (`genai-gw: fast`, `agent-i: fast`)
2. **Per-request override** qua HTTP header `X-Guardian-Mode: fast | deep`
3. **Allowed modes** giới hạn theo tenant (vd `agent-i.allowed_modes = [fast]` để cấm deep mode tốn budget)
4. **Optional**: header `X-Guardian-Latency-Budget-Ms` cho client tự định budget

```yaml
tenant_defaults:
  genai-gw:
    default_mode: fast
    allowed_modes: [fast, deep]
  agent-i:
    default_mode: fast
    allowed_modes: [fast]   # M1 chưa bật deep cho consumer
```

---

## 4. Kỹ thuật chốt áp dụng

| # | Kỹ thuật | Áp dụng cho | Hiệu quả |
|---|---|---|---|
| 1 | **Tiered cascading + early-exit** — chạy T1 trước, escalate T2 chỉ khi confidence trong vùng borderline | cả 2 mode | -50–70% latency vs always-T2 |
| 2 | **Speculative execution** — fork song song input check + LLM call, kill LLM nếu UNSAFE | fast mode | hide hầu hết Guardian latency |
| 3 | **Multi-layer cache** (exact-match Redis + semantic embedding) | cả 2 mode | hit rate kỳ vọng 50–70%, avg ~10ms |
| 4 | **Chunk streaming + sliding window** | fast output | UX gần real-time, có context bridging |
| 5 | **Atomic-forced** — buffer full LLM output rồi scan 1 lần | deep output | đảm bảo policy yêu cầu full content (vd PHI compliance) |
| 6 | **Post-stream async jailbreak scan** | fast output | check sâu mà không block stream; alert retro nếu fail |
| 7 | **Budget-based scheduler** — abort tier escalation khi gần hết budget | cả 2 mode | đảm bảo SLA p99 |

Chi tiết flow đã có ở [05-scan-flow-design.md](./05-scan-flow-design.md) — proposal này thêm **trục mode** lên trên 3 trục cũ (option × protocol × action).

---

## 5. API contract đề xuất (delta vs hiện tại)

```http
POST /v1/check/input
X-Guardian-Mode: fast              # optional, override tenant default
X-Guardian-Latency-Budget-Ms: 80   # optional
```

```jsonc
// Request body bổ sung (alternative to header)
{
  "mode": "fast",                   // "fast" | "deep"
  "cache_policy": "normal",         // "normal" | "no_cache"
  ...
}

// Response bổ sung (telemetry cho client)
{
  "verdict": "SAFE",
  "mode_used": "fast",
  "tier_used": 1,
  "cache_hit": true,
  "confidence": 0.92,
  "total_latency_ms": 12,
  ...
}
```

Header > body field nếu cả 2 cùng có. Validate `mode ∈ tenant.allowed_modes` ở entry; reject 400 nếu không hợp lệ.

---

## 6. SLA và alert đề xuất

| Mode | p50 | p95 | p99 | Alert breach |
|---|---|---|---|---|
| fast | 30ms | 80ms | 120ms | >150ms / 5min |
| deep | 100ms | 350ms | 500ms | >800ms / 5min |

Alert thêm: cache hit rate < 0.40 / 10min (kỳ vọng ~0.60), vLLM queue depth > 100 / 2min.

---

## 7. Rollout đề xuất (phased)

| Phase | Scope | Latency target |
|---|---|---|
| 1 | fast mode only, T1+T2 cascading, exact cache | <100ms p95 |
| 2 | thêm deep mode + atomic-forced + mode selector | <500ms p95 (deep) |
| 3 | semantic cache + speculative exec + post-stream async jailbreak | <80ms p95 (fast) |
| 4 (future) | T3 LLM-as-judge cho edge case (deep mode opt-in) | — |

---

## 8. Tham chiếu industry (để team nội bộ check khi triển khai)

- **AWS Bedrock Guardrails** — sync vs async streaming mode, severity threshold
- **Azure Content Safety** — severity scale 0–7
- **OpenAI Moderation** — per-category score, client tự chọn threshold
- **NVIDIA NeMo Guardrails** — input/output/dialogue rails, IORails parallel
- **Cloudflare WAF** — early-exit rule eval
- **Stripe Radar** — 100ms latency SLA + tiered ML detector
- **Anthropic Prompt Caching** — prefix cache để giảm token cost

Chi tiết comparison + paper references ở report nguồn.

---

## 9. Open questions cần tech lead chốt

1. **T3 (LLM-as-judge) có nên có ở M1?** Cost ~$0.03/req + ~1s latency cho 5–10% deep request — đáng không, hay defer Phase 4?
2. **Semantic cache embedding model** — `all-MiniLM-L6` (22M, ~5ms) hay `all-mpnet-base` (110M, ~50ms)?
3. **Async jailbreak alert UX** — nếu phát hiện UNSAFE sau khi stream end, có gửi SSE event retroactive về client không, hay chỉ log nội bộ?
4. **Cache invalidation policy** — khi update model T2, invalidate toàn bộ entry hay chỉ borderline confidence (0.3–0.7)?
5. **vLLM isolation** — tách pool fast/deep hay shared với QoS priority?
6. **Atomic-forced memory ceiling** — output 100K token có nguy cơ OOM, đặt hard limit và fallback chunk?
7. **Per-request budget override** — nếu client gửi budget < default, auto-degrade về T1-only hay reject?
8. **`agent-i` có nên mở `deep` mode** cho subset use case (vd Domain Agent finance) không?
