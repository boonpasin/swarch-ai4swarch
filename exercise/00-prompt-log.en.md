**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Pasin Thanyakasikol / …
**Agent(s) used:** Claude Code (desktop app); separate subagent instances per run
**Model(s):** Claude Sonnet 5.5 (`claude-sonnet-5-5`) for every task run; Claude Opus 5.5 (`claude-opus-5-5`) as the auditor/orchestrator
**Plan / tier:** …

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, Sonnet 5.5 | `prompts/1-…` run A, evidence `protos/demo.proto` | `prompts/1-recover-architecture.md`, `protos/demo.proto` (as reported) | Yes (as reported) | 2 | Correct but almost empty: 9 services, 0 dependencies. Inferred deps correctly kept in Gaps. Orphan `Person`. |
| 2 | 1-B | Claude Code, Sonnet 5.5 | `prompts/1-…` run B, evidence proto + `kubernetes-manifests/` | task file, `demo.proto`, all 14 files in `kubernetes-manifests/` (as reported) | Yes (as reported) | 4 | All 17 deps verified. Missed HTTP edge to `shoppingassistantservice` (kept in Gaps); `loadgenerator` inside boundary. |
| 3 | 2-a | Claude Code subagent, Sonnet 5.5 | `prompts/2-…` run a, context none, R-12 | `AGENTS.md`, `prompts/2-write-adr.md`, `docs/adr/template.md` | Yes, did **not** open `docs/scenario.md` | 3 | Generic broker fan-out, no numbers, honest Open questions; offered an orchestrator option b lacked. |
| 4 | 2-b | Claude Code subagent, Sonnet 5.5 | `prompts/2-…` run b, context scenario + ADR-0001 + `checkoutservice/main.go`, R-12 | `AGENTS.md`, `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-…`, `src/checkoutservice/main.go` | Yes | 4 | Used every scenario number, chose RabbitMQ. Weak: best-effort publish loses events, no per-consumer queues, straw-man managed option. |
| 5 | 3 | Claude Code subagent, Sonnet 5.5 | `prompts/3-…`, adr `docs/adr/0002-publish-order-placed-events-to-rabbitmq.md` | `AGENTS.md`, `prompts/3-review-adr.md`, the ADR, `docs/scenario.md` | Yes; did not edit the ADR | 4 | Accept with changes. Best: in-memory buffer loss window, DLQ draining during a MailFast outage. |
| 6 | Audit | Claude Code, Opus 5.5 | "Act as me, critique 1A/1B", then "fill the exercises" | `AGENTS.md`, `CLAUDE.md`, all prompts and worksheets, task outputs, `docs/scenario.md`, `docs/adr/*`, all manifests, parts of `src/frontend`, `src/checkoutservice`, `src/cartservice`, `src/recommendationservice`, `src/productcatalogservice` | **No** (by student permission, time exception): read source and ran read-only shell commands | 4 | Verified Task 1-B, ran tasks 2a/2b/3 as separate subagents, wrote ADR-0002 from 2b and filled the worksheets. **The ADR edits were made by the AI, not by me, against the Task 2 instruction.** |
| 7 | | | | | | | |
