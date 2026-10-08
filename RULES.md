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

## 3. Git Commit Message Format
All commits made by developers or AI agents must strictly follow this format:
```
GENAI=YES/NO <type>(<scope>): <description>
```
- Set `GENAI=YES` if any AI tools were used while making changes in the codebase.
- Set `GENAI=NO` if no AI tools were used.
- Example: `GENAI=YES feat(core): initialize memory extraction`

## 4. Post-Implementation Summary (`GIT.md`)
Upon successful completion of any implementation plan/task, the agent must write a `GIT.md` file in the root directory summarizing everything implemented in that stage. The developer or agent reads this summary to craft a strong commit message. Once committed, `GIT.md` must be archived to `docs/history/`.

