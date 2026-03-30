# cstack — Specification

> A skill pack for GitHub Copilot in VS Code, inspired by [gstack](https://github.com/garrytan/gstack).
> gstack is to Claude Code what cstack is to GitHub Copilot + VS Code.

---

## What is cstack?

cstack is an opinionated, open-source set of **Agent Skills**, **Custom Agents**, and **Prompt Files** for GitHub Copilot in VS Code. It turns Copilot from a general-purpose autocomplete into a virtual engineering team with specialized roles.

The goal: install cstack once, and gain `/plan`, `/review`, `/qa`, `/ship`, and more — all wired to the right Copilot tools, models, and handoffs.

---

## Core Principles

1. **Markdown-first.** Everything is a `.agent.md`, `SKILL.md`, or `.prompt.md`. No compiled binaries, no build step required.
2. **Portable.** Skills follow the [Agent Skills open standard](https://agentskills.io) and work across VS Code Copilot, Copilot CLI, and the Copilot coding agent.
3. **Opinionated defaults, easy overrides.** Each skill ships with sensible defaults. Users can fork and customize without touching the core.
4. **Think → Plan → Build → Review → Test → Ship → Reflect.** Skills map to sprint phases and feed into each other.
5. **GitHub Copilot-native.** Uses Copilot's built-in tools (`codebase`, `edit`, `web`, `terminal`, etc.) — no MCP dependency required (MCP is opt-in).

---

## Directory Structure

```
cstack/
├── README.md
├── SPEC.md                        ← this file
├── setup                          ← install script (bash)
├── .github/
│   └── copilot-instructions.md    ← cstack meta-instructions
├── skills/                        ← Agent Skills (SKILL.md standard)
│   ├── office-hours/              ← Start here. Reframe the problem.
│   ├── plan/                      ← Generate implementation plan (read-only)
│   ├── review/                    ← Staff-level code review
│   ├── qa/                        ← QA with real terminal + test runner
│   ├── ship/                      ← Run tests, commit, open PR
│   ├── retro/                     ← Weekly retrospective
│   ├── document/                  ← Update docs to match what shipped
│   ├── investigate/               ← Systematic debugging
│   └── cso/                       ← Security audit (OWASP + STRIDE)
├── agents/                        ← Custom Agents (.agent.md)
│   ├── planner.agent.md           ← Read-only planning persona
│   ├── reviewer.agent.md          ← Code review persona
│   ├── implementer.agent.md       ← Full-edit implementation persona
│   ├── qa.agent.md                ← QA persona with terminal access
│   └── security.agent.md          ← Security-only persona
└── prompts/                       ← Reusable prompt files (.prompt.md)
    ├── pr-description.prompt.md
    ├── commit-message.prompt.md
    └── test-coverage.prompt.md
```

---

## Skills

### `/office-hours`
**Role:** YC-style office hours facilitator  
**When:** Start of any new feature or project  
**What it does:**
- Asks 6 forcing questions to challenge framing, scope, and assumptions
- Pushes back on feature requests by surfacing the underlying pain
- Generates 3 implementation approaches with effort estimates
- Produces a `DESIGN.md` that feeds downstream skills

**Tools:** `web/fetch`, `search/codebase`  
**Model:** Claude Sonnet or GPT-5.2 (copilot)

---

### `/plan`
**Role:** Engineering Manager  
**When:** After `/office-hours` or when starting a feature  
**What it does:**
- Reads `DESIGN.md` if present
- Generates implementation plan with: overview, requirements, data flow diagrams (ASCII), edge cases, test matrix
- Does NOT make code edits (read-only tools only)
- Handoff → `/implementer` agent

**Tools:** `search/codebase`, `search/usages`, `web/fetch`  
**Model:** Claude Opus or GPT-5.2  
**Handoffs:** → Implementer agent

---

### `/review`
**Role:** Staff Engineer  
**When:** Before any PR or after feature branch is done  
**What it does:**
- Reviews diff vs main for logic bugs, edge cases, missing error handling
- Auto-flags issues with severity (CRITICAL / WARN / NOTE)
- Does NOT auto-fix — reports only, developer approves
- Generates a review summary

**Tools:** `search/codebase`, `search/usages`, `read/terminalLastCommand`  
**Model:** Claude Sonnet or GPT-5.2

---

### `/qa`
**Role:** QA Lead  
**When:** After implementation, before ship  
**What it does:**
- Runs existing test suite via terminal
- Identifies untested paths from coverage report
- Writes missing tests
- Verifies fixes

**Tools:** `terminal`, `edit`, `search/codebase`  
**Model:** GPT-5.2 or Claude Sonnet

---

### `/ship`
**Role:** Release Engineer  
**When:** When feature is done and reviewed  
**What it does:**
- Syncs with main
- Runs tests (fails if tests fail)
- Audits test coverage delta
- Commits with conventional commit message
- Opens GitHub PR with auto-generated description

**Tools:** `terminal`, `edit`, `github`  
**Model:** GPT-5.2

---

### `/retro`
**Role:** Engineering Manager  
**When:** Weekly  
**What it does:**
- Reads recent git log and PR history
- Summarizes: commits, lines changed, test health, open PRs
- Identifies shipping streaks, stale branches, coverage trends

**Tools:** `terminal`, `search/codebase`  
**Model:** GPT-4o or Haiku (fast/cheap)

---

### `/document`
**Role:** Technical Writer  
**When:** After `/ship`  
**What it does:**
- Diffs what changed vs current docs
- Updates README, ARCHITECTURE.md, and any stale inline docs
- Commits doc changes

**Tools:** `edit`, `search/codebase`, `terminal`  
**Model:** GPT-5.2

---

### `/investigate`
**Role:** Debugger  
**When:** Facing a bug with unknown root cause  
**What it does:**
- Forces systematic investigation before any fix
- Traces data flow, surfaces hypotheses, tests each one
- Stops after 3 failed fix attempts and escalates
- Iron Law: no fix without root cause

**Tools:** `search/codebase`, `search/usages`, `terminal`, `read/terminalLastCommand`  
**Model:** Claude Opus (deep reasoning)

---

### `/cso`
**Role:** Chief Security Officer  
**When:** Before any release, or on demand  
**What it does:**
- OWASP Top 10 scan
- STRIDE threat model
- Confidence gate: only surfaces 8/10+ confidence findings
- Each finding includes a concrete exploit scenario
- Zero noise policy: false-positive exclusion list

**Tools:** `search/codebase`, `web/fetch`  
**Model:** Claude Opus

---

## Custom Agents

### `planner.agent.md`
Read-only planning persona. Tools: `web/fetch`, `search/codebase`, `search/usages`.  
Handoffs → implementer.

### `reviewer.agent.md`
Code review persona. Tools: `search/codebase`, `search/usages`.  
No edit access. Handoffs → implementer or qa.

### `implementer.agent.md`
Full edit persona. Tools: `edit`, `terminal`, `search/codebase`.  
Handoffs → reviewer, qa.

### `qa.agent.md`
QA persona. Tools: `terminal`, `edit`, `search/codebase`.  
Handoffs → ship.

### `security.agent.md`
Security-only. Tools: `search/codebase`, `web/fetch`.  
No edits. Reports only.

---

## Installation

### Install globally (personal skills)
```bash
git clone --depth 1 https://github.com/dyrota/cstack.git ~/.copilot/skills/cstack
cd ~/.copilot/skills/cstack && ./setup
```

### Install to a project
```bash
git clone --depth 1 https://github.com/dyrota/cstack.git .github/skills/cstack
cd .github/skills/cstack && ./setup --local
```

The `setup` script:
- Copies agents to `.github/agents/`
- Copies prompts to `.github/prompts/`
- Optionally appends a cstack block to `.github/copilot-instructions.md`

---

## Workflow Example

```
/office-hours         ← reframe the problem
/plan                 ← generate implementation plan
                      ← [implement via Copilot agent mode]
/review               ← staff engineer review
/qa                   ← run tests, fix gaps
/ship                 ← commit + open PR
/document             ← update docs
/retro                ← weekly reflection
```

---

## Tech Stack

- **Format:** Markdown (SKILL.md, .agent.md, .prompt.md)
- **Standard:** [agentskills.io](https://agentskills.io) open standard
- **Install:** Bash setup script
- **Language:** None (pure Markdown config, no runtime)
- **License:** MIT

---

## MVP Scope (v0.1)

- [ ] `skills/plan/SKILL.md`
- [ ] `skills/review/SKILL.md`
- [ ] `skills/qa/SKILL.md`
- [ ] `skills/ship/SKILL.md`
- [ ] `skills/investigate/SKILL.md`
- [ ] `agents/planner.agent.md`
- [ ] `agents/reviewer.agent.md`
- [ ] `agents/implementer.agent.md`
- [ ] `agents/qa.agent.md`
- [ ] `setup` script
- [ ] `README.md`
- [ ] GitHub repo public at `dyrota/cstack`

## Post-MVP

- [ ] `skills/office-hours/SKILL.md`
- [ ] `skills/cso/SKILL.md`
- [ ] `skills/retro/SKILL.md`
- [ ] `skills/document/SKILL.md`
- [ ] `agents/security.agent.md`
- [ ] VS Code Marketplace extension (contributes skills via `chatSkills` contribution point)
- [ ] `prompts/` directory with reusable prompt files
- [ ] `/create-cstack` command that generates customization files with AI
