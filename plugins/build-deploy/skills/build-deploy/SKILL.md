---
name: build-deploy
description: Use when building, deploying, restarting, rebuilding, running make, generating build artifacts, creating test files, or any build/deploy workflow. Also use when the user says "deploy", "rebuild", "restart", or "make".
---

# Build & Deploy

## Build & Deploy Workflow

### Always Check for a Makefile First

Before running any build or deploy command, check if a `Makefile` (or `makefile`, `GNUmakefile`) exists in the project root.

```bash
# Check for Makefile
ls -la Makefile makefile GNUmakefile 2>/dev/null
```

| Situation | Action |
|-----------|--------|
| Makefile exists with relevant target | **Use it** — `make deploy`, `make build`, `make restart`, etc. |
| Makefile exists but no matching target | List targets (`make help` or `grep '^[a-zA-Z]' Makefile`), pick closest match |
| No Makefile | Fall back to language-specific commands (`cargo build`, `go build`, etc.) |

**Never bypass the Makefile** with raw build commands unless explicitly asked. The Makefile encodes project-specific build steps (embedding assets, setting flags, post-build hooks) that raw commands miss.

### Check Permissions & Ownership Before Building

Before any build or deploy, verify file permissions and ownership are correct:

```bash
# Check ownership — identify the majority owner/group
stat -c '%U:%G' * | sort | uniq -c | sort -rn | head -5

# Check for permission anomalies
find . -maxdepth 2 -not -user $(stat -c '%U' .) -o -not -group $(stat -c '%G' .) 2>/dev/null
```

| Rule | Detail |
|------|--------|
| Match the majority | Files should match the dominant owner:group in the project |
| Fix before building | Wrong ownership causes permission errors in build artifacts |
| Respect existing patterns | If specific files have different ownership for a reason (e.g., root-owned configs), leave them |
| Check after deploy too | Deployed artifacts should have correct permissions for the runtime user |

```bash
# Fix ownership to match project majority
chown -R <user>:<group> .

# Fix typical permission patterns
find . -type f -name '*.sh' -exec chmod +x {} \;   # scripts executable
find . -type d -exec chmod 755 {} \;                 # directories traversable
```

### Temporary Build Files

**Always use `/tmp` for build intermediates** — never pollute the project directory.

| File type | Location | Why |
|-----------|----------|-----|
| Build intermediates | `/tmp` | Keep project dir clean, avoid accidental commits |
| Generated artifacts (final) | Project's output dir (`dist/`, `target/`, `build/`) | Part of the build output |
| Downloaded dependencies | Language default (`~/.cargo`, `~/go/pkg`, `node_modules/`) | Standard locations |

```bash
# Use /tmp for scratch work
TMPDIR=$(mktemp -d /tmp/build-XXXXXX)
# ... build steps using $TMPDIR ...
rm -rf "$TMPDIR"   # cleanup
```

**Exception:** If project instructions (CLAUDE.md, Makefile, README) specify a different temp location, follow those.

---

## Test File Placement

### Temporary Tests

Tests created for one-off debugging, quick verification, or exploration:

| Rule | Detail |
|------|--------|
| Location | `/tmp` — never in the project |
| Naming | Descriptive: `/tmp/test_auth_flow.py`, `/tmp/verify_endpoint.sh` |
| Cleanup | Delete after use or leave for session — `/tmp` is ephemeral |

### Permanent Tests

Tests meant to stay in the project (unit tests, integration tests, regression tests):

```
# Decision flow:
1. Does the project have an existing tests/ or test/ directory?  → Put tests there
2. Does the language have a convention (e.g., Go _test.go beside source)?  → Follow it
3. Neither exists and no instructions specify?  → Create tests/ at project root
```

| Situation | Action |
|-----------|--------|
| `tests/` or `test/` exists | Place tests there, following existing structure |
| Language convention (Go `_test.go`, Rust `#[cfg(test)]`) | Follow the convention — tests beside source |
| No test dir, no convention specified | Create `tests/` at project root, place all tests inside |
| Specific instructions exist (CLAUDE.md, README) | Follow those instructions |

```bash
# Check for existing test directories
ls -d tests/ test/ spec/ __tests__/ 2>/dev/null

# If nothing exists and permanent tests are needed:
mkdir -p tests
```

**Never mix temporary and permanent tests.** Temporary goes to `/tmp`, permanent goes to the project.
