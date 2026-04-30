# Streaming LLM Guardrail Patterns: Chunk Scanning & Decision Strategies

**Date:** 2026-04-16 | **Status:** Research Complete | **Scope:** Guardian Phase 1 streaming design

---

## Executive Summary

Streaming guardrails require **two-phase validation**: chunk-level scanning (near-realtime, low latency) + optional full-response scanning (high-precision, deferred). Production systems employ **sliding window buffers** (50–256 tokens), **64-byte overlap** for entity boundary handling, and **early termination** when safety threshold breached. Input/output checks serve different purposes—input prevents leakage, output detects hallucinated PII—and require separate decision trees.

---

## 1. Chunk-Based Scanning with Sliding Window

### 1.1 Buffer Sizes & Token Windows (Production Patterns)

**NVIDIA NeMo (production baseline):**
- `context_size`: 50 tokens default (recent context for validation)
- `chunk_size`: 200 tokens typical (balance latency vs context)
- Alternative: 128 tokens for low-latency, 256 tokens for hallucination detection [1]
- Sliding window validates incrementally as buffer fills; no full response wait

**AWS Bedrock Moderation:**
- Break streaming into ~1,000 character batches (≈250 tokens per batch)
- Full response < 25 units (25K chars) = single batch; larger = multi-batch pipeline [2]
- Each batch evaluated separately, with results aggregated per policy

**LiteLLM/Azure Content Safety:**
- Hard limit: 10,000 chars per chunk (auto-split at word boundaries)
- Configurable `severity_threshold` per category, OR `severity_threshold_by_category` dict [3]

### 1.2 Overlap & Boundary Handling

**Cross-chunk entity detection:**
- Sliding window overlap: **64-byte buffer** recommended for contextual PII (names, addresses)
- Structured PII (credit cards, emails): ~30-char lookahead sufficient
- Flush trigger: sentence boundary (period/newline), NOT character limit [4]

**Implementation pattern:**
```
Chunk N: "John Smith works at..."
Overlap: "...works at..." (carries forward context)
Chunk N+1: "...Apple. His email is..."
→ PII detector sees "John Smith" + context → HIGH confidence
→ If split mid-name: "John Sm" + "ith" → buffered via overlap
```

**Cost efficiency (Guardian-specific):**
- Scan chunk N + overlap of N-1 = detect ~95% violations by seeing first 18% of tokens [5]
- Avoid re-scanning full history; maintain running entity detection state

---

## 2. Decision Patterns When Chunk UNSAFE

### 2.1 Release-First-Then-Recall (SSE Architecture)

**When applicable:** Output streaming, low-compliance regulations, user experience prioritized

**Flow:**
```
Chunk 1: scan → SAFE → stream to user immediately
Chunk 2: scan → SAFE → stream
Chunk 3: scan → UNSAFE (PII detected) → Block chunk 3+
                                      → Send "[Response truncated - policy violation]"
```

**Cost:** User sees partial response; must request retry or re-prompt LLM.
**Production examples:** Anthropic Claude (streaming with post-hoc safety), OpenAI GPT (async moderation) [6][7]

### 1.2 Delay-Until-Safe (Buffered Release)

**When applicable:** High-compliance, short responses (< 512 tokens), latency tolerance

**Flow:**
```
Buffer 256 tokens → Scan (parallel PII + InfoClass)
                  → UNSAFE → Block, return safe message
                  → SAFE → Release tokens to user
```

**Latency cost:** +50–150ms first-token latency (acceptable for high-stakes use cases)
**Production examples:** Enterprise compliance, regulated industries [8]

### 2.3 Partial Masking (Chunk-Level)

**When applicable:** PII detected but response salvageable, masking policy defined

**Pattern:**
```
Chunk: "Contact John Smith at john@company.com"
Scan: PII detected (PERSON, EMAIL)
Mask: "Contact [PERSON] at [EMAIL]"
Release: masked chunk to user
```

**Requirement:** Pre-define masking rules per entity type + context (names vs credentials vs emails have different risk profiles)

---

## 3. Full-Content Scanning Triggers

### 3.1 When Full-Response Scan Is Mandatory

**Jailbreak/prompt-injection detection:**
- RLM-JB framework: chunks input into sub-sequences, analyzes procedurally vs single-pass [9]
- Rationale: Long-context hiding + semantic camouflage evades chunk-level detection
- Scan: Full prompt/response normalized, de-obfuscated, cross-chunk correlation performed

**Document-level classification:**
- Hierarchical info classification (public → top_secret) may require aggregate semantic signal
- Single chunk "secret" level unlikely; multi-chunk patterns (technical diagrams + code + credentials) composite → "top_secret"

**Compliance audits:**
- Scan entire session/conversation for policy violations (for post-hoc audit/logging)
- Not per-request blocking; fire-and-forget analysis

### 3.2 Hybrid Pattern: Stream + Defer Full Scan

**Approach (recommended for Guardian):**

1. **Streaming phase (online):**
   - Chunk-based PII + InfoClass scanning (50ms per 256-token chunk)
   - Release tokens immediately if SAFE
   - Block/truncate if chunk UNSAFE

2. **Post-streaming phase (offline, async):**
   - Full-content jailbreak/injection analysis (via RLM-JB or semantic embeddings)
   - Classification confirmation (is full response truly "secret"?)
   - Audit logging (which chunks raised flags)

**Latency:** Streaming unaffected; full scan async, results available in Cloudwatch/audit logs

---

## 4. Input vs Output Check Strategies

### 4.1 Input-Only Pattern (Data Leakage Prevention)

**Purpose:** Block dangerous prompts before LLM sees them.

**Checks:**
- PII detection: email, credit card, SSN, API keys in prompt
- Prompt injection: SQL, code injection patterns
- Credential detection: AWS key format, database URL

**Decision:** Fail-closed. If UNSAFE → block, return "prompt policy violation"

**Why separate:** Input leakage compounds—if user accidentally sends database URL to LLM, it enters model cache, logs, fine-tuning data. One-way prevention > later recovery.

**Cost benefit:** Block before LLM call saves compute and token cost [10]

### 4.2 Output-Only Pattern (Hallucinated Content Detection)

**Purpose:** Catch PII, secrets, toxic content LLM generates (real or hallucinated).

**Checks:**
- PII detection on generated text (model may invent plausible-looking SSN, email)
- Toxic/harmful content generation
- Factual hallucination (claims falsehoods as fact)

**Decision:** Fail-open or fail-masked. If UNSAFE → mask, truncate, or retry prompt

**Why separate:** LLM can generate convincing fake PII that blocks filtering (looks real but is fabricated). Output-only rules must account for hallucinated + legitimate data.

**Example:** LLM output "Send invoice to janedoe@example.com" → PII detector flags email → Guardian masks → "Send invoice to [EMAIL]". Correct action (email is hallucinated, not real leakage).

### 4.3 Both Input + Output (Full Pipeline)

**Recommended default for Guardian Phase 1.**

```
Input check: PII + InfoClass
  ├─ UNSAFE → block, don't call LLM
  └─ SAFE → proceed to LLM

LLM response streaming:
Output check: PII + InfoClass (chunk-based)
  ├─ UNSAFE → truncate/mask chunk
  └─ SAFE → stream to user
```

**Why both:** Layered defense.
- Input checks catch obvious leakage (cost-efficient)
- Output checks catch hallucination + subtle injection (completeness)
- Parallel input check (ẩn behind LLM call) → zero latency overhead [11]

---

## 5. Architectural Patterns from Production

### 5.1 Two-Phase Executor (Detection vs Transformation)

**From NVIDIA NeMo warning:** Parallel detection + serial masking [12]

```python
# Phase 1: Detection (parallel, read-only)
detections = await asyncio.gather(
    pii_detect(text),        # Returns entities, no mutation
    info_classify(text),     # Returns level + confidence
)

# Phase 2: Transformation (serial, mutation)
if detections[0].unsafe:
    masked_text = apply_mask(text, detections[0].entities)
    return masked_text
```

**Why:** Parallel detection adds zero latency (both models scan same content). Masking is serial (must accumulate entity positions without races).

### 5.2 StreamRunner Pattern (Guardrails-AI)

**chunking strategy:**
```yaml
streaming:
  enabled: true
  chunk_size: 256         # tokens per chunk
  context_size: 50        # sliding window context
  stream_first: true      # send tokens immediately, buffer async
```

**Flow:** Tokens flow to user while being buffered. Scan triggers when buffer ≥ chunk_size. If violation in chunk N → block N+, send truncation notice [2]

### 5.3 Early Stopping on Safety Threshold

**LiteLLM/Azure pattern:**
```python
violation_score = 0.0
for chunk in streaming_response:
    score = await scan(chunk)
    violation_score += score
    if violation_score >= THRESHOLD:
        # Terminate stream, send safety notice
        return truncation_response()
```

**Threshold tuning:** Start conservative (70% confidence), adjust per policy. High false-positive rate → user friction. [3]

---

## 6. Concrete Recommendations for Guardian

### 6.1 Streaming Mode Defaults

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `chunk_size` | 256 tokens | Balance latency (50ms) vs context (multi-sentence entity) |
| `context_size` | 50 tokens | Overlap buffer for cross-chunk PII |
| `overlap_bytes` | 64 bytes | Reconstructs split PII (names, emails) |
| `stream_first` | true | Release tokens immediately, guard async |
| `flush_trigger` | sentence boundary | Avoid mid-word chunking |

### 6.2 Decision Tree

```
Input Check:
  ├─ PII UNSAFE → Block (fail-closed)
  ├─ InfoClass ≥ Secret → Block (fail-closed)
  └─ SAFE → Proceed to LLM

Output Check (streaming):
  ├─ Chunk PII UNSAFE → Mask chunk, continue stream
  ├─ Chunk InfoClass ≥ Secret → Truncate + notice
  ├─ Full-response jailbreak detected (async) → Log alert, don't block (already streamed)
  └─ SAFE → Stream chunk, continue
```

### 6.3 Implementation Priorities

1. **Week 1-2:** Implement chunk scanner (256-token buffer) with overlap handling. Test PII boundary splits.
2. **Week 3:** Add async full-content jailbreak scan (post-stream, RLM-JB or embedding-based).
3. **Week 4:** Integrate InfoClass hierarchical rules (public → secret decision tree).

---

## 7. Open Questions for Guardian Policy

1. **Masking vault persistence:** When PII masked → store masked→original mapping in Redis? Who can deprotect? (Privacy/GDPR implications)
2. **Hybrid input/output SLA:** Input blocks immediately (latency-free). Output scanning adds ~50ms. Is this acceptable for peak load (340 req/sec)?
3. **Full-content jailbreak frequency:** Scan every response or sample (e.g., 5%)? Estimate cost at 20K users.
4. **Retries on mask:** When output masked or truncated → auto-retry with safer prompt template, or require user re-prompt?
5. **Chunk UNSAFE notification:** Send user "[Response truncated - policy violation]" or vague "[Response ended]"? UX/compliance trade-off.

---

## Sources

[1] https://developer.nvidia.com/blog/stream-smarter-and-safer-learn-how-nvidia-nemo-guardrails-enhance-llm-output-streaming/

[2] https://aws.amazon.com/blogs/machine-learning/use-the-applyguardrail-api-with-long-context-inputs-and-streaming-outputs-in-amazon-bedrock/

[3] https://docs.litellm.ai/docs/proxy/guardrails/quick_start

[4] https://dev.to/yemrealtanay/how-i-built-a-pii-tokenization-middleware-to-keep-sensitive-data-out-of-llm-apis-18po

[5] https://arxiv.org/html/2506.09996v1 (Early Stopping LLM Harmful Outputs via Streaming Content Monitoring)

[6] https://platform.claude.com/docs/en/about-claude/use-case-guides/content-moderation

[7] https://developers.openai.com/api/docs/guides/moderation

[8] https://www.guardrailsai.com/docs/how_to_guides/enable_streaming

[9] https://arxiv.org/abs/2602.16520v1 (RLM-JB: Recursive Language Models for Jailbreak Detection)

[10] https://www.truefoundry.com/blog/ai-guardrails

[11] /Users/phucnt/workspace/guardrails-study/docs/01-architecture-design.md (Section 2.1: Parallel Pattern)

[12] /Users/phucnt/workspace/guardrails-study/docs/03-open-source-reference-analysis.md (Section 2.5: IORails Pattern)

---

## Unresolved Questions

- Guardian's masking vault strategy (ephemeral vs persistent)?
- Frequency of full-content jailbreak scanning at scale?
- User notification UX when output truncated?
- Policy on auto-retry vs manual re-prompt after mask/truncate?
