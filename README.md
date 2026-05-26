# opsx-superpowers

OpenSpec workflow with built-in TDD + code review gates for Claude Code.

Four-phase development discipline:

```
/opsx:explore <topic>   → discuss → requirements.md (DRAFT → REVIEWED)
/opsx:propose <topic>   → proposal + specs + design + tasks
/opsx:apply <topic>     → TDD execution + code review gates
/opsx:archive <topic>   → archive + capability spec + CLAUDE.md
```

Each phase is a hard boundary. You cannot skip phases.

## Prerequisites

| Prerequisite | Install |
|---|---|
| [superpowers](https://github.com/obra/superpowers) — skills infrastructure | `claude --plugin-url https://github.com/obra/superpowers` |
| [OpenSpec CLI](https://github.com/Fission-AI/OpenSpec) — schema + workflow engine (Node 20.19+) | `npm install -g @fission-ai/openspec@latest` |

## Install

### 1. Install the plugin

```bash
claude --plugin-url https://github.com/austinxyz/opsx-superpowers
```

### 2. Promote the schema (run once, and after each upgrade)

Run `opsx-install` **from your project root** (the directory that contains `.claude/`):

**Mac / Linux / Git Bash:**
```bash
opsx-install
```

**Windows (PowerShell terminal):**

Running `bash` in PowerShell routes through WSL on most Windows machines. If WSL is not set up, you'll see:
```
WSL ERROR: execvpe(/bin/bash) failed: No such file or directory
```

Fix: run the install from the Claude Code prompt using the `!` prefix (which uses Git Bash, not WSL):
```
! bash /c/Users/<you>/projects/opsx-superpowers/bin/opsx-install
```

Or open **Git Bash** and run `opsx-install` from there.

**Verify:** `openspec schemas` should list `superpowers-driven` with `Source: user`.

## New Project Setup

```bash
cd my-project
openspec init --tools none
# Copy the starter config
cp <plugin-cache-dir>/config-template.yaml openspec/config.yaml
# Edit openspec/config.yaml — fill in your project section and context
```

Minimum `openspec/config.yaml`:

```yaml
schema: superpowers-driven

project:
  dev_stack_command: "docker compose up -d"
  test_commands:
    - "pytest"
  design_system: "notion"

context: |
  Tech stack: Python + FastAPI + PostgreSQL
  Tests: pytest from repo root
```

Then start your first change:

```
/opsx:explore my-first-feature
```

## Upgrading

```bash
# Pull latest plugin
claude --plugin-url https://github.com/austinxyz/opsx-superpowers

# Re-promote the schema (required after every upgrade)
opsx-install
```

## What goes in config.yaml

| Key | Purpose | Example |
|---|---|---|
| `project.dev_stack_command` | Bring up local dev environment | `"npm run dev:up"` |
| `project.test_commands` | List of test commands | `["pytest", "npm test"]` |
| `project.e2e_command` | E2E test command (optional) | `"npm run e2e"` |
| `project.design_system` | Design system for UI mocks | `"notion"` or `"linear"` |
| `project.custom_verification_checks` | Project-specific checks in final verification | `["grep -rn 'secret' src/"]` |
| `context` | Project description for Claude | Natural language |
| `rules` | Per-artifact generation rules | See config-template.yaml |

## Detailed workflow reference

See [docs/workflow.md](docs/workflow.md) for:

- When to use OpenSpec (and when not to)
- Phase-by-phase breakdown with inputs, outputs, and anti-patterns
- Project file layout
- Known gotchas (including the `openspec status` false `[ ]` issue)
- Quarterly upstream sync routine

## Does this conflict with official OpenSpec commands?

No. Official OpenSpec uses the `openspec-*` prefix. This plugin uses `opsx:*`. Zero collision.
