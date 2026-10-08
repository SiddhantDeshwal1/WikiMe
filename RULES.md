# Project Rules

These rules apply to both developers and AI agents working on this project.

## 1. Documentation Organization
All documentation files must reside in the `docs/` directory. This includes, but is not limited to:
- Architecture specifications
- File structure overviews
- Core fundamentals and requirements
- Design docs and technical guides

## 2. Implementation Notes & Markdown Archiving
Any intermediate markdown files, task notes, or scratchpads created during implementation (e.g., `notes.md`, task checklists, temporary scratch notes, etc.) must be moved to the `docs/history/` directory once they have served their purpose or implementation has concluded.

## 3. Progress Tracking & Sprint Workflow (`docs/PROGRESS.md` & `SPRINT.md`)

### A. Master Progress (`docs/PROGRESS.md`)
- All project roadmap items, ideas, and feature tasks must be maintained in `docs/PROGRESS.md` using checkbox format (`- [ ]`) with structured task IDs (e.g., `V1_001`, `V1_002`).
- **Strict Registration Policy:** Every task or feature must exist in `docs/PROGRESS.md` *before* it can ever be loaded into `SPRINT.md` or implemented.
  - If a user requests a feature that is not present in `docs/PROGRESS.md`, the agent must **strictly pause and ask the user** to define and register the task into `docs/PROGRESS.md` first.

### B. Task Intake & Assignment Conflicts
1. **Intake & Matching:**
   - Always search `docs/PROGRESS.md` first.
   - If a similar task exists, ask the user: *"Found task `V1_005` similar to your request, should I load it into SPRINT.md?"*
2. **Handling Existing / Assigned Tasks in `SPRINT.md`:**
   - If `SPRINT.md` already contains an active task assigned to a user:
     - The agent **must ask the user**: *"Task [ID: Name] is already active and assigned to @<user>. Should I add another task to the sprint, replace the current task, or finish the existing task first?"*
   - **Paused/Replaced Tasks:** If an unfinished task is replaced:
     - It must be moved back / marked paused in `docs/PROGRESS.md`.
     - Append a mandatory `NOTE:` under that task in `docs/PROGRESS.md` detailing:
       - What was completed so far.
       - Why it was paused / replaced.
     - The agent must explicitly ask the user for both pieces of information before updating `docs/PROGRESS.md`.

### C. Active Sprint Tracking (`SPRINT.md`)
- Maintained at the root of the project to actively track the current sprint.
- **Mandatory Task Metadata:**
  - **Assignee / Author:** Tagged with GitHub username (e.g., `Author/Assignee: @siddhantdeshwal1`).
  - **Task ID & Name:** (e.g., `Task: V1_005 - Memory Ingestion`).
  - **Timestamps:**
    - `Started: YYYY-MM-DD HH:MM:SS` (recorded when sprint begins).
    - `Finished: YYYY-MM-DD HH:MM:SS` (recorded when sprint work concludes, prior to commit).
  - **Checklists:** Actionable checklist of subtasks (`- [ ]`).
- **Retention Rule:** `SPRINT.md` is **never** moved to `docs/history/`. It remains actively tracked in the root directory.

## 4. Git Commits
All staging, pre-commit task verification against `docs/PROGRESS.md`, `GIT.md` change summary creation, commit message formatting, and post-commit archiving are delegated to the `git-commit` skill.



