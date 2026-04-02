# Weekly OpenCode Update

Check if there is a new release of OpenCode and update the Dockerfile version if a newer version is available.

## Trigger

- **CronTask**: Weekly Monday at 9:00 UTC (`deploy/crontask-opencode-update.yaml`)
- **Manual**: `check opencode update`, `update opencode version`

## Phase 1: Check Version

1. Navigate to the cloned `kubeopencode` directory

2. Read current version from `agents/opencode/Dockerfile`:
   ```bash
   grep 'ARG OPENCODE_VERSION=' agents/opencode/Dockerfile
   ```

3. Fetch latest release from GitHub:
   ```bash
   curl -s https://api.github.com/repos/anomalyco/opencode/releases/latest | jq -r '.tag_name'
   ```

4. Compare versions (strip `v` prefix from GitHub version):
   - If same → exit successfully with "OpenCode is already up to date"
   - If newer → proceed to Phase 2

## Phase 2: Update

1. Update the `ARG OPENCODE_VERSION=x.y.z` line in `agents/opencode/Dockerfile`
2. No other changes to the file

## Phase 3: PR

```bash
git checkout -b chore/bump-opencode-<version>
git add agents/opencode/Dockerfile
git commit -s -m "chore(agents): bump opencode version to <version>"
git push -u origin HEAD
gh pr create \
  --title "chore(agents): bump opencode version to <version>" \
  --body "Bumps OpenCode from <old-version> to <new-version>.

Release: https://github.com/anomalyco/opencode/releases/tag/v<new-version>

---
_Automated by kubeopencode-agent_"
```

## Rules

- Only update if there is actually a newer version
- Do not make any other changes to the file
- Use the exact version number from the GitHub release (without `v` prefix)
