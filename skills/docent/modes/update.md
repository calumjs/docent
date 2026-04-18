# Mode: `update`

Refresh status and overview. Run this on the scheduled workflow or when the
user says "update Docent" / "refresh the site."

## Preconditions

- `docent.config.json` exists. If not, direct the user to `init` mode.
- `/docs/content/` exists.

## Procedure

### Step 1 — Read config

Load `docent.config.json`. Note:
- Which sections are enabled.
- `status.excludeLabels`.
- `tone`.

### Step 2 — Regenerate `status.json`

1. Fetch open issues: `gh issue list --state open --limit 200 --json number,title,body,labels,updatedAt,url,assignees`.
2. Filter out issues with excluded labels.
3. Group issues (same strategy as `init` — by label and content signals).
4. Write `docs/content/status.json`.
5. Compare with previous version. If semantically identical (same issues,
   same groupings, summaries within trivial diff), discard and mark status
   as unchanged.

### Step 3 — Consider refreshing `overview.mdx`

Check the frontmatter of the existing `overview.mdx` for `generatedAt`.
Refresh if:
- `README.md` has been modified since that timestamp (check `git log -1 --format=%cI README.md`), OR
- The file is older than 90 days.

If refreshing: regenerate using the same procedure as `init` Step 7.

### Step 4 — Consider updating `changelog.mdx`

Run `git tag --sort=-creatordate | head -1` to find the most recent tag.
Check whether `changelog.mdx` already has an entry for that tag. If not, and
the tag is newer than the most recent entry in the file, run the `release`
mode procedure for that tag (without opening a separate PR; combine into
this update's PR).

### Step 5 — If nothing changed, exit

If `status.json` was unchanged, `overview.mdx` was not refreshed, and no new
releases were added, do nothing. Report "Docent: nothing to update" and exit.

### Step 6 — Otherwise, commit and open PR

Branch: `docent/update-$(date -u +%Y-%m-%d)`.

PR title: `Docent: update {YYYY-MM-DD}`.

PR body: bullet list of what changed.

```bash
git checkout -b docent/update-$(date -u +%Y-%m-%d)
git add docs/content/
git commit -m "Docent: update status and content"
git push -u origin HEAD
gh pr create --title "Docent: update $(date -u +%Y-%m-%d)" --body "..."
```

## Exit conditions

- PR opened with the content changes, or
- No-op reported if nothing changed.
