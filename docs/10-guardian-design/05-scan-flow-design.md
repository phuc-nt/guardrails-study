# Guardian — Scan Function Flow Design

**Version:** 0.2 (Draft) | **Date:** 2026-04-30
**Scope:** Thiết kế function flow cho mọi tổ hợp check-point / protocol / action pattern, áp dụng cho cả 2 consumer Milestone 1 (GenAI Gateway, Agent i)
**Audience:** Engineering team, Tech Lead, Product owner GenAI Gateway / Agent i, CISO

> Tài liệu này bổ sung cho [01-architecture-design.md](./01-architecture-design.md) và [04-top-5-adopt-patterns-explained.md](./04-top-5-adopt-patterns-explained.md). Tập trung vào **cơ chế scan** từng use case, không đi sâu vào infra/deployment.

> **Bối cảnh:** Guardian là milestone đầu của Guardrail Platform toàn tập đoàn LY Corp. M1 phục vụ song song:
> - **GenAI Gateway** (B2E) — proxy giữa agent của 20K engineer ↔ external LLM. Đa phần atomic, latency tolerant.
> - **Agent i** (B2C) — unified AI agent (Yahoo! JAPAN + LINE), 7 Domain Agents, có memory + task execution. Đa phần streaming chunk, latency khắt khe. Ref: https://www.lycorp.co.jp/en/news/release/020398/
>
> Mọi flow dưới đây đều phải multi-tenant: config / policy / scanner pack chọn theo `tenant_id` (`genai-gw` | `agent-i`).

---

## 1. Mục đích

Guardian phải phủ được tất cả tổ hợp giữa:
- **Điểm check**: input only / output only / cả 2
- **Giao thức truyền**: trọn khối (atomic) / streaming theo chunk
- **Hành vi xử lý kết quả**: masking / classify (dán nhãn rồi cho đi) / revoke (chặn)

Tài liệu mô tả function flow cho từng tổ hợp để stakeholder (eng, security, product) cùng đồng thuận cơ chế trước khi vào implement.

---

## 2. Thuật ngữ — 3 trục độc lập

### 2.1 Trục A — Check Point Option

| Option | Mô tả | Use case điển hình |
|---|---|---|
| **input-only** | Chỉ scan prompt từ user trước khi tới LLM | Prevent data leakage, credential protection, prompt injection filter. Caller tin tưởng LLM output. |
| **output-only** | Chỉ scan response từ LLM trước khi trả user | Detect hallucinated PII, LLM-generated toxic content. Caller đã tự validate input. |
| **both** | Scan cả input và output | Default cho high-stakes service, defense-in-depth |

**Quy tắc:** Option này do **caller** (GenAI Gateway policy per-API-key) quyết định, không phải Guardian.

### 2.2 Trục B — Transport Protocol

| Protocol | Mô tả | Latency | Khi nào bắt buộc |
|---|---|---|---|
| **atomic** (non-chunk) | Scan một khối nội dung trọn vẹn, 1 lần call | Cao (chờ đủ content) | Input (user đã gửi full prompt); output scan yêu cầu full context (jailbreak detection, hierarchical document classification) |
| **chunk** (streaming) | Scan theo từng chunk đi qua, sliding window | Thấp (near real-time) | Output streaming SSE, user cần phản hồi token-by-token |

**Quy tắc:**
- Input **mặc định là atomic** (user gửi trọn prompt mới call API).
- Output có thể chọn atomic hoặc chunk tùy LLM response mode (non-streaming vs streaming).
- **Một số loại scan buộc phải atomic kể cả khi output streaming** — xem Section 6.

### 2.3 Trục C — Action Pattern (per scanner result)

Mỗi scanner (PII, InfoClass, Jailbreak, ...) trả ra một `ScanResult`. Action pattern quyết định làm gì với kết quả đó.

| Pattern | Mô tả | Output | Ai quyết định |
|---|---|---|---|
| **masking** | Thay thế phần vi phạm bằng placeholder, cho nội dung đi tiếp | `sanitized_content` | Scanner + policy (per entity type) |
| **classify** | Dán nhãn phân loại vào response, cho nội dung đi tiếp (không sửa) | `classification_label` + `confidence` | Scanner; caller xử lý downstream |
| **revoke** | Chặn hoàn toàn, không cho nội dung qua | `block_reason` | Policy khi `level ≥ threshold` hoặc scanner fail-closed |

**Lưu ý:** 3 pattern này **không loại trừ nhau** — một request có thể qua 2+ scanner, mỗi scanner trả pattern khác nhau. Ví dụ: PII scanner trả `masking` + InfoClass scanner trả `classify`. Guardian aggregate rồi gửi cho caller.

---

## 3. Main Flow — Toàn cảnh

```
┌────────────────────────────────────────────────────────────────────┐
│  User / Agent / GenAI Gateway                                       │
└────────────────────────────┬───────────────────────────────────────┘
                             │ prompt
                             ▼
                ┌──────────────────────────┐
                │  [INPUT CHECK]            │ ◄── option: input-only | both
                │  Guardian /v1/check/input │
                │  (atomic only)            │
                └────────────┬─────────────┘
                             │
                  verdict + actions
                             │
             ┌───────────────┼───────────────┐
             │               │               │
           revoke         masking         classify
             │               │               │
             ▼               ▼               ▼
        Return block    Forward masked   Forward as-is
        to user         prompt to LLM   + classification
             │               │          label → LLM
             │               │               │
             └───────┬───────┴───────────────┘
                     ▼
           ┌─────────────────────┐
           │  External LLM        │
           │  (Claude/GPT)        │
           └──────────┬───────────┘
                      │ response (atomic OR streaming chunks)
                      ▼
                ┌──────────────────────────┐
                │  [OUTPUT CHECK]           │ ◄── option: output-only | both
                │  Guardian /v1/check/output│
                │  (atomic OR chunk mode)   │
                └────────────┬─────────────┘
                             │
                   verdict + actions per chunk/blob
                             │
             ┌───────────────┼───────────────┐
             │               │               │
           revoke         masking         classify
             │               │               │
             ▼               ▼               ▼
        Truncate +     Forward masked  Forward as-is
        notice         content          + label
             │               │               │
             └───────┬───────┴───────────────┘
                     ▼
                ┌─────────┐
                │  User   │
                └─────────┘
```

### 3.1 Visual glossary — 3 options cạnh nhau

```
  INPUT-ONLY              OUTPUT-ONLY             BOTH (recommended)
  ──────────              ───────────             ──────────────────

   ┌──────┐                ┌──────┐                 ┌──────┐
   │ User │                │ User │                 │ User │
   └──┬───┘                └──┬───┘                 └──┬───┘
      │                       │                        │
      ▼                       ▼                        ▼
  ┌───────┐               ┌───────┐                ┌───────┐
  │Caller │               │Caller │                │Caller │
  └───┬───┘               └───┬───┘                └───┬───┘
      │                       │              ┌─────────┴─────────┐
      ▼                       │              │                   │
  ╔═══════╗                   │              ▼                   ▼
  ║ INPUT ║                   │          ╔═══════╗           ┌─────┐
  ║ CHECK ║                   │          ║ INPUT ║           │ LLM │
  ╚═══╤═══╝                   │          ║ CHECK ║           └──┬──┘
      │ (SAFE?)               │          ╚═══╤═══╝              │
      ▼                       ▼              │                  │
   ┌─────┐                 ┌─────┐           └─► if UNSAFE       │
   │ LLM │                 │ LLM │               cancel LLM      │
   └──┬──┘                 └──┬──┘                               │
      │                       │                                  │
      │                       ▼                                  ▼
      │                   ╔════════╗                         ╔════════╗
      │                   ║OUTPUT  ║                         ║OUTPUT  ║
      │                   ║CHECK   ║                         ║CHECK   ║
      │                   ╚════╤═══╝                         ╚════╤═══╝
      ▼                        ▼                                  ▼
   ┌──────┐                ┌──────┐                           ┌──────┐
   │ User │                │ User │                           │ User │
   └──────┘                └──────┘                           └──────┘

  Check: 1×                Check: 1×                          Check: 2×
  Latency: ~0ms*           Latency: +50-150ms                 Latency: ~50-150ms**
  Block-ability: before    Block-ability: after               Block-ability: both
                LLM call                 LLM call
```

\* input check parallel behind LLM call cho option `both`, nhưng `input-only` không có LLM chờ để ẩn → vẫn mất ~100ms.
** output check chiếm hầu hết overhead; input check ẩn song song với LLM.

### 3.2 Visual glossary — Protocol side-by-side

```
  ATOMIC (non-chunk)           CHUNK (streaming)            ATOMIC-FORCED
  ──────────────────           ─────────────────            ─────────────

  content = full blob          content = chunk 1,2,3,...    content = full blob
  1 scan call                  N scan calls                 1 scan call
  wait all → scan → emit       scan each → emit each        wait all → scan → emit

  ┌─────────────┐              ┌───┐                        ┌─────────────┐
  │███████████  │ full         │ 1 │ chunk arrives          │███████████  │
  │███████████  │ content      └─┬─┘                        │███████████  │ buffer
  └──────┬──────┘                ▼                          │███████████  │ toàn bộ
         ▼                    [scan ctx]                    └──────┬──────┘
     [scan all]                   │                                ▼
         │                        ▼                          [scan all]
         ▼                  emit (masked/as-is)                    │
   emit full result               │                                ▼
                             ┌────┴────┐                  emit 1 lần (không
                             │ 2 │ ... │                  streaming cho user)
                             └─────────┘
                             mỗi chunk:
                             scan window
                             chứa context

  Latency: 50-150ms          Latency: ~50ms/chunk          Latency: full LLM
  User experience: block     User experience: stream       time + 50-150ms
  1 lần đến khi xong         near real-time                User experience: block
                                                            toàn bộ cho đến xong
```

### 4.1 Option `input-only`

**Khi nào dùng:** Caller chỉ lo data leakage (user gửi credentials, PII, secret lên LLM). Không lo hallucination ở output.

**Flow:**

```
T=0   : Guardian nhận /v1/check/input (atomic)
T=0-X : Run scanners parallel (PII, InfoClass, ...)
T=X   : Aggregate verdict
         ├── Any revoke → return UNSAFE + block_reason
         ├── Any masking → return SAFE + sanitized_content
         ├── Only classify → return SAFE + labels
T=X+1 : Caller gửi (sanitized or original) prompt tới LLM
T=LLM : LLM response → caller forward NGUYÊN VĂN tới user (Guardian không check output)
```

**Đặc điểm:**
- Chỉ 1 Guardian call, không có post-response check
- Caller phải quyết định: gửi `original_content` hay `sanitized_content` tới LLM (tùy policy)
- Guardian luôn trả cả 2 (original + sanitized) để caller tự chọn

### 4.2 Option `output-only`

**Khi nào dùng:** Caller đã tự scan/sanitize input. Chỉ lo LLM hallucinate PII, sinh toxic content, hoặc leak từ training data.

**Flow:**

```
T=0    : User → Caller → LLM (Guardian không nhận input)
T=LLM  : LLM response
T=L+0  : Guardian nhận /v1/check/output
         - atomic nếu response full blob
         - chunk nếu response streaming SSE
T=L+X  : Run scanners → aggregate verdict → return actions
T=L+Y  : Caller apply action (truncate/mask/label) → user
```

**Đặc điểm:**
- Không block trước LLM call → chi phí compute LLM luôn phát sinh
- Latency phụ thuộc protocol (atomic ~50-150ms, chunk ~50ms/chunk)

### 4.3 Option `both` (recommended default)

**Flow tối ưu với Parallel Pattern (xem [01-architecture-design.md §2.1](./01-architecture-design.md)):**

```
T=0    : Caller nhận request
T=1    : Fork 2 tasks song song:
         ├── Task A: send to LLM (500ms-5s)
         └── Task B: Guardian input check (~100ms)
T=100  : Guardian input verdict
         ├── revoke → cancel Task A, return block
         ├── masking/classify → replace prompt trong Task A nếu cần, chờ LLM
T=LLM  : LLM response → Guardian output check (atomic or chunk)
T=L+Y  : Apply output actions → user
```

**User-perceived latency overhead:** ~0ms cho input check (ẩn sau LLM), ~50-150ms cho output check.

---

## 5. Flow theo Transport Protocol

### 5.1 Protocol: atomic (non-chunk)

**Applicable for:**
- **Input check (luôn atomic)** — user gửi full prompt
- **Output check khi LLM response không streaming**
- **Output check khi scan type buộc full content** (jailbreak, document-level classification, long-context injection)

**Flow:**

```
Caller → Guardian
  Body: { content: "<full text>", guards: ["pii", "infocls", ...] }

Guardian:
  Stage 1 — Detection (parallel asyncio.gather):
    ├── PII scanner → entities + score
    ├── InfoClass scanner → level + confidence
    └── [optional] Jailbreak scanner → is_jailbreak + score

  Stage 2 — Transformation (serial, if needed):
    if PII entities detected AND policy == masking:
      sanitized = apply_mask(content, entities)

  Stage 3 — Aggregate per policy:
    verdict = aggregate(scanner_results, policy)
    return { verdict, actions, sanitized_content?, labels? }
```

**Latency:** `max(scanner_latency) + masking_time ≈ 50-150ms` cho 2 scanners song song.

**Atomic timeline (minh hoạ):**

```
T=0ms     T=50ms                    T=150ms              T=151ms
  │          │                         │                    │
  ▼          ▼                         ▼                    ▼
┌───┐  ┌─────────────┐         ┌──────────────┐       ┌─────────┐
│REQ│─►│ PII scanner │────┐    │  Aggregate & │   ───►│ Response│
└───┘  │ (parallel)  │    ├───►│  transform   │       │ to user │
       └─────────────┘    │    └──────────────┘       └─────────┘
       ┌─────────────┐    │
       │ InfoClass   │────┘
       │ (parallel)  │
       └─────────────┘
       ◄─ 50-100ms ──►     ◄─ 10-50ms ─►
         scan phase         masking
                            + aggregate

User sees nothing until T=151ms. Acceptable vì content ngắn (prompt).
```

### 5.2 Protocol: chunk (streaming)

**Core concept — Sliding Window with Context Accumulation:**

Mỗi chunk đến KHÔNG được scan độc lập. Guardian duy trì **stateful session** per `request_id` chứa các chunk trước đó và scan chunk mới **trong ngữ cảnh** của N chunks trước nó.

**Session state (stored in Redis per request_id):**

```python
class StreamSession:
    request_id: str
    chunks: list[Chunk]              # tất cả chunks đã nhận
    window_size: int                 # max chunks/tokens trong 1 lần scan
    context_size: int                # số chunks trước dùng làm context
    emitted_positions: list[int]     # đã release cho user tới đâu
    scanner_state: dict              # running detection state (partial entities...)
    policy: PolicyConfig
```

**Flow cho mỗi chunk đến:**

```
┌─────────────────────────────────────────────────────────────────┐
│  Chunk N arrives                                                 │
│                                                                  │
│  1. Append to session.chunks                                     │
│                                                                  │
│  2. Build scan window:                                           │
│     - Start = max(0, N - context_size)                           │
│     - End   = N (current chunk inclusive)                        │
│     - Scanning content = concat(chunks[Start..End])              │
│                                                                  │
│  3. Enforce sliding window max:                                  │
│     if (End - Start) > window_size:                              │
│       Start = End - window_size                                  │
│     # Không scan quá window_size chunks/tokens dù history dài    │
│                                                                  │
│  4. Run scanners on window:                                      │
│     scan_result = await run_scanners(window_content)             │
│                                                                  │
│  5. Localize verdict cho chunk N:                                │
│     # Scanner trả entities ở position toàn window                │
│     # Chỉ phần overlap với chunk N là actionable cho N           │
│     chunk_N_verdict = intersect(scan_result.entities,            │
│                                 chunk_N.position_range)          │
│                                                                  │
│  6. Apply action cho chunk N:                                    │
│     if revoke:                                                   │
│       drop chunk N (hoặc replace bằng [REDACTED])                │
│     if masking:                                                  │
│       emit masked(chunk N)                                       │
│     if classify/safe:                                            │
│       emit chunk N as-is (+ classification in SSE metadata)      │
│                                                                  │
│  7. Update session.emitted_positions                             │
│                                                                  │
│  8. Retro-scan flag (optional):                                  │
│     if entity detected spanning chunks [N-1..N]:                 │
│       - chunk N-1 đã emit rồi → chỉ có thể gửi "retract"         │
│         notice (tùy policy UX)                                   │
│       - log alert + audit                                        │
└─────────────────────────────────────────────────────────────────┘
```

**Sliding window mechanism — minh họa:**

```
window_size = 4 chunks
context_size = 2 chunks (gồm chunk hiện tại → scan current + 1 previous)

Timeline:
  chunk 1 arrives → scan window = [1]                    # cold start
  chunk 2 arrives → scan window = [1, 2]                 # tích lũy context
  chunk 3 arrives → scan window = [2, 3]                 # context_size=2
  chunk 4 arrives → scan window = [3, 4]
  chunk 5 arrives → scan window = [4, 5]
  ...
  (chunks 1, 2 vẫn còn trong session để audit/rescan
   nhưng không tham gia scan window mặc định)

Nếu entity bắc cầu xuất hiện (chunk 3-4):
  chunk 4's scan nhìn thấy [3, 4] → phát hiện entity
  → mask phần thuộc chunk 4
  → chunk 3 đã emit → log retroactive detection
```

**Tại sao cần cả `window_size` VÀ `context_size`:**

| Param | Purpose |
|---|---|
| `context_size` | Số chunks đứng trước current chunk được lấy làm ngữ cảnh mỗi lần scan. Giữ nhỏ (1-3) để tiết kiệm compute. |
| `window_size` | Giới hạn **tuyệt đối** tổng chunks/tokens 1 lần scan. Phòng khi user/LLM spam chunks quá nhanh, session bloat, scan cost nổ. |

**Default config (dựa trên research):**
- `chunk_size_tokens`: 256 tokens / chunk
- `context_size`: 2 chunks (bao gồm chunk hiện tại) ≈ 512 tokens
- `window_size`: 4 chunks ≈ 1024 tokens (hard ceiling)
- `flush_trigger`: sentence boundary (`.`, `!`, `?`, `\n\n`) hoặc 256 tokens

**Sliding window timeline — visual:**

```
context_size=2, window_size=4
Session chunks buffer: [C1][C2][C3][C4][C5][C6][C7]...

T1: C1 arrives
    ┌────┐
    │ C1 │                    scan window = [C1]
    └────┘                    (cold start, < context_size)

T2: C2 arrives
    ┌────┬────┐
    │ C1 │ C2 │               scan window = [C1, C2]
    └────┴────┘               (context_size=2 đạt)

T3: C3 arrives
    ┌────┬────┬────┐
    │ C1 │ C2 │ C3 │          scan window = [C2, C3]
    └────┴════╧════┘          (slide forward, chỉ 2 chunks mới nhất)
         ◄────►
         context

T4: C4 arrives
    ┌────┬────┬────┬────┐
    │ C1 │ C2 │ C3 │ C4 │     scan window = [C3, C4]
    └────┴────┴════╧════┘
              ◄────►

T5: C5 arrives
    ┌────┬────┬────┬────┬────┐
    │ C1 │ C2 │ C3 │ C4 │ C5 │ scan window = [C4, C5]
    └────┴────┴────┴════╧════┘ window_size=4 — vẫn trong giới hạn

T6: C6 arrives
    ┌────┬────┬────┬────┬────┬────┐
    │ C1 │ C2 │ C3 │ C4 │ C5 │ C6 │ scan window vẫn [C5, C6]
    └────┴────┴────┴────┴════╧════┘ (context_size quyết định)
                              
    Nhưng nếu context_size bị cấu hình = 5:
    scan window sẽ bị clip bởi window_size=4
    → [C3, C4, C5, C6] (4 chunks hard cap)
    
    Chunks C1, C2 vẫn được giữ trong session cho
    audit/retro-scan, KHÔNG tham gia scan window.
```

**Entity bắc cầu — visual:**

```
LLM streams:  "Email me at john.doe" │ "@company.com please"
              ├────────── C5 ────────┤├────── C6 ──────────┤

Tại T5 (C5 arrives):
  scan window [C4, C5] chứa "...Email me at john.doe"
  scanner: không match full email (thiếu @domain.tld)
  verdict C5: SAFE → emit "Email me at john.doe" ← (!! đã lộ 1 phần)

Tại T6 (C6 arrives):
  scan window [C5, C6] chứa "Email me at john.doe@company.com please"
  scanner: MATCH EMAIL entity span [13..34]
  
  Localize cho C6:
    - C5 range: [0..20] (đã emit)
    - C6 range: [20..40]
    - Entity overlap với C6: [20..34]
  
  Action cho C6:
    emit "[EMAIL_TAIL] please"  ← mask phần thuộc C6
  
  Retro alert:
    log: entity span [13..34] partially leaked through C5
    SSE event: { type: "retroactive_flag", chunk_id: 5 }
```

**Alternative: delay-until-safe mode (buffer + delay)**

```
T5: C5 arrives → KHÔNG emit ngay, buffer
    ┌────────────┐
    │ C5 pending │
    └────────────┘

T6: C6 arrives → buffer + scan [C5, C6]
    entity detected spanning [C5, C6]
    mask toàn bộ entity, release cả 2 chunks cùng lúc
    
    emit: "Email me at [EMAIL] please"  ← trọn vẹn, không lộ

Cost: latency = thời gian chờ C6 (~50ms) thay vì stream C5 ngay.
Tradeoff: UX hơi chậm vs an toàn hơn.
```

### 5.3 Edge case quan trọng cho chunk protocol

#### 5.3.1 Entity bị cắt giữa 2 chunks

**Ví dụ:** Email `john.doe@company.com` bị LLM stream làm 2 chunks: `"...email is john.doe@com"` + `"pany.com..."`.

**Xử lý:**
- `context_size ≥ 2` đảm bảo khi scan chunk 2, window chứa cả chunk 1 → regex/NER nhận ra full email.
- Entity position nằm ở cả chunk 1 (đã emit) và chunk 2 (chưa emit).
- Quyết định:
  - Chunk 2: mask phần thuộc chunk 2 (`"[EMAIL_SUFFIX]"`)
  - Chunk 1: đã gửi → không recall được; log alert + audit
  - **Alternative (delay-until-safe mode):** buffer chunk 1 chưa emit, chờ chunk 2, mask trọn vẹn, rồi emit cùng lúc. Latency cao hơn nhưng an toàn.

#### 5.3.2 Phát hiện UNSAFE sau khi đã emit một phần

**Scenario:** Chunks 1-3 emit (SAFE). Chunk 4 chứa secret rõ ràng → revoke.

**Options (configurable per policy):**

| Mode | Behavior |
|---|---|
| `truncate-notice` | Drop chunk 4+, gửi SSE event `{ "type": "truncated", "reason": "policy_violation" }`. User thấy phản hồi bị cắt. |
| `replace-notice` | Replace từ chunk 4 bằng template `"[Response withheld due to policy]"`. |
| `silent-stop` | Chỉ đóng stream, không giải thích. UX kém, chỉ cho edge case. |

**Recommendation:** default `truncate-notice`.

#### 5.3.3 Policy yêu cầu full content (chunk không đủ)

Một số scanner (jailbreak detection, document-level classification) cần toàn bộ response để scan chính xác. Không thể chunk.

**Hybrid approach:**
1. **Streaming layer (online):** chunk scan với PII, InfoClass → emit tokens cho user real-time.
2. **Post-stream layer (offline, async):** khi stream kết thúc, Guardian run full-content scan (jailbreak, long-context injection).
3. Kết quả post-scan:
   - Log + alert → Security team review
   - Không thể recall tokens đã emit; dùng cho audit + future prompt rejection
   - Nếu confidence rất cao → có thể gửi SSE event `{ "type": "post_scan_violation" }` để UI hiển thị cảnh báo

### 5.4 Protocol: atomic CƯỠNG CHẾ dù LLM streaming

**Khi nào:** Policy của caller yêu cầu `full_content_required: true` cho loại scan nào đó.

**Flow:**

```
LLM streams response → Guardian buffers TOÀN BỘ chunks
  ↓
Đợi LLM kết thúc stream (EOF)
  ↓
Run atomic scan trên full buffered content
  ↓
  ├── revoke → không release gì cho user, trả block message
  ├── masking → release full masked content 1 lần
  └── classify → release full content + label
```

**Latency cost:** User không thấy streaming. Perceived latency = `LLM_stream_total_time + scan_time`. Phải chấp nhận.

**Use case điển hình:**
- Compliance service: mọi output phải qua jailbreak + hierarchical classification full-scan trước khi user thấy.
- Short-response service: response ngắn (<512 tokens), chờ full không đáng kể.

**Atomic-forced timeline vs chunk streaming — visual:**

```
CHUNK STREAMING (default)
═════════════════════════
T=0     T=200ms          T=500ms          T=2000ms         T=2100ms
 │        │                │                 │                │
 ▼        ▼                ▼                 ▼                ▼
REQ    C1 emit            C3 emit          C8 emit          C10 emit + EOF
       (user sees)        (user sees)      (user sees)       (done)
       ├──► scan ─► ok    ├──► scan ─► ok   ├──► scan ─► ok
       
User experience: thấy content xuất hiện dần dần theo thời gian thực.


ATOMIC-FORCED (high-compliance)
═══════════════════════════════
T=0                              T=2000ms                T=2100ms
 │                                 │                        │
 ▼                                 ▼                        ▼
REQ ─────── LLM streaming ──────► EOF ──► full-scan ──► emit ALL
                                          (~100ms)         (1 lần)

User experience: thấy "loading..." 2.1 giây, rồi toàn bộ response hiện ra.

Tradeoff: +5-10% total latency đổi lấy 100% content được scan kèm
đủ ngữ cảnh (jailbreak, hierarchical classification).
```

---

## 6. Action Pattern — Chi tiết hành vi

### 6.1 Masking

**Input/Output contract:**
```json
{
  "verdict": "SAFE_WITH_MASKING",
  "action": "masking",
  "sanitized_content": "My name is [PERSON] and email is [EMAIL]",
  "original_content": "<optional, chỉ trả nếu policy cho phép>",
  "entities": [
    { "type": "PERSON", "start": 11, "end": 18, "placeholder": "[PERSON]" },
    { "type": "EMAIL", "start": 32, "end": 51, "placeholder": "[EMAIL]" }
  ]
}
```

**Subpatterns:**

| Variant | Mô tả | Use case |
|---|---|---|
| **redact** | Thay bằng placeholder cố định `[REDACTED]`, `[EMAIL]` | Default, đơn giản |
| **tokenize** | Thay bằng token duy nhất `<TOK_a7f3>`, lưu mapping trong vault Redis | Khi cần de-tokenize sau (ví dụ downstream service cần data gốc) |
| **fuzz** | Thay bằng synthetic data gần giống (`[FAKE_EMAIL]` = `user-123@example.com`) | Khi LLM cần format hợp lệ để xử lý tiếp |

**Policy config:**
```yaml
masking:
  strategy: redact        # redact | tokenize | fuzz
  placeholder_style: bracket   # bracket [EMAIL] | angle <EMAIL> | unicode ■
  preserve_format: true   # giữ độ dài khoảng bằng original
  vault:
    enabled: false        # true → lưu mapping để de-tokenize
    ttl_seconds: 3600
```

### 6.2 Classify

**Input/Output contract:**
```json
{
  "verdict": "SAFE",
  "action": "classify",
  "classification": {
    "level": "internal_use_only",
    "confidence": 0.87,
    "probabilities": {
      "public": 0.02,
      "internal_use_only": 0.87,
      "restricted": 0.10,
      "secret": 0.01,
      "top_secret": 0.00
    }
  },
  "content_forwarded_as_is": true
}
```

**Đặc điểm:**
- Content **không bị sửa**, chỉ dán label vào response metadata
- Caller tự quyết định xử lý: log, alert, downstream routing, audit

**Policy config:**
```yaml
classify:
  emit_label_in_header: true    # SSE metadata event hoặc response header
  emit_label_in_body: false     # không chèn label vào content
  log_level: info
```

### 6.3 Revoke

**Input/Output contract:**
```json
{
  "verdict": "UNSAFE",
  "action": "revoke",
  "block_reason": "pii_secret_level",
  "violated_guards": ["info_classification"],
  "user_message": "This request was blocked due to policy.",
  "audit_id": "uuid-for-security-review"
}
```

**Subpatterns:**

| Variant | Mô tả |
|---|---|
| **hard-revoke** | Không release gì. Trả user_message cố định. |
| **soft-revoke (with explanation)** | Kèm hint ngắn: "Your prompt contained secret-level info. Please rephrase." |
| **shadow-revoke** | (Chỉ log mode) không block, chỉ log + alert. Dùng trong shadow deploy giai đoạn đầu. |

**Policy config:**
```yaml
revoke:
  mode: hard                # hard | soft | shadow
  user_message_template: "Request blocked ({reason})"
  include_audit_id: true    # user có thể dùng để ticket
  alert_security: true
```

### 6.4 Aggregation khi nhiều scanner trả khác pattern

Một request có thể có nhiều scanners, mỗi scanner trả action khác nhau. Aggregation rules:

```
Priority (cao → thấp):
  1. Revoke — nếu BẤT KỲ scanner revoke → final = revoke (fail-strict)
  2. Masking — nếu có masking → apply cho content, tiếp tục check classify
  3. Classify — dán labels từ tất cả scanners
```

**Ví dụ:**
- PII scanner: masking (mask email)
- InfoClass scanner: classify (level=restricted)
- Jailbreak scanner: safe
- **Final verdict:** `SAFE_WITH_MASKING`, `sanitized_content` = masked version, `classification.level` = restricted

**Ví dụ 2:**
- PII scanner: masking
- InfoClass scanner: classify (level=secret)
- **Final verdict:** `UNSAFE`, action = revoke (vì level=secret theo policy default `on_fail_by_level.secret: reject`). Masked content không được forward.

**Aggregation decision tree — visual:**

```
                    ┌──────────────────────┐
                    │ Scanner results vào  │
                    │ (N scanner × results)│
                    └──────────┬───────────┘
                               │
                               ▼
                ┌──────────────────────────────┐
                │ Bất kỳ scanner nào trả       │
                │ action = revoke?             │
                └──┬───────────────────────┬───┘
                   │ yes                   │ no
                   ▼                       ▼
          ┌────────────────┐     ┌───────────────────────┐
          │ Policy check:  │     │ Bất kỳ scanner nào    │
          │ level ≥ thresh?│     │ trả action = masking? │
          │ (ví dụ secret) │     └──┬────────────────┬───┘
          └──┬─────────┬───┘        │ yes            │ no
             │ yes     │ no         ▼                ▼
             ▼         ▼      ┌──────────────┐  ┌──────────────┐
       ┌─────────┐  ┌─────────┤ Apply mask   │  │ Keep content │
       │ FINAL:  │  │ Degrade │ to content   │  │ as-is        │
       │ REVOKE  │  │ to      └──────┬───────┘  └──────┬───────┘
       │ (block) │  │ masking        │                 │
       └─────────┘  └────┬───┘       ▼                 ▼
                         │      ┌──────────────────────────────┐
                         ▼      │ Aggregate tất cả classify    │
                   (go right)   │ labels vào response metadata │
                                └────────────┬─────────────────┘
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │ FINAL:               │
                                  │ SAFE / SAFE_WITH_    │
                                  │ MASKING + labels[]   │
                                  └──────────────────────┘

Priority order: revoke > masking > classify > safe
Fail-strict: 1 scanner revoke → toàn bộ revoke (không forward gì)
```

---

## 7. Flow Combination Matrix

Bảng tổ hợp Option × Protocol × Pattern (patterns xảy ra trong runtime, không phải config cứng):

| # | Option | Protocol | Patterns có thể xảy ra | Flow reference |
|---|---|---|---|---|
| 1 | input-only | atomic | masking, classify, revoke | §4.1, §5.1 |
| 2 | input-only | chunk | **N/A** (input luôn atomic) | - |
| 3 | output-only | atomic | masking, classify, revoke | §4.2, §5.1 |
| 4 | output-only | chunk | masking per chunk, classify per chunk, revoke (truncate) | §4.2, §5.2 |
| 5 | output-only | atomic-forced (full content policy) | masking, classify, revoke | §4.2, §5.4 |
| 6 | both | input atomic + output atomic | tất cả pattern ở cả 2 check | §4.3, §5.1 |
| 7 | both | input atomic + output chunk | masking/classify/revoke ở input; per-chunk actions ở output | §4.3, §5.2 |
| 8 | both | input atomic + output atomic-forced | atomic ở cả 2 | §4.3, §5.4 |
| 9 | both | input atomic + output hybrid (chunk + async full-scan) | streaming actions + post-stream async alert | §5.3.3 |

**Ghi chú ô #2:** Input không hỗ trợ chunk protocol vì user gửi full prompt trong 1 HTTP request. Nếu tương lai hỗ trợ voice streaming input / incremental input, sẽ reopen.

---

## 8. API Contract extensions

### 8.1 `/v1/check/input` (atomic only)

```
POST /v1/check/input
Body:
  {
    "request_id": "uuid",
    "content": "<full prompt>",
    "user_id": "string",
    "guards": ["pii", "info_classification", "jailbreak"],
    "policy_id": "default" | "high_compliance" | ...,
    "context": { ... }
  }

Response:
  {
    "request_id": "uuid",
    "verdict": "SAFE" | "SAFE_WITH_MASKING" | "UNSAFE",
    "actions": [
      { "guard": "pii", "pattern": "masking", ... },
      { "guard": "info_classification", "pattern": "classify", ... }
    ],
    "sanitized_content": "<masked>",
    "classifications": { ... },
    "block_reason": null | "<reason>",
    "total_latency_ms": 98
  }
```

### 8.2 `/v1/check/output` atomic mode

Schema tương tự `/check/input`. Field `content` = full LLM response.

### 8.3 `/v1/check/output/stream` chunk mode

**SSE hoặc WebSocket stream:**

```
Client → Server (open stream):
  POST /v1/check/output/stream
  Headers: Content-Type: text/event-stream
  Body (initial frame):
    {
      "request_id": "uuid",
      "session_config": {
        "chunk_size_tokens": 256,
        "context_size": 2,
        "window_size": 4,
        "guards": ["pii", "info_classification"],
        "policy_id": "default"
      }
    }

Client → Server (per chunk):
  event: chunk
  data: { "chunk_id": 1, "content": "<tokens>", "is_final": false }

Server → Client (per chunk response):
  event: verdict
  data: {
    "chunk_id": 1,
    "verdict": "SAFE" | "SAFE_WITH_MASKING" | "UNSAFE",
    "emitted_content": "<forward hoặc masked>",
    "actions": [...],
    "drop": false | true
  }

Server → Client (end of stream):
  event: session_summary
  data: {
    "total_chunks": 42,
    "unsafe_chunks": [7, 15],
    "retroactive_alerts": [...],
    "audit_id": "uuid"
  }
```

### 8.4 `/v1/check/output` atomic-forced (full content policy)

Dùng chung endpoint `/v1/check/output` atomic. Caller gửi full response sau khi LLM stream xong. Không phải endpoint mới.

---

## 9. Sequence Diagrams

### 9.1 Option `both`, output streaming (chunk protocol) — happy path

```
User → GenAI GW : POST /chat (stream=true)
GenAI GW → Guardian : POST /check/input (prompt)
GenAI GW → LLM : POST /completions (stream=true)   # parallel
Guardian → GenAI GW : {verdict: SAFE, actions: [classify]}
LLM -->> GenAI GW : chunk 1 "Hello, "
GenAI GW → Guardian : /check/output/stream (chunk 1)
Guardian → GenAI GW : {verdict: SAFE, emitted: "Hello, "}
GenAI GW -->> User : "Hello, "
LLM -->> GenAI GW : chunk 2 "my name is "
GenAI GW → Guardian : /check/output/stream (chunk 2)
Guardian → GenAI GW : {verdict: SAFE, emitted: "my name is "}
GenAI GW -->> User : "my name is "
LLM -->> GenAI GW : chunk 3 "john@example.com"
GenAI GW → Guardian : /check/output/stream (chunk 3)
Guardian → GenAI GW : {verdict: SAFE_WITH_MASKING, emitted: "[EMAIL]"}
GenAI GW -->> User : "[EMAIL]"
LLM -->> GenAI GW : [EOF]
GenAI GW → Guardian : /check/output (atomic-forced for jailbreak scan, async)
Guardian -->> GenAI GW : post-scan result (if violation, log only)
```

### 9.2 Option `input-only` atomic, input UNSAFE revoke

```
User → GenAI GW : POST /chat (prompt contains secret)
GenAI GW → Guardian : /check/input (prompt)
Guardian : run scanners
Guardian → GenAI GW : {verdict: UNSAFE, action: revoke, reason: "top_secret_content"}
GenAI GW → User : { "error": "policy_blocked", "message": "..." }
# LLM không được gọi
```

### 9.3 Output chunk — entity bắc cầu

```
Chunk 1 arrives: "My email is john.doe@"
  window = [chunk1]
  scan → email pattern incomplete, no entity detected
  emit chunk1 AS-IS (vì SAFE)

Chunk 2 arrives: "company.com for support"
  window = [chunk1, chunk2]   # context_size=2
  scan → detect EMAIL entity spanning chunk1+chunk2
  entity position = chars [12..32]
    chunk1 range = [0..22] (đã emit)
    chunk2 range = [22..40]
  
  Action cho chunk 2:
    - mask phần thuộc chunk2: "com for support"
      → "[EMAIL_TAIL] for support" (partial mask)
    - alternative: mask toàn chunk2 với notice
  
  Retro alert:
    - chunk 1 đã emit "john.doe@" → log audit
    - gửi SSE event: { "type": "retroactive_flag", "chunk_id": 1 }
```

### 9.4 Output atomic-forced (full-content policy)

```
User → GW : POST /chat (stream=true, policy=high_compliance)
GW → LLM : stream=true
LLM -->> GW : chunks 1..N
GW : buffer all chunks (không forward cho user)
LLM -->> GW : [EOF]
GW → Guardian : /check/output (full buffered content)
Guardian : atomic scan (PII + InfoClass + Jailbreak)
  → verdict SAFE_WITH_MASKING
Guardian → GW : { sanitized_content: "<full masked>" }
GW -->> User : "<full masked>" (1 lần, không streaming)
```

---

## 10. Configuration Template

```yaml
# policies.yaml (per consumer / API key)
policies:
  default:
    input_check:
      enabled: true
      protocol: atomic
      guards: [pii, info_classification]
      on_unsafe: revoke
    output_check:
      enabled: true
      protocol: chunk          # atomic | chunk | atomic_forced
      guards: [pii, info_classification]
      chunk_config:
        chunk_size_tokens: 256
        context_size: 2
        window_size: 4
        flush_trigger: sentence
      post_stream_async:
        enabled: true
        guards: [jailbreak]
      on_unsafe: truncate_notice
  
  high_compliance:
    input_check:
      enabled: true
      protocol: atomic
      guards: [pii, info_classification, jailbreak]
      on_unsafe: revoke
    output_check:
      enabled: true
      protocol: atomic_forced  # buộc full-content scan, accept high latency
      guards: [pii, info_classification, jailbreak]
      on_unsafe: revoke
  
  input_only_service:
    input_check:
      enabled: true
      protocol: atomic
      guards: [pii]
      on_unsafe: mask_and_forward
    output_check:
      enabled: false
  
  output_only_monitor:
    input_check:
      enabled: false
    output_check:
      enabled: true
      protocol: chunk
      guards: [pii]
      on_unsafe: mask
```

---

## 11. Performance & Resource Implications

| Protocol | Guardian calls/request | State tracking | Cost profile |
|---|---|---|---|
| atomic input | 1 | none | ~1x vLLM inference |
| atomic output | 1 | none | ~1x |
| atomic-forced output | 1 (sau khi full response) | none | ~1x, latency cao |
| chunk output | ~N calls (N = #chunks, ~20-40 per 1K tokens) | Redis session ~1KB/request | N × scanner_cost, cache hit rate cao (context overlap) |

**Optimization cho chunk mode:**
- Cache key: hash(window_content) → reuse nếu chunks sớm giống nhau
- Batch scan: gom 2-3 chunks pending nếu arrival rate > throughput
- Skip-rescan: nếu context không đổi (chỉ thêm chunk mới), chỉ scan delta

---

## 12. Risk & Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| Chunk boundary cắt entity, miss detection | Medium (PII leak qua chunk) | `context_size ≥ 2`, sentence-boundary flush |
| Retro detection nhưng chunk đã emit | High (PII đã gửi user) | Delay-until-safe mode cho high-compliance; audit log ngay |
| Session state bloat khi stream dài | Medium (Redis cost) | `window_size` hard cap; TTL session 5 phút |
| Chunk scan cost cao hơn atomic | Medium (vLLM load) | Cache theo window hash; skip-rescan optimization |
| Policy quá strict → false positive revoke | High (UX) | Shadow mode 2 tuần, threshold tuning |
| Atomic-forced tăng latency không chấp nhận được | Medium | Chỉ áp cho `high_compliance` policy, không phải default |

---

## 13. Câu hỏi còn mở

1. **De-tokenize vault governance:** Khi dùng masking `tokenize` — ai có quyền de-tokenize? Audit log cho de-tokenize event? (GDPR)
2. **Retro detection UX:** Khi phát hiện UNSAFE sau khi đã emit — gửi SSE `retroactive_flag` event hay silent log? Stakeholder (product) cần quyết định.
3. **Post-stream async scan cadence:** Chạy cho mọi response hay sample (5-10%)? Cost vs coverage tradeoff.
4. **Chunk session state storage:** Redis per session có scale được cho 340 req/sec với session 5 phút TTL? Cần benchmark.
5. **Input streaming support (future):** Voice input, incremental UI input có cần chunk protocol cho input không? Scope Phase 2+.
6. **Classify không block — caller có buộc phải tôn trọng label không?** Hay Guardian trust caller policy? Ảnh hưởng đến accountability.
7. **Atomic-forced + chunk combo:** Output có thể vừa stream (chunk) vừa có atomic-forced post-scan không? (§9.1 hybrid). Nếu atomic-forced revoke sau khi đã stream → xử lý sao?

---

## Sources

- [Research report: streaming guardrail patterns](../plans/reports/researcher-260416-2244-streaming-guardrail-patterns.md)
- [01-architecture-design.md](./01-architecture-design.md) — §2.1 Parallel Pattern, §2.2 Streaming output
- [04-top-5-adopt-patterns-explained.md](./04-top-5-adopt-patterns-explained.md) — Pattern 3, Pattern 5
- NVIDIA NeMo streaming docs — https://developer.nvidia.com/blog/stream-smarter-and-safer-learn-how-nvidia-nemo-guardrails-enhance-llm-output-streaming/
- AWS Bedrock ApplyGuardrail streaming — https://aws.amazon.com/blogs/machine-learning/use-the-applyguardrail-api-with-long-context-inputs-and-streaming-outputs-in-amazon-bedrock/
- Guardrails AI streaming — https://www.guardrailsai.com/docs/how_to_guides/enable_streaming
- Early Stopping LLM Harmful Outputs — https://arxiv.org/html/2506.09996v1
