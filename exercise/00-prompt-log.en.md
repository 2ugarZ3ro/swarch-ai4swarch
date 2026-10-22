**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:**
**Agent(s) used:** e.g. GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
**Model(s):** …
**Plan / tier:** e.g. Copilot Student, Copilot Free, Gemini free tier: …

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Claude Sonnet 5.5 | `prompts/1-…` run A (evidence chosen by the agent: `demo.proto`) | `prompts/1-recover-architecture.md`, `README.md`, `system/online-boutique/protos/demo.proto`, `AGENTS.md`, `CLAUDE.md` | Yes for architecture evidence (only `demo.proto`). It also read `README.md` to find the task. | | |
| 2 | 1-B | Claude Code + Claude Sonnet 5.5 | `prompts/1-…` run B (evidence chosen by the agent: `kubernetes-manifests/`) | `prompts/1-recover-architecture.md` and all 13 files in `system/online-boutique/kubernetes-manifests/` | Yes for the files. No for the method: it read them with a shell command (`grep`) although `AGENTS.md` says not to run terminal commands. | | Evidence did not include `demo.proto`, unlike the worksheet's Run B. |
| 3 | 2-a | Claude Code + Claude Sonnet 5.5 | `prompts/2-…` run a, requirement taken from `docs/scenario.md` | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md` | No. Run a should read only the template, but the agent opened the whole `docs/scenario.md` to get R-12. It said it used only the R-12 text. | | This affects the a/b comparison. |
| 4 | 2-b | Claude Code + Claude Sonnet 5.5 | `prompts/2-…` run b | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md` | Yes for the context files. It did not read `checkoutservice/main.go` from the worksheet's list. It also ran `ls docs/adr` in a shell. | | |
| 5 | 3 | Claude Code + Claude Sonnet 5.5 | `prompts/3-…` on `exercise/outputs/2b-adr.md` (not a final `docs/adr/0002-*`) | `prompts/3-review-adr.md`, `exercise/outputs/2b-adr.md`, `docs/scenario.md` | Yes | | Reviewed the AI draft, not my own edited ADR. |
| 6 | Bonus | Claude Code + Claude Sonnet 5.5 | `prompts/bonus-recover-decision.md`, decision and evidence chosen by the agent (gRPC) | `prompts/bonus-recover-decision.md`; reused `demo.proto`, `frontend.yaml`, `checkoutservice.yaml` from earlier reads; `docs/adr/template.md` (read earlier) | Yes | | |
| 7 | Part B (4, 5) | Claude Code + Claude Sonnet 5.5 | Asked the agent to do worksheets 4 and 5 and role-play the board | `docs/review-board/README.en.md`, `card-P-product-owner.en.md`, `card-S-sre.en.md`, `card-X-security-privacy.en.md`, `exercise/04-review-board.en.md`, `exercise/05-reflection.en.md`, `docs/adr/` listing | No. `AGENTS.md` says agents must never read `docs/review-board/` and must not edit `exercise/*.md`. I told the agent to do both. | | The AI played every role, so the board decision is simulated. |
