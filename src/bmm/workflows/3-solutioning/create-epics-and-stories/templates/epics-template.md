---
stepsCompleted: []
inputDocuments: []
---

# {{project_name}} - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for {{project_name}}, decomposing the requirements from the PRD, UX Design if it exists, and Architecture requirements into implementable stories.

## Requirements Inventory

### Functional Requirements

{{fr_list}}

### NonFunctional Requirements

{{nfr_list}}

### Additional Requirements

{{additional_requirements}}

### FR Coverage Map

{{requirements_coverage_map}}

## Story Size Rules

> CRITICAL: Each story MUST be small enough to implement in a SINGLE Claude Code session (~200k tokens).
> A story that causes context compaction = a story that loses context = bugs and rework.

| Constraint | Max | If exceeded |
|---|---|---|
| Acceptance Criteria per story | 5 | Split into sub-stories |
| Tasks per story | 3 | Split into sub-stories |
| Files modified per story | 5-7 | Split by domain/layer |
| Estimated implementation | 1 session | Split into smaller stories |

> Format: concise Given/When/Then. No prose. No explanations in AC — only verifiable criteria.

---

## Epic List

{{epics_list}}

<!-- Repeat for each epic in epics_list (N = 1, 2, 3...) -->

## Epic {{N}}: {{epic_title_N}}

{{epic_goal_N}}

<!-- Repeat for each story (M = 1, 2, 3...) within epic N -->

### Story {{N}}.{{M}}: {{story_title_N_M}}

As a {{user_type}},
I want {{capability}},
So that {{value_benefit}}.

**Acceptance Criteria:** (max 5)

<!-- for each AC on this story -->

**Given** {{precondition}}
**When** {{action}}
**Then** {{expected_outcome}}

<!-- End story repeat -->

---

## Feature Coverage Check

> At the end of epic/story creation, verify against checkup-feature.md:
> - [ ] Every feature in checkup-feature.md has at least 1 story
> - [ ] Every security-critical feature has security acceptance criteria
> - [ ] No feature has been silently dropped
