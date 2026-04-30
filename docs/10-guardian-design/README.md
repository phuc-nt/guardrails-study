# 10 — Guardian Design (Milestone 1)

Design docs cho Guardian API server M1, phục vụ GenAI Gateway (B2E) và Agent i (B2C).

Output ở đây dùng làm **spec/blueprint** cho team nội bộ implement trên repo production.

## Files

- [01-architecture-design.md](./01-architecture-design.md) — system architecture, component, K8s topology
- [02-technical-proposal.md](./02-technical-proposal.md) — bài toán, constraint, tech stack, roadmap
- [05-scan-flow-design.md](./05-scan-flow-design.md) — function flow cho mọi tổ hợp check-point × protocol × action pattern
- [06-dual-mode-proposal.md](./06-dual-mode-proposal.md) — đề xuất 2 mode `fast` (latency) / `deep` (quality), tier scanner, mode selector, SLA
- [07-architectural-patterns-proposal.md](./07-architectural-patterns-proposal.md) — 8 pattern guardrail ngoài API server (SDK, sidecar, observability, MCP, …) đáng adopt cho platform
- [08-agent-i-guardrails-proposal.md](./08-agent-i-guardrails-proposal.md) — Agent i Guardrail Suite: 10 module (PII, Memory, Payment, Medical, Copyright, Merchant, Commerce, Financial, News, Child) theo integration map LINE × Yahoo!

## Files dự kiến bổ sung

- `api-contracts.md` — REST/SSE schema, error codes, headers
- `tenant-onboarding.md` — quy trình thêm consumer mới (genai-gw, agent-i, …)
- `threat-model.md` — STRIDE / abuse case cho B2E vs B2C

## Liên quan

- Cơ sở giải pháp → [`../20-solution-landscape/`](../20-solution-landscape/)
- Vision / roadmap / watchlist → [`../00-platform/`](../00-platform/)
