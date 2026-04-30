# Guardian — Beyond API Server: Complementary Pattern Proposal

**Version:** 0.1 | **Date:** 2026-04-30
**Scope:** Đề xuất các pattern guardrail KHÁC API server đáng adopt vào platform để bổ sung cho Guardian M1.
**Audience:** Tech Lead, Platform Architect, PO Agent i / GenAI GW

> Nguồn research đầy đủ: [`plans/reports/researcher-260430-1725-non-api-server-guardrail-patterns.md`](../../plans/reports/researcher-260430-1725-non-api-server-guardrail-patterns.md)
> Bổ sung cho: [01-architecture-design.md](./01-architecture-design.md), [06-dual-mode-proposal.md](./06-dual-mode-proposal.md)

---

## 1. Vì sao cần xét pattern khác

API server (Guardian M1) tối ưu cho **central policy enforcement** ở B2E (GenAI GW). Nhưng platform dài hạn phục vụ nhiều consumer với constraint khác nhau:

- **Agent i mobile / edge** — không muốn round-trip đến Guardian, cần latency <30ms
- **GenAI GW ingress** — muốn intercept ở proxy layer, không sửa app code
- **Agent i tool-use** — guardrail cho MCP / function-calling, không phải prompt I/O
- **CI/CD pipeline** — chặn prompt template / RAG corpus xấu **trước khi** deploy
- **Audit / compliance** — cần log mọi trace + retroactive scan, không chỉ block realtime

Một pattern đơn lẻ không cover hết. Platform cần **portfolio of patterns**.

---

## 2. 8 pattern đã đánh giá

| # | Pattern | Verdict | Khi nào adopt |
|---|---|---|---|
| 1 | **SDK / library embed in-process** | ✅ Q3 2026 | Agent i edge / offline, latency <30ms |
| 2 | **Sidecar / Proxy** (LiteLLM, Portkey, Envoy filter) | ✅ M2 (eval) | GenAI GW ingress, multi-language client |
| 3 | **LLM-as-Judge / Self-reflection** | ⏳ Defer | Chỉ khi <5% edge case + compliance drive |
| 4 | **Inference-time intervention** (Llama Guard, ShieldGemma, activation steering) | ⏳ Defer | Self-hosted model only, chưa robust |
| 5 | **Pre-deploy / static scanner** (PromptFoo, Garak) | ✅ M2 | CI/CD gate cho prompt template + RAG corpus |
| 6 | **Observability + retroactive guardrail** (Langfuse, Helicone, Phoenix) | ✅ M2 | Audit trail + async jailbreak alert |
| 7 | **Policy-as-code** (OPA, Cedar) | ⏳ Defer | Khi multi-tenant isolation phức tạp lên |
| 8 | **MCP-layer guardrail** | ✅ Q3 2026 | Agent i tool-use authorization |

---

## 3. Trade-off matrix (rút gọn)

| Pattern | Latency | Deploy complexity | Policy agility | B2E fit | B2C fit | Agent fit |
|---|---|---|---|---|---|---|
| API server (Guardian M1) | 50–500ms | Med | High (hot reload) | ★★★ | ★★ | ★★ |
| 1. SDK embed | 20–50ms | Low | Low (redeploy app) | ★ | ★★★ | ★★ |
| 2. Sidecar proxy | 55–115ms | Med-High | Med | ★★★ | ★ | ★★ |
| 3. LLM-as-judge | 500–2000ms | Low | High | ★★ (compliance) | ✗ | ★ |
| 4. Inference-time | 0–10ms (in-model) | High (self-host only) | Low | ★ | ★ | ★ |
| 5. Pre-deploy scanner | N/A (offline) | Low | High (CI gate) | ★★★ | ★★★ | ★★★ |
| 6. Observability | 0ms (async) | Low | High | ★★★ | ★★★ | ★★★ |
| 7. Policy-as-code | +5–10ms | High | Very high | ★★ | ★ | ★★★ |
| 8. MCP-layer | <5ms | Med | High | ★ | ★★ | ★★★ |

**Đọc bảng:** Guardian (API server) là default; SDK + observability + scanner là **complement**, không phải replacement. LLM-as-judge và inference-time có giá trị giới hạn.

---

## 4. Giá trị mỗi pattern theo consumer

### GenAI Gateway (B2E — 20K engineers ↔ external LLM)

| Pattern | Giá trị cụ thể | Pain point giải quyết |
|---|---|---|
| API server (Guardian M1) | Central policy enforcement, hot-reload rule cho 20K eng cùng lúc; tách concern khỏi GW | GW không phải implement guardrail logic; Sec team kiểm soát chính sách 1 chỗ |
| 1. SDK embed | Thấp — GW là server-side, latency hop tới Guardian không vấn đề | Không phù hợp |
| 2. Sidecar proxy | Cover client multi-language (Go/Java/Node/Python plugin của engineer) **không cần SDK riêng**; intercept tại ingress trước khi vào GW logic | Engineer dùng SDK lạ, custom client → khó force gọi Guardian; sidecar fix |
| 3. LLM-as-judge | Compliance review cho code review / sensitive doc (deep mode) — chấp nhận chờ vài giây để chắc chắn | Edge case khó của T1+T2 (vd subtle data exfil qua code comment) |
| 4. Inference-time | Chỉ giá trị nếu tập đoàn self-host model nội bộ (Llama 3 fine-tune) | N/A nếu chỉ dùng OpenAI/Anthropic API |
| 5. Pre-deploy scanner | Chặn prompt template / system prompt / RAG corpus xấu **trước khi** engineer push lên prod GW config | Hiện không có gate → bad prompt slip qua review |
| 6. Observability | Audit trail cho compliance (SOC2/ISO), trace ai dùng prompt nào → khi có incident truy vết được; async jailbreak phát hiện attack pattern mới | Không có forensic data khi incident; Sec team mù |
| 7. Policy-as-code | Phân quyền fine-grained: team A được dùng model X, team B chỉ Y; rule per-cost-center | Hiện YAML config đủ; cần khi >50 team có policy khác nhau |
| 8. MCP-layer | Engineer dùng Claude Code / Cursor có MCP tool — guardrail tool-call (vd chặn `rm -rf`, chặn DB write) | Tool-use chưa có guardrail riêng |

### Agent i (B2C — consumer chat, mobile + web)

| Pattern | Giá trị cụ thể | Pain point giải quyết |
|---|---|---|
| API server (Guardian M1) | Central policy, easy rollout fix cho UNSAFE pattern mới phát hiện | Đảm bảo policy đồng nhất cho mọi user |
| 1. SDK embed | **Latency 20–50ms** thay vì 80–150ms qua mạng → UX streaming mượt hơn; offline mode (subway, flight) vẫn block PII cơ bản | UX consumer rất nhạy latency; offline use case mobile |
| 2. Sidecar proxy | Thấp — Agent i client mobile/web, không có K8s sidecar | Không phù hợp |
| 3. LLM-as-judge | Quá chậm cho UX consumer (>500ms); cost không scale với traffic B2C | Không phù hợp |
| 4. Inference-time | Nếu có Yahoo!/LINE-tuned model nội bộ chạy on-device, có thể thêm safety classifier | Hạn chế (chỉ self-host model) |
| 5. Pre-deploy scanner | Chặn prompt persona / character config xấu trước khi release agent mới | Agent i có nhiều persona/skill → cần gate config |
| 6. Observability | Phát hiện jailbreak retroactive → patch nhanh; A/B test policy mới shadow mode trước rollout | Consumer creative, attack pattern mới liên tục → cần feedback loop |
| 7. Policy-as-code | Thấp ở M1; có giá trị khi launch nhiều domain agent (finance, health) cần policy riêng | Defer |
| 8. MCP-layer | **Cao** — Agent i sẽ có tool-calling (search, calendar, payment); guardrail authz bắt buộc | Tool-use không có policy = risk financial / data leak |

**Tóm gọn ưu tiên:**
- **GenAI GW cần ngay**: Guardian + (5) scanner + (6) observability ở M2; (2) sidecar nếu client multi-language
- **Agent i cần ngay**: Guardian + (6) observability ở M2; (1) SDK + (8) MCP ở Q3 khi tool-use launch

---

## 5. Critical findings từ research

1. **LLM-as-judge không reliable như tưởng** — paper 2024 (arxiv:2411.15594, 2410.21819) cho thấy judge model có **~37% misclassification rate** + bias prefer own output. Cost $0.01/call + latency 500–2000ms khó justify ngoại trừ <5% edge case. **→ Không bật T3 cho M1; defer hẳn.**

2. **Pre-deploy scanner có gap 40%** — analysis 6 scanner OSS (arxiv:2410.16527) cho thấy miss ~40% jailbreak. **→ Scanner KHÔNG thay thế runtime guardrail; chỉ catch low-hanging fruit ở CI.**

3. **Retroactive guardrail (observability) là win lớn** — Langfuse + async jailbreak scan → zero false positive (không block) + full audit trail. **→ Adopt M2 ngay sau Guardian launch.**

4. **Sidecar pattern bù được multi-language** — nếu GenAI GW có client Go/Java/Node không muốn integrate Guardian SDK, LiteLLM proxy + filter là alternative. Cost: +5–15ms latency.

5. **MCP guardrail là khoảng trống** — Agent i sẽ có tool-calling; chưa có pattern nào ở M1 cover. Cần plan Q3 khi MCP spec ổn định.

---

## 6. Adoption roadmap

| Phase | Thời gian | Pattern adopt | Output |
|---|---|---|---|
| M1 | Apr–May 2026 | Guardian API server (dual-mode) | Đã có |
| M2 | Jun–Aug 2026 | (a) Langfuse observability + async jailbreak<br>(b) PromptFoo + Garak ở CI/CD<br>(c) Eval LiteLLM sidecar cho GenAI GW ingress | Audit + CI gate + ingress option |
| Q3 2026 | Sep–Nov 2026 | (a) Agent i SDK embed (offline edge)<br>(b) MCP-layer guardrail cho tool-use | Edge + agentic coverage |
| Defer / revisit | Q4+ | LLM-as-judge, inference-time, OPA policy-as-code | Khi compliance / scale demand tăng |

---

## 7. Tác động lên platform watchlist

Cập nhật `docs/00-platform/solution-watchlist.md` (khi tạo) với 4 nhóm mới:

- **Sidecar / Proxy** — LiteLLM, Portkey, Helicone, Cloudflare AI Gateway, Envoy LLM filter
- **Observability** — Langfuse (OSS), Helicone, Arize Phoenix, OpenLLMetry
- **Pre-deploy scanner** — PromptFoo, Garak (NVIDIA), Giraffe, Prompt Injection detector
- **MCP guardrail** — Anthropic MCP gateway pattern, emerging spec từ Nov 2024

Watchlist hiện đã track: llm-guard, Guardrails-AI, NeMo, Bedrock Guardrails, Azure Content Safety, OpenAI Moderation, Anthropic Constitutional AI.

---

## 8. Open questions cần tech lead chốt

1. **Sidecar vs direct ingress** — GenAI GW nên đặt LiteLLM trước Guardian hay client gọi Guardian trực tiếp? Impact: thêm 1 hop nhưng cover multi-language client.
2. **SDK Agent i** — Agent i mobile có yêu cầu offline guardrail thực sự không, hay luôn online? Quyết định adopt SDK hay không.
3. **Async alert UX** — Langfuse phát hiện jailbreak sau 30s–24h, alert gửi đến đâu (Slack on-call? user chỉ log nội bộ? notify lại consumer)?
4. **Langfuse cost ở 10M req/day** — self-host hay cloud, infra budget bao nhiêu?
5. **CI scanner integration timing** — gate ở M2 hay sớm hơn cho prompt template repo?
6. **MCP policy language** — YAML / OPA / Cedar khi triển khai MCP guardrail Q3?
7. **Self-hosted model trong tập đoàn** — có không? Nếu có, inference-time intervention (Llama Guard) có giá trị; nếu không, defer hẳn.
8. **Portkey eval** — có cần evaluate Portkey như alternative cho LiteLLM + Guardian không?
