# Guardrail Platform — Research & Design Documentation

**Repo này là tài liệu research + design**, không phải production codebase. Output ở đây phục vụ nhóm developer **trong môi trường tập đoàn LY Corp** triển khai thực tế (benchmark, experiment, deploy diễn ra ở repo nội bộ khác).

> **Vision:** Platform sống — liên tục cập nhật giải pháp guardrail mới của thế giới để team nội bộ thử nghiệm và áp dụng. Không dừng lại ở module API server (Guardian M1).

## Cấu trúc

| Thư mục | Nội dung | Audience chính |
|---|---|---|
| [`00-platform/`](./00-platform/) | Vision, roadmap, principles, multi-tenant model, watchlist giải pháp đang theo dõi | Tech Lead, CISO, PM platform |
| [`10-guardian-design/`](./10-guardian-design/) | Design của Guardian (M1: API server cho GenAI GW + Agent i) | Eng team M1 (triển khai nội bộ) |
| [`20-solution-landscape/`](./20-solution-landscape/) | Phân tích giải pháp / pattern trên thế giới (open-source, vendor, paper) — input cho design | Eng + research |

## Phạm vi repo này

**Có:**
- Phân tích, so sánh giải pháp trên internet
- Đề xuất architecture / API contract / function flow
- Pattern recommendation kèm lý do
- Constraint, trade-off, decision context để team nội bộ tham chiếu

**Không có:**
- Source code production
- Benchmark trên data nội bộ tập đoàn (làm ở repo nội bộ với data governance)
- Experiment log / shadow mode result (làm ở repo nội bộ)
- Production runbook, on-call config

→ Khi team nội bộ cần benchmark / experiment / triển khai, họ **đọc docs ở đây làm input** rồi thực hiện ở môi trường có data + infra phù hợp.

## Quy ước cập nhật

- Giải pháp mới phát hiện → research note ở `20-solution-landscape/`
- Pattern đáng adopt cho M1 → cập nhật vào `10-guardian-design/`
- Roadmap / watchlist tổng → `00-platform/`
- Doc cũ outdated → cập nhật trực tiếp với header "**Updated YYYY-MM-DD**", không tạo file mới song song

## Consumer hiện tại (M1)

- **GenAI Gateway** (B2E) — proxy cho ~20K engineer ↔ external LLM (OpenAI, Anthropic)
- **Agent i** (B2C) — unified AI agent của LY Corp (Yahoo! JAPAN + LINE), 7 Domain Agents, có memory + task execution. Ref: https://www.lycorp.co.jp/en/news/release/020398/
