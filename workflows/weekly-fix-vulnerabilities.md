# Weekly Fix Vulnerabilities

Check open Dependabot alerts on kubeopencode/kubeopencode and fix them by adding or updating pnpm overrides for vulnerable transitive dependencies.

## Trigger

- **CronTask**: Weekly on Wednesday at 6:00 UTC (`deploy/crontasks/crontask-fix-vulnerabilities.yaml`)
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

For each npm vulnerability:

1. Check which subprojects use the package:
   ```bash
   cd ui && pnpm why <package-name> 2>&1
   cd ../website && pnpm why <package-name> 2>&1
   ```

2. If the package is a transitive dependency, add a pnpm override in the relevant `package.json` under `pnpm.overrides`:
   ```json
   "pnpm": {
     "overrides": {
       "<package-name>": ">=<patched-version>"
     }
   }
   ```

3. If the package is a direct dependency, update the version directly

4. Run `pnpm install --no-frozen-lockfile` in the affected subproject

5. Verify the fix: `pnpm why <package-name>` should show the patched version

## Phase 3: Fix Go Vulnerabilities

For each Go vulnerability:

1. Run `go get <module>@<patched-version>`
2. Run `go mod tidy`
3. Run `go mod vendor` if a vendor directory exists

## Phase 4: Commit and Push

```bash
git add -A
git commit -s -m "fix(deps): resolve Dependabot vulnerabilities

<list each fixed package and CVE>"
git push
```

Push directly to main (no PR needed for dependency overrides).

## Rules

- Only fix vulnerabilities that have a patched version available
- Do NOT modify code logic — only dependency versions and lockfiles
- Verify each fix with `pnpm why` or `go list -m` before committing
- If a fix breaks `pnpm install` or `go mod tidy`, revert that specific change and skip it
