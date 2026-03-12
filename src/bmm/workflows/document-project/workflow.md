---
name: document-project
description: 'Document brownfield projects for AI context. Use when the user says "document this project" or "generate project docs"'
---

# Document Project Workflow

**Goal:** Document brownfield projects for AI context.

**Your Role:** Project documentation specialist.
- Communicate all responses in {communication_language}

---

## INITIALIZATION

### Configuration Loading

Load config from `{project-root}/_bmad/bmm/config.yaml` and resolve:

- `project_knowledge`
- `user_name`
- `communication_language`
- `document_output_language`
- `user_skill_level`
- `date` as system-generated current datetime

### Paths

- `installed_path` = `{project-root}/_bmad/bmm/workflows/document-project`
- `instructions` = `{installed_path}/instructions.md`
- `validation` = `{installed_path}/checklist.md`
- `documentation_requirements_csv` = `{installed_path}/documentation-requirements.csv`

---

## PRE-EXECUTION (MANDATORY)

Before executing ANY step below, you MUST:

1. **Read and follow** `{project-root}/_bmad/bmm/data/global-agent-rules.md`
2. **Search the web** for up-to-date information relevant to this workflow's topic — your knowledge has a cutoff date

## EXECUTION

Read fully and follow: `{installed_path}/instructions.md`
