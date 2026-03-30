# cstack

> gstack is to Claude Code what cstack is to GitHub Copilot in VS Code.

A skill pack that turns GitHub Copilot into a virtual engineering team. Opinionated slash commands for every phase of a sprint — plan, review, QA, ship, and reflect — built on VS Code's [Agent Skills](https://agentskills.io) open standard.

## Quick Start

```bash
# Personal install (works across all projects)
git clone --depth 1 https://github.com/dyrota/cstack.git ~/.copilot/skills/cstack
cd ~/.copilot/skills/cstack && ./setup

# Project install
git clone --depth 1 https://github.com/dyrota/cstack.git .github/skills/cstack
cd .github/skills/cstack && ./setup --local
```

## Skills

| Command | Role | What it does |
|---|---|---|
| `/office-hours` | YC Office Hours | Reframe the problem before writing code |
| `/plan` | Eng Manager | Read-only implementation plan with diagrams |
| `/review` | Staff Engineer | Find production bugs before they ship |
| `/qa` | QA Lead | Run tests, find gaps, write missing coverage |
| `/ship` | Release Engineer | Sync, test, commit, open PR |
| `/investigate` | Debugger | Systematic root-cause debugging |
| `/document` | Tech Writer | Update docs to match what shipped |
| `/retro` | Eng Manager | Weekly shipping retrospective |
| `/cso` | Security Officer | OWASP + STRIDE audit |

## Workflow

```
/office-hours → /plan → [implement] → /review → /qa → /ship → /document
```

## Why

Copilot is powerful. cstack makes it structured. Each skill knows its role, its tools, and when to hand off to the next step.

See [SPEC.md](./SPEC.md) for the full design.

## License

MIT
