# CLAUDE.md

- **Mandatory:** Read the `RULES.md` file first and follow all the rules mentioned in it.

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:

- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:

- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

## 5. Structured Execution Loop (Strict Checklists & HITL)

**Plan first, execute sequentially, never skip steps.**

1. **Understand & Clarify:** Thoroughly understand the requirement before taking action. If anything is ambiguous or underspecified, stop and ask the user for clarification first.
2. **Create a `plan_*.md` File:** Before touching or modifying any code/files, always create a `plan_<task>.md` file in the working directory:
   - Break the task into distinct **Phases**.
   - Under each phase, list discrete, actionable steps formatted with checkboxes (`- [ ]`).
3. **Strict Sequential Execution:**
   - Work through checkboxes one at a time.
   - **Strict rule:** Never proceed to the next step until all preceding checkboxes are fully executed, verified, and checked off (`- [x]`).
4. **Human-in-the-Loop (HITL) on Blockers:**
   - If any issue, implementation blocker, or unexpected behavior arises, stop immediately and ask the user.
   - Never attempt speculative fixes or bypass an uncompleted/failing checkbox.
5. **Post-Task Summary (`GIT.md`):**
   - Upon successful completion and verification of the plan, it is the agent's explicit duty to create a `GIT.md` file in the root directory.
   - Summarize all changes, implementations, touched files, and verification steps from this stage.
6. **Commit & Archive:**
   - Read the summary in `GIT.md` to formulate a strong, descriptive commit message following Section 6.
   - Once committed, move both `plan_*.md` and `GIT.md` into `docs/history/` as required by `RULES.md`.

## 6. Git Commit Format

**Strict commit message convention for developers and agents.**

All commits must strictly follow this format:

```
GENAI=YES/NO <type>(<scope>): <description>
```

- **`GENAI=YES`**: Mandatory if any AI tools/assistants were used while changing or generating anything in the codebase.
- **`GENAI=NO`**: Only when changes were made completely without AI assistance.
- **Workflow**:
  1. Read the summary in `GIT.md`.
  2. Craft a strong, descriptive commit message following this format.
  3. Commit the changes.
  4. Move `GIT.md` to `docs/history/`.
- **Example**: `GENAI=YES feat(auth): add google oauth flow` or `GENAI=NO fix(docs): fix typo in README`

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
