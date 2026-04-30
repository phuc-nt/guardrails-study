# 10 — Guardian Design (Milestone 1)

Design docs cho Guardian API server M1, phục vụ GenAI Gateway (B2E) và Agent i (B2C).

Output ở đây dùng làm **spec/blueprint** cho team nội bộ implement trên repo production.

## Files

- [01-architecture-design.md](./01-architecture-design.md) — system architecture, component, K8s topology
- [02-technical-proposal.md](./02-technical-proposal.md) — bài toán, constraint, tech stack, roadmap
- [05-scan-flow-design.md](./05-scan-flow-design.md) — function flow cho mọi tổ hợp check-point × protocol × action pattern

## Files dự kiến bổ sung

- `api-contracts.md` — REST/SSE schema, error codes, headers
- `tenant-onboarding.md` — quy trình thêm consumer mới (genai-gw, agent-i, …)
- `threat-model.md` — STRIDE / abuse case cho B2E vs B2C

## Liên quan

- Cơ sở giải pháp → [`../20-solution-landscape/`](../20-solution-landscape/)
- Vision / roadmap / watchlist → [`../00-platform/`](../00-platform/)
