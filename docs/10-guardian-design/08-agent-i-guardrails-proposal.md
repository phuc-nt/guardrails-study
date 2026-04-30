# Guardian — Agent i Guardrails Proposal

**Version:** 0.1 | **Date:** 2026-04-30
**Scope:** Đề xuất bộ guardrail products / modules cụ thể cho Agent i (アイ — LINE × Yahoo! かんたんAI), dựa trên integration map vào hệ sinh thái LY và risk landscape theo từng domain.
**Audience:** PO Agent i, Tech Lead Guardian, Legal & Compliance LY Corp

> Nguồn research đầy đủ: [`plans/reports/researcher-260430-1735-agent-i-line-yahoo-integration.md`](../../plans/reports/researcher-260430-1735-agent-i-line-yahoo-integration.md)
> Bổ sung cho: [01-architecture-design.md](./01-architecture-design.md), [06-dual-mode-proposal.md](./06-dual-mode-proposal.md), [07-architectural-patterns-proposal.md](./07-architectural-patterns-proposal.md)

---

## 1. Bối cảnh Agent i (tóm tắt)

Agent i = brand AI thống nhất của LY Corp (launch 2026-04-20), tagline "毎日のそばに、だれでも使えるAIを" — embed vào **>100 service** của LINE và Yahoo! JAPAN, scale 100M+ MAU.

**7 domain agent (active hoặc β):** Shopping, Travel, Weather, Automotive (β), Relationships (β), Work (β), Recipes (β).

**Roadmap quan trọng:**
- **Jun 2026** — Memory + autonomous task execution (purchase, booking, payment)
- **Aug 2026** — Agent i Biz (B2B) + LINE OA AI Mode (merchant agent)
- **Q3+ 2026** — multi-step agentic workflow, có thể có 3rd-party skill marketplace

**Tech stack inferred:** OpenAI APIs (SoftBank/OpenAI JV) + Kong API GW + 100+ LY service API + likely vector DB cho memory.

**Đặc điểm khiến guardrail Agent i khác Guardian M1 cho GenAI GW:**
- B2C consumer → false-positive cao = UX hỏng → ưu tiên fail-open + retroactive
- Tool-calling autonomous (LINE Pay, booking) → guardrail KHÔNG chỉ check text mà phải check **action**
- Multi-domain → 1 user request có thể chạm finance + health + commerce trong 1 turn
- Regulatory đa cơ quan (FSA, MHLW, PPC, METI) → mỗi domain có rule riêng

---

## 2. Surface map: Agent i × LY service × risk

| Service | Agent i capability | Risk dominance | Guardrail module cần |
|---|---|---|---|
| LINE Messenger (chat) | Primary entry, memory | PII leak, chat history | M1 PII, M2 Memory governance |
| LINE Pay | Autonomous payment (Jun 2026) | Fraud, PSA, unauth txn | M3 Payment authz |
| LINE Doctor | Symptom routing | 薬機法, misdiagnosis | M4 Medical |
| LINE Manga / Music | Recommendation, summary | 著作権法 | M5 Copyright |
| LINE OA AI Mode | Merchant custom agent | Rogue merchant, customer data leak | M6 Merchant agent vetting |
| Yahoo! ショッピング | Shopping agent, compare | 特定商取引法, affiliate bias | M7 Commerce fairness |
| Yahoo! ファイナンス | Stock sentiment | 金融商品取引法 (FIEA) | M8 Financial advice |
| Yahoo! ニュース | News summarization | Bias, election interference | M9 News & misinfo |
| Yahoo! 知恵袋 | Q&A synthesis | Medical/legal bad advice | M4 + M10 |
| Yahoo! 天気 / 路線 | Location-aware | Location privacy | M1 PII |
| Yahoo! トラベル | Booking | 特定商取引法, double-book | M7 |
| Yahoo! 不動産 / オークション | Marketplace | Fair housing, fraud | M7 |

→ **10 module guardrail** đề xuất (M1–M10), gọi tắt là **Agent i Guardrail Suite**.

---

## 3. 10 guardrail product / module đề xuất

### M1. PII Shield (APPI compliance) — **P0, M1**

**Mục đích:** chặn PII leak qua chat output + audit cross-service PII flow.

- **Detect:** Japanese-aware NER (氏名, 住所, 電話番号, マイナンバー, クレジットカード番号, パスポート番号, 銀行口座) + cross-service joined profile detection (vd output đề cập "anh A vừa khám LINE Doctor và mua thuốc trên Yahoo Shopping" → cross-domain leak)
- **Action:** mask / redact / block + log
- **Output mode:** chunk streaming với sliding window (fast mode)
- **Compliance hook:** purpose-limitation tag — mỗi request gắn `intended_use`, từ chối nếu output vượt purpose
- **Retention:** memory feature (Jun 2026) phải gắn TTL + right-to-erasure API

### M2. Memory Governance — **P0, Jun 2026 (cùng lúc memory feature launch)**

**Mục đích:** kiểm soát feature "memory" (long-term user preference store) tuân thủ APPI right-to-erasure + purpose limitation.

- **Consent ledger:** mỗi domain phải có opt-in riêng (memory cho shopping ≠ memory cho health)
- **Selective recall:** scanner check memory snippet trước khi inject vào prompt — chặn nếu snippet thuộc domain khác purpose hiện tại (vd user hỏi shopping mà inject memory health → block)
- **Erasure API:** xoá vector + cascade xoá ở backup
- **Audit:** mỗi memory write/read log lại ai (domain agent nào) ghi/đọc khi nào

### M3. Payment Authorization Guardrail — **P0, Jun 2026 (autonomous payment launch)**

**Mục đích:** ngăn unauthorized / fraudulent autonomous transaction qua LINE Pay.

- **Rate limit:** max N giao dịch / ngày / user qua agent
- **Amount ceiling:** giao dịch >100K JPY → bắt OTP + biometric
- **Anomaly score:** ML model phát hiện merchant lạ, time lạ, amount lạ, geo lạ → human review
- **Intent confirmation:** Agent KHÔNG được tự suy diễn "user muốn mua" — phải có explicit confirm phrase (vd "はい、購入してください")
- **Chargeback log:** full audit trail attach vào LINE Pay dispute system
- **Pattern reuse:** kết hợp **MCP-layer guardrail** (xem [07-architectural-patterns-proposal.md](./07-architectural-patterns-proposal.md) §2 pattern 8) cho tool-call authz

### M4. Medical & Pharma Guardrail (薬機法) — **P0, M1 nếu LINE Doctor enable**

**Mục đích:** ngăn Agent i tự diagnose, prescribe, recommend OTC drug — vi phạm 薬機法.

- **Hard block list:** từ vựng diagnosis ("〜病です", "〜と診断"), drug recommendation ("〜を飲んでください"), dosage advice
- **Whitelist source:** chỉ cite MHLW guideline, peer-review, official drug info — không cite blog/Q&A
- **Auto-route:** mọi health query → "医師にご相談ください" + link LINE Doctor / kokushou number
- **Sensitive topic immediate-escalate:** mental health (suicide ideation), pregnancy, addiction → bypass LLM, route trực tiếp human / hotline
- **Disclaimer injection:** mọi health response auto-append disclaimer

### M5. Copyright & IP Guardrail (著作権法) — **P1, M2**

**Mục đích:** tránh sinh content vi phạm bản quyền (manga summary, lyrics, news verbatim).

- **Verbatim detector:** check output similarity với LINE Manga / LINE Music corpus + Yahoo News headlines → nếu >X% overlap, rewrite hoặc cite
- **Attribution forcing:** mọi recommendation gắn creator/source link
- **Generation block:** chặn yêu cầu "vẽ cover manga theo style 〜", "viết lyrics giống bài 〜"
- **Whitelist licensed content:** chỉ summarize content có license clearance trong LINE Manga / Music partnership database

### M6. Merchant Agent Vetting (LINE OA AI Mode) — **P0, Aug 2026**

**Mục đích:** vet custom agent do merchant tạo (LINE OA AI Mode launch Aug 2026) — tránh rogue merchant abuse customer.

- **Pre-deploy scan:** PromptFoo + Garak chạy trên system prompt + tool config của merchant — xem [07 §2 pattern 5](./07-architectural-patterns-proposal.md)
- **Runtime sandbox:** merchant agent không được call API ngoài whitelist (LINE Pay merchant scope only, không touch customer profile rộng hơn)
- **Customer data minimization:** chỉ pass minimal context cho merchant agent (no shopping history cross-merchant)
- **Discrimination check:** auto-test merchant agent với synthetic customer demographic — flag nếu reject pattern lệch demographic
- **Audit transparency:** merchant phải khai báo mục đích, customer thấy badge "AI agent của 〜"

### M7. Commerce Fairness Guardrail (特定商取引法 + 不正競争防止法) — **P1, M2**

**Mục đích:** tránh affiliate bias, fake review amplification, misleading recommendation trong shopping/travel/realestate/auction.

- **Affiliate disclosure:** nếu link kèm commission, output phải có "PR" / "アフィリエイト" tag (luật Nhật bắt buộc)
- **Price sanity check:** so output price với market median (Rakuten, Amazon JP) — flag nếu >20% deviation
- **Review authenticity:** lọc fake review trước khi aggregate (collab với existing Yahoo/LINE fraud detection)
- **Conflict-of-interest disclosure:** nếu recommend Yahoo!ショッピング trong khi user query trung lập, phải show alternative (Rakuten, Amazon)
- **Purchase confirmation:** explicit confirm trước transaction (overlap M3)

### M8. Financial Advice Guardrail (金融商品取引法 / FIEA) — **P0, M1 nếu Finance domain enabled**

**Mục đích:** tránh unregistered investment advice.

- **Hard block:** directional recommendation ("買うべき", "売るべき", "上がる", "下がる") trong context cá nhân
- **Sentiment is informational only:** sentiment score được phép, kèm label "情報提供のみ・投資助言ではありません"
- **Data freshness:** stock data >15 phút cũ → block (tránh stale data lừa user)
- **No insider info:** scanner detect institutional data leak (earnings preview, analyst report private)
- **Disclaimer injection:** "本回答は投資助言ではありません" force-append
- **Restriction:** Finance domain chỉ chạy ở **deep mode** (atomic-forced) — đảm bảo full scan trước khi user thấy

### M9. News & Misinformation Guardrail — **P1, M2**

**Mục đích:** ngăn news summarization bias, election interference, misinformation amplification.

- **Source citation enforcement:** mọi claim factual link đến Yahoo!ニュース article cụ thể (không tự suy)
- **Election-period mode:** từ ngày公示 đến開票, news summarization chuyển sang strict mode (chỉ đọc-tóm chính thức, không infer)
- **Multi-source diversity:** tránh chỉ dùng 1 outlet — auto-include perspective khác nếu detected political
- **Fact-check integration:** plug Yahoo factcheck partner (Japan Fact-check Center)
- **User-disable option:** consumer có thể tắt news summarization, quay về raw article

### M10. Child Safety Guardrail — **P0, M1**

**Mục đích:** LINE có user từ 6+ tuổi, Agent i không có age gate riêng → cần child safety layer.

- **Age inference:** account-bound age signal (LINE registration age) + behavioral signal
- **Content filter for minors:** sex/violence/gambling/finance/medical sensitive auto-blocked nếu age <13
- **Memory consent:** parental consent bắt buộc cho memory feature với <18
- **Ad targeting block:** không personalize ad cho <13 (COPPA-like, dù APPI không bắt buộc rõ)
- **Self-harm immediate:** child mention self-harm → bypass LLM, route đến よりそいホットライン

---

## 4. Pattern foundation cho Suite

Suite này không build từ đầu — leverage các pattern đã đề xuất ở [07-architectural-patterns-proposal.md](./07-architectural-patterns-proposal.md):

| Module | Pattern foundation chính |
|---|---|
| M1 PII | Guardian API server (M1) + SDK embed cho mobile edge (Q3) |
| M2 Memory | Guardian API + Observability retroactive (Langfuse) |
| M3 Payment | **MCP-layer guardrail** (pattern 8) cho tool-call authz |
| M4 Medical | Guardian API server, deep mode, T2 medical NER classifier |
| M5 Copyright | Pre-deploy scanner (corpus check) + runtime detector |
| M6 Merchant | **Pre-deploy scanner** (pattern 5) for vet + runtime sandbox |
| M7 Commerce | Guardian API server, fast mode + post-stream async audit |
| M8 Financial | Guardian API server, deep mode (atomic-forced) bắt buộc |
| M9 News | Guardian API server + observability cho election mode |
| M10 Child | Guardian API + per-tenant policy override theo age tier |

→ Confirm là **Guardian API server (M1) là backbone**, không phải nhóm 10 service riêng. Mỗi module = config + classifier + rule trong Guardian, không phải service tách rời.

---

## 5. Roadmap đề xuất

| Phase | Module ưu tiên | Lý do |
|---|---|---|
| **M1 (Apr–May 2026)** | M1 PII, M4 Medical, M8 Financial, M10 Child | Risk pháp lý cao nhất, blocker cho launch domain liên quan |
| **M2 (Jun 2026)** | M2 Memory, M3 Payment | Cùng lúc với memory + autonomous payment launch |
| **M3 (Aug 2026)** | M6 Merchant agent vetting | Blocker cho LINE OA AI Mode launch |
| **M4 (Q3 2026)** | M5 Copyright, M7 Commerce, M9 News | Reduce regulator + reputation risk, không block launch |
| **Continuous** | All — Langfuse retroactive jailbreak scan, A/B test policy | Adapt theo attack pattern mới |

---

## 6. SLA & UX guard cho B2C

Khác với GenAI GW B2E (chấp nhận latency cho compliance), Agent i là consumer:

| Constraint | Target |
|---|---|
| Guardrail latency (fast mode default) | p95 ≤ 80ms, p99 ≤ 120ms |
| False-positive rate (block correct content) | <0.5% |
| Hard-block latency (M4, M8, M10 critical) | ≤ 50ms (rule-based, T1 only) |
| Async audit lag (M2, M5, M9 retroactive) | <5 phút phát hiện, <1h escalate |

Strict false-positive budget vì 0.5% × 100M MAU = 500K complaint/ngày.

---

## 7. Open questions

1. **Domain agent boundary** — Agent i có expose 7 domain rõ ràng (như 7 agent riêng) hay 1 router? Ảnh hưởng module M apply ở layer nào.
2. **Memory architecture** — Vector DB nào (Pinecone, Weaviate, Yahoo proprietary)? Erasure mechanism ra sao? M2 design phụ thuộc.
3. **MCP usage** — Agent i có dùng Model Context Protocol cho tool-calling không, hay function-calling OpenAI native? Ảnh hưởng M3 (payment authz).
4. **3rd-party skill marketplace** — có roadmap cho phép developer ngoài LY upload skill không? Nếu có, cần module M11 (3rd-party skill vetting) tương tự M6.
5. **LINE Doctor scope** — Agent i route đến LINE Doctor hay tự trả lời basic? Quyết định độ chặt M4.
6. **Election-mode trigger** — Ai/cơ chế nào trigger M9 election mode (manual policy push, auto theo lịch 公示)?
7. **Age inference reliability** — LINE registration age đáng tin bao nhiêu (kid dùng account của parent)? M10 có cần behavioral signal bổ sung?
8. **Cross-domain memory leak test** — có red-team plan thử leak memory shopping vào health context không?
9. **Multi-language** — Agent i có serve user JP/KR/TW/TH? PII NER (M1) phải multilingual ra sao?
10. **Liability allocation** — nếu M8 fail và user lỗ tiền, LY chịu hay disclaimer đủ? Cần legal sign-off trước khi enable Finance domain.
