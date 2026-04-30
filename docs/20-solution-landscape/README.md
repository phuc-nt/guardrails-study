# 20 — Solution Landscape

Phân tích giải pháp guardrail trên thế giới (open-source, cloud vendor, LLM-native, academic). Đầu vào cho design ở [`../10-guardian-design/`](../10-guardian-design/).

## Files

- [03-open-source-reference-analysis.md](./03-open-source-reference-analysis.md) — so sánh llm-guard, Guardrails-AI, NVIDIA NeMo
- [04-top-5-adopt-patterns-explained.md](./04-top-5-adopt-patterns-explained.md) — 5 pattern chốt khuyến nghị cho M1

## Naming convention

`landscape-{topic-slug}.md` — file mới về một giải pháp / nhóm pattern.
Khi cập nhật, sửa trực tiếp + thêm header `**Updated YYYY-MM-DD**`.

## Scope theo dõi (rolling)

- **Open-source frameworks**: llm-guard, Guardrails-AI, NVIDIA NeMo Guardrails, LiteLLM guards
- **Cloud vendor**: AWS Bedrock Guardrails, Azure AI Content Safety, GCP Model Armor
- **LLM-native**: OpenAI Moderation, Anthropic Constitutional AI, Anthropic Prompt Shield
- **Academic**: prompt injection defense, jailbreak detection, PII masking, multilingual NER (JP/KR/TH)
- **Standards**: NIST AI RMF, EU AI Act, OWASP LLM Top 10

## Lưu ý

Đây là **research/desk study** — không có benchmark trên data tập đoàn ở repo này. Khi team nội bộ muốn POC một giải pháp, họ đọc landscape note → quyết định scope POC → thực hiện ở môi trường nội bộ với data governance phù hợp.
