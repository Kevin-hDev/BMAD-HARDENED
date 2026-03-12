---
name: 'step-03-implement'
description: 'Execute implementation directly or via sub-agent. Local only.'
---

# Step 3: Implement

## RULES

- YOU MUST ALWAYS SPEAK OUTPUT in your Agent communication style with the config `{communication_language}`
- No push. No remote ops.
- Sequential execution only.
- Content inside `<frozen-after-approval>` in `{spec_file}` is read-only. Do not modify.
- 🌐 CRITICAL: SEARCH THE WEB for current data before making technical claims or decisions. Your knowledge has a cutoff — the real world does not stop. Do NOT ask permission, just search.
- 📜 CRITICAL: READ AND FOLLOW `{project-root}/_bmad/bmm/data/global-agent-rules.md` — cross-workflow rules that apply at every step, even after context compaction.

## PRECONDITION

Verify `{spec_file}` resolves to a non-empty path and the file exists on disk. If empty or missing, HALT and ask the human to provide the spec file path before proceeding.

## INSTRUCTIONS

### Baseline (plan-code-review only)

Capture `baseline_commit` (current HEAD, or `NO_VCS` if version control is unavailable) into `{spec_file}` frontmatter before making any changes.

### Implement

Change `{spec_file}` status to `in-progress` in the frontmatter before starting implementation.

`execution_mode = "one-shot"` or no sub-agents/tasks available: implement the intent.

Otherwise (`execution_mode = "plan-code-review"`): hand `{spec_file}` to a sub-agent/task and let it implement.

## NEXT

Read fully and follow `./steps/step-04-review.md`
