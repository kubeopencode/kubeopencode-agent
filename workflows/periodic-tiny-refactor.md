# Periodic Tiny Refactor

Identify and implement **ONE small, safe refactoring** in the kubeopencode codebase that improves code quality without changing behavior.

## Trigger

- **CronTask**: Every 3 days at 8:00 UTC (`deploy/crontask-tiny-refactor.yaml`)
- **Manual**: `run tiny refactor`, `find a refactoring opportunity`

## Phase 1: Analyze

Use static analysis tools to discover refactoring opportunities:

```bash
cd kubeopencode

# Find unused code, shadowed variables, etc.
go vet ./...

# Find dead code, unused parameters (if available)
staticcheck ./... 2>/dev/null || true

# Check for inefficient code patterns
golangci-lint run --no-config --disable-all \
  --enable=unused,deadcode,ineffassign,unconvert \
  ./... 2>/dev/null || true
```

If tools report issues, prioritize fixing those first (they are verified safe).

### Refactoring Priority Order

Choose the FIRST applicable type:

1. **Dead Code Removal** (safest) — unused imports, variables, unexported functions, commented-out code blocks
2. **Naming Improvements** — single-letter vars in long functions, misleading names, inconsistent patterns
3. **Magic Values** — numbers appearing 2+ times, repeated strings 3+ times, unexplained timeouts
4. **Code Simplification** — complex booleans, deep nesting (>3 levels), long `if err != nil` chains
5. **Small Extractions** — functions >50 lines, duplicate fragments >8 lines

### File Focus

Prefer (in order): `internal/controller/`, `cmd/kubeopencode/`, `pkg/`

Skip: `api/` (public API), `zz_generated*`, `vendor/`, `testdata/`, `e2e/`, `hack/`, YAML/JSON/TOML, files modified in last 5 commits

## Phase 2: Implement

1. Create branch:
   ```bash
   git checkout -b refactor/$(date +%Y%m%d)-<short-description>
   ```
2. Make the minimal change required
3. Self-check: Does this change behavior? Is this the smallest possible change?

## Phase 3: Verify

Run in order — stop on first failure:

```bash
go build ./...
make lint
make test
```

**On failure**: rollback (`git checkout -- . && git clean -fd`), try a different refactoring or exit cleanly.

**Do NOT attempt to fix failing tests** — that changes behavior.

## Phase 4: PR

```bash
git add -A
git commit -s -m "refactor(<scope>): <what changed>

<why this improves the code - one line>

Type: <Priority N: Category Name>"

git push -u origin HEAD
gh pr create \
  --title "refactor(<scope>): <what changed>" \
  --body "**Type:** <Priority N: Category Name>

**Change:** <one sentence>

**Why:** <one sentence>

**Verified:** go build, make lint, make test all pass

---
_Automated by kubeopencode-agent_" \
  --label "refactor" \
  --label "automated"
```

## No Refactoring Found

If no safe refactoring is found:
1. Do not force a change
2. Do not create an issue
3. Exit cleanly — this is normal for a well-maintained codebase

## Safety Rules (Non-Negotiable)

**Absolute Prohibitions:**
- Never change exported function signatures or struct field names
- Never change error messages that callers might parse
- Never reorder struct fields or rename packages
- Never modify externally-referenced constants

**Size Limits:** max 3 files, 50 lines, 5 functions

**Uncertainty = Stop:** If unsure whether a change is safe, choose a different refactoring or exit.

**Verification is mandatory:** Never skip `go build`, `make lint`, or `make test`.
