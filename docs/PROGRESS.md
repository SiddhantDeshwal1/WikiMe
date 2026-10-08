# Problem Statement

- Human memory is not built to retain large volumes of fast-moving, passive content — most of what's consumed is forgotten within days
- Short-form content (Reels, Shorts, TikToks) is scattered across multiple platforms with no unified way to organize or reference it later
- Users save content with intent to revisit ("I'll come back to this"), but rarely do — saved items pile up unseen, and users often forget they ever saved it at all

# Objective

Build an AI agent that acts as an external memory layer for the user — capturing, understanding, and retaining social content automatically, so recall doesn't depend on human memory

# Solution

- Capture — user shares a link/content with the agent
- Process — agent extracts and summarizes it
- Confirm — user approves the summary before it's saved
- Store — saved into a persistent, searchable memory
- Recall — user asks questions later; agent answers from stored memory
- Manage — user updates/deletes memories via natural language

# Roadmap & Tasks

- [x] **V1_001: Improving the Onboarding**
  - [x] Standardize git commit format with `GENAI=YES/NO` convention in `RULES.md` and `GEMINI.md`
  - [x] Establish post-task summary generation (`GIT.md`) and archiving into `docs/history/`
  - [x] Configure `.gitignore` to ignore root `GIT.md`
  - [x] Establish sprint tracking workflow (`SPRINT.md`) and task registration policy in `docs/PROGRESS.md`
  - [x] Add pre-commit policy violation guardrails and task ID validation
  - [x] Create `git-commit` skill (`.agents/skills/git-commit/SKILL.md`) enforcing pre-commit verification and archiving

