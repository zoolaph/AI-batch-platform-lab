# CLAUDE.md — Project Context for Claude Code

## What this project is

A reproducible AI batch platform lab comparing Slurm and Kubernetes GPU scheduling.
Public portfolio project — the commit history is part of the signal.

## Rules

### File comment convention

Every non-placeholder file must open with a purpose comment.

- Shell / Python / YAML / Makefile / Terraform HCL: `# Purpose: <one line>`
- Markdown: `<!-- Purpose: <one line> -->` on line 1
- JSON: no comment syntax — add purpose to a sibling README or `purpose` key

### Block comment convention

Comment *why*, not *what*. Inline comments only for non-obvious logic.

### Commit discipline

- One logical change per commit
- Imperative mood, ≤72 chars, no trailing period
- Include issue ref: `feat(infra): add GPU node Terraform (#3)`

### File placement

Follow Section 4 of PROJECT_PLAN.md exactly. Never create files outside the
defined tree without updating the plan first.

### IaC rules

- Never commit `*.tfvars` — they contain secrets
- Terraform state is remote — never commit `*.tfstate` or `*.tfstate.backup`
- Ansible roles must be idempotent: running twice produces zero changes

### Python rules

- All scripts: `#!/usr/bin/env python3`
- Linter: `ruff` — run before every commit

### Kubernetes rules

- All manifests must pass `kubeconform` before commit
- Always specify namespaces — never rely on `default`

## Phase awareness

Current phase: **0 — Foundation**

See PROJECT_PLAN.md for per-phase validation criteria.
Do not close an issue until every validation checkbox is checked.

## What to avoid

- Do not add files outside Section 4 without updating PROJECT_PLAN.md
- Do not skip validation checkboxes — they are acceptance criteria
- Do not commit credentials, API keys, or kubeconfig files
- Do not amend published commits — always create a new one
