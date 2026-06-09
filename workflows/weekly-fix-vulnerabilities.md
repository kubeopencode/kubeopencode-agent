# Weekly Fix Vulnerabilities

Check open Dependabot alerts on kubeopencode/kubeopencode and fix them by adding or updating pnpm overrides for vulnerable transitive dependencies.

## Trigger

- **CronTask**: Daily at 6:00 UTC (`deploy/crontasks/crontask-fix-vulnerabilities.yaml`)
- **Manual**: `fix vulnerabilities`, `check dependabot alerts`

## Phase 1: Query Alerts

1. Navigate to the cloned `kubeopencode` directory

2. Query open Dependabot alerts:
   ```bash
   gh api repos/kubeopencode/kubeopencode/dependabot/alerts \
     --jq '.[] | select(.state=="open") | {number, severity: .security_vulnerability.severity, package: .security_vulnerability.package.name, ecosystem: .security_vulnerability.package.ecosystem, patched: .security_vulnerability.first_patched_version.identifier, vulnerable_range: .security_vulnerability.vulnerable_range, summary: .security_advisory.summary}'
   ```

3. If no open alerts, exit successfully with "No open Dependabot vulnerabilities found."

4. Group alerts by ecosystem:
   - **npm** packages: fix via pnpm overrides in `ui/package.json` and/or `website/package.json`
   - **go** packages: fix via `go get` and `go mod tidy`
   - **other**: report but skip

## Phase 2: Fix npm Vulnerabilities

**Memory optimization**: ui and website are independent projects. Process them **serially** and clean up `node_modules` between projects to avoid keeping two dependency trees in memory.

Set memory cap for Node.js before running any pnpm command:
```bash
export NODE_OPTIONS="--max-old-space-size=3072"
```

### Per-project workflow (run for ui first, then website)

1. Check if the package exists in this subproject:
   ```bash
   cd ui  # or cd website
   pnpm why <package-name> 2>&1
   ```
   If not found, skip this project.

2. Add the pnpm override in this project's `package.json`:
   ```json
   "pnpm": {
     "overrides": {
       "<package-name>": ">=<patched-version>"
     }
   }
   ```
   If multiple packages are vulnerable, add **all overrides at once** before running install (avoids repeated lockfile rewrites).

3. Run install with reduced concurrency:
   ```bash
   pnpm install --no-frozen-lockfile --child-concurrency 2
   ```

4. Verify the fix:
   ```bash
   pnpm why <package-name>
   ```

5. **Clean up before switching to the next project**:
   ```bash
   rm -rf node_modules
   cd ..
   ```

## Phase 3: Fix Go Vulnerabilities

For each Go vulnerability:

1. Run `go get <module>@<patched-version>`
2. Run `go mod tidy`
3. Run `go mod vendor` **only if** a `vendor/` directory already exists **and** the module is in the vendor tree:
   ```bash
   if [ -d vendor ] && grep -q "<module>" vendor/modules.txt 2>/dev/null; then
     go mod vendor
   fi
   ```
   Skip vendor update if the fix is purely for an unused transitive dependency — CI will catch it.

## Phase 4: Commit and Create PR

```bash
git add -A
git commit -s -m "fix(deps): resolve Dependabot vulnerabilities

<list each fixed package and CVE>"
git push -u origin <branch-name>
```

Create a pull request:
```bash
gh pr create --title "fix(deps): resolve Dependabot vulnerabilities" \
  --body "<list each fixed package and patched version>"
```

## Rules

- Only fix vulnerabilities that have a patched version available
- Do NOT modify code logic — only dependency versions and lockfiles
- **Batch npm overrides**: collect all npm vulnerabilities first, add ALL overrides to `package.json` in one edit, then run `pnpm install` once. Avoid rewriting the lockfile multiple times.
- **Process projects serially**: finish `ui` completely (including `rm -rf node_modules`) before touching `website`
- Verify each fix with `pnpm why` or `go list -m` before committing
- If a fix breaks `pnpm install` or `go mod tidy`, revert that specific change and skip it
