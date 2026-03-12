---
name: 'step-05-present'
description: 'Present findings, get approval, create PR'
---

# Step 5: Present

## RULES

- YOU MUST ALWAYS SPEAK OUTPUT in your Agent communication style with the config `{communication_language}`
- NEVER auto-push.
- 🌐 CRITICAL: SEARCH THE WEB for current data before making technical claims or decisions. Your knowledge has a cutoff — the real world does not stop. Do NOT ask permission, just search.
- 📜 CRITICAL: READ AND FOLLOW `{project-root}/_bmad/bmm/data/global-agent-rules.md` — cross-workflow rules that apply at every step, even after context compaction.

## INSTRUCTIONS

1. Change `{spec_file}` status to `done` in the frontmatter.
2. If version control is available and the tree is dirty, create a local commit with a conventional message derived from the spec title.
3. Display summary of your work to the user, including the commit hash if one was created. Advise on how to review the changes. Offer to push and/or create a pull request.

Workflow complete.
