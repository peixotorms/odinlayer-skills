# Build & Deploy Plugin

Language-agnostic build, deploy, and testing workflow rules for Claude Code.

## What's Included

**Skills** (auto-activate based on context):

| Skill | Focus |
|-------|-------|
| `build-deploy` | Makefile detection, permission/ownership checks, temporary file hygiene, test file placement |

## Rules

- **Always use Makefile** if one exists — never bypass with raw build commands
- **Build temps go to `/tmp`** — keep the project directory clean
- **Check permissions before building** — fix ownership to match project majority
- **Temporary tests to `/tmp`** — permanent tests to `tests/` directory (create if needed)

## Installation

```bash
claude plugin marketplace add peixotorms/odinlayer-skills
claude plugin install build-deploy
```
