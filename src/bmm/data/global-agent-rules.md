# Global Agent Rules

> These rules apply to ALL agents in every phase.
> Load this file at the start of any workflow execution.

## Web Search Systematic

EVERY agent MUST search the web for up-to-date information BEFORE working on a topic.
Do NOT ask permission. Do it automatically when:
- A technical decision needs current data (latest versions, CVEs, best practices)
- A market or competitive analysis is needed
- A security review references threats or attack patterns
- Architecture decisions reference specific technologies
- Any topic where the agent's training data might be outdated

The agent's knowledge has a cutoff date. The real world does not stop.
Search first, then work.

## Checkup Feature Tracking

After EACH phase completion, the agent MUST:
1. Load `{planning_artifacts}/checkup-feature.md` if it exists
2. Verify that all features are still tracked through the current phase
3. Flag any feature that was not addressed in the current phase
4. Add any new features discovered during the phase

## Story Size Enforcement

Stories MUST be small enough for a single Claude Code session:
- Max 5 acceptance criteria
- Max 3 tasks
- Max 5-7 files modified
- If a story is too large: split into sub-stories BEFORE implementation

## Code Review Scope

The review agent MUST NOT:
- Rewrite code in files not modified by the story
- Add features not in the acceptance criteria
- Refactor code unrelated to the story
- Add comments/docstrings to unchanged code

## Security DATA Loading

When security is relevant, agents MUST:
1. Load `{project-root}/_bmad/bmm/data/security/index.md` FIRST
2. Match tags in the index against the current context (tech stack, story domain)
3. Load ONLY the 3-5 most relevant files — NEVER load all files at once

These files are loaded ON DEMAND (not permanently in context).
NEVER skip this step for security-relevant work.
