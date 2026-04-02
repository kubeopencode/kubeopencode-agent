# Daily PR Review

Automatically review all open PRs without the `ai-reviewed` label in the kubeopencode/kubeopencode repository.

## Trigger

- **CronTask**: Daily at 7:00 UTC (`deploy/crontask-pr-review.yaml`)
- **Manual**: `run daily PR review`, `review open PRs`

## Phase 1: Discover Open PRs

Query GitHub for open PRs without the `ai-reviewed` label:

```bash
gh pr list --repo kubeopencode/kubeopencode --state open --json number,title,labels \
  --jq '[.[] | select(.labels | map(.name) | index("ai-reviewed") | not)] | .[].number'
```

If no PRs found, exit successfully with "No open PRs without ai-reviewed label found."

## Phase 2: Review Each PR

For each discovered PR:

1. **Fetch PR details:**
   ```bash
   gh pr view ${PR_NUMBER} --repo kubeopencode/kubeopencode --json title,body,files,additions,deletions,commits
   ```

2. **Review the diff:**
   ```bash
   gh pr diff ${PR_NUMBER} --repo kubeopencode/kubeopencode
   ```

3. **Review for:**
   - Code correctness and potential bugs
   - Error handling and edge cases
   - Security vulnerabilities
   - Style violations and consistency
   - Performance concerns
   - Test coverage

4. **Submit review** with inline comments:
   ```bash
   gh api repos/kubeopencode/kubeopencode/pulls/${PR_NUMBER}/reviews \
     -f event="COMMENT" \
     -f body="## Review Summary

   [Your summary here]

   ---
   *Review by kubeopencode-agent*" \
     -f 'comments[0][path]=path/to/file.go' \
     -f 'comments[0][line]=42' \
     -f 'comments[0][body]=Your inline comment'
   ```

   If no issues found, submit a summary-only review (omit `comments` fields).

## Phase 3: Label

After review is submitted, add the label to prevent re-review:

```bash
gh pr edit ${PR_NUMBER} --repo kubeopencode/kubeopencode --add-label "ai-reviewed"
```

## Rules

- Use `COMMENT` event only — do NOT approve or request changes
- Be constructive and specific in feedback
- Suggest code improvements with examples
- Always add the `ai-reviewed` label after review
- Write review comments in English
