# Security DATA

This folder contains universal security reference data for BMAD workflows.
Agents load these files on-demand based on context — never all at once.

## How It Works

1. Workflows read `index.md` first (file listing with tags)
2. Match tags against the current story/code context
3. Load only 3-5 relevant files

## File Naming

- `atk-*` — Attack patterns (how attackers exploit)
- `def-*` — Defense patterns (how to protect)
- `ref-*` — Reference data (threat models, CVE catalogs, matrices)

## Adding Data

Add `.md` files following the naming convention above.
Update `index.md` with the new file, tags, and one-line summary.
