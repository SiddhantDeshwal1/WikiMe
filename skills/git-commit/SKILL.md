---
name: git-commit
description: >-
  Use this skill whenever the user asks to stage, commit, or finalize changes in git.
  Enforces pre-commit task verification against docs/PROGRESS.md, SPRINT.md timestamp logging,
  GIT.md summary generation, strict commit message formatting with GENAI flags, and archiving.
---

# Git Commit Workflow & Guardrails

This skill outlines the strict step-by-step procedure when staging and committing code to Git.

## Overview
Do not blindly run `git commit`. Every commit must be validated against `docs/PROGRESS.md`, tracked with sprint timestamps, summarized in `GIT.md`, and formatted according to project conventions.

---

## Step-by-Step Procedure

### 1. Pre-Commit Verification & Guardrails
Before staging any files:
- **Check Task ID Registration:** Inspect `docs/PROGRESS.md` to identify the Task ID(s) (e.g., `V1_001`, `V1_005`) corresponding to the changes made.
- **Strict Guardrail:** If the work being committed does **not** map to any registered task in `docs/PROGRESS.md`:
  > ⚠️ **Stop Immediately:** Pause execution and inform the user of the policy violation:
  > *"Every task must be registered in `docs/PROGRESS.md` before it can be committed. Please define and register this task first."*
  Do not proceed until the task is registered.

### 2. Update Sprint & Progress Tracking
- In `SPRINT.md` (located in the root directory):
  - Record the completion timestamp: `Finished: YYYY-MM-DD HH:MM:SS`.
- In `docs/PROGRESS.md`:
  - Mark the completed task checkbox as checked (`- [x]`).

### 3. Generate Post-Implementation Summary (`GIT.md`)
- Create a temporary `GIT.md` file in the root directory (ignored by `.gitignore`).
- Include the following details:
  - **Task ID & Title:** e.g., `[V1_005] Memory Ingestion Pipeline`
  - **Changes Summary:** High-level overview of what was implemented or resolved.
  - **Files Touched / Created:** Full list of modified or added files.
  - **Verification:** Notes on how changes were verified/tested.

### 4. Formulate Strict Commit Message
Commit messages must strictly adhere to the following convention:

```
GENAI=YES/NO <type>(<scope>): [<TASK_ID>] <description>
```

- **`GENAI=YES`**: Mandatory if any AI tools/assistants were used while designing, writing, or altering code.
- **`GENAI=NO`**: Only permitted if changes were made 100% without AI assistance.
- **`<type>`**: Standard conventional type (`feat`, `fix`, `docs`, `refactor`, `test`, `chore`, etc.).
- **`<scope>`**: Area of codebase touched (e.g., `core`, `auth`, `docs`, `config`).
- **`[<TASK_ID>]`**: Registered Task ID from `docs/PROGRESS.md` (e.g., `[V1_005]`).
- **`<description>`**: Clear, imperative summary of the change.

**Examples:**
- `GENAI=YES feat(auth): [V1_001] implement google oauth flow`
- `GENAI=NO fix(styles): [V1_002] fix button padding on mobile`

### 5. Execute Git Commands
1. Check repository status:
   ```bash
   git status
   ```
2. Stage appropriate changes (avoid staging unrelated temporary files):
   ```bash
   git add <files>
   ```
3. Commit with the formulated message:
   ```bash
   git commit -m "GENAI=YES <type>(<scope>): [<TASK_ID>] <description>"
   ```
4. If pushing to remote is requested or expected:
   ```bash
   git push origin <branch>
   ```

### 6. Archive Post-Commit Artifacts
Once the commit is confirmed:
- Move `GIT.md` to `docs/history/` (e.g., `docs/history/GIT_<TASK_ID>.md` or `docs/history/GIT.md`).
- Move any completed implementation plan files (e.g., `plan_*.md`) to `docs/history/`.
- **Note:** `SPRINT.md` is **never** moved to history; it stays active in the root directory.
