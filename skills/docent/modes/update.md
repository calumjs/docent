# Mode: `update`

Refresh status, overview, and changelog when their sources have changed.
Runs on a schedule (typically daily) or when the user says "update
Docent" / "refresh the site."

## Preconditions

- `docent.config.json` exists. If not, direct the user to `init` mode.
- `/docs/content/` exists.

## Procedure

### Step 1 — Read config

Load `docent.config.json`. Note:
- Which sections are enabled.
- `status.excludeLabels`.
- `tone`.

### Step 2 — Decide what, if anything, to regenerate

Each content file records a **source anchor** in its frontmatter /
top-level JSON (see `init.md` Step 7 and `schemas/frontmatter.schema.json`).
Compare the recorded anchor to the current repo state; only regenerate
files whose anchors are stale.

Before regenerating any file, check whether the file has been edited by
hand since Docent last wrote it:

```bash
git diff --quiet HEAD -- docs/content/{file}    # 1 if dirty
git log -1 --pretty=format:"%s" -- docs/content/{file}   # last commit subject
```

Treat the file as **human-owned** (skip regeneration, even if the
anchor is stale) if either:
- `git diff --quiet` returns non-zero (uncommitted changes present), or
- the last commit touching the file has a subject that does NOT start
  with `Docent:`.

This is the chosen answer to the hand-edit edge case (issue #4). The
alternative — comparing the file's own blob sha to a recorded hash —
requires a hash-of-self field that's awkward to maintain. Git's
authorship record is already the source of truth.

### Step 3 — Check `status.json` freshness

Read `docs/content/status.json`'s `sourceSnapshot`. Fetch the current
state from GitHub:

```bash
gh issue list --state open --limit 200 \
  --json number,title,body,labels,updatedAt,url,assignees
```

Filter out excluded labels. Compute `currentNewestUpdatedAt` (max
`updatedAt` across filtered issues) and `currentOpenIssueCount`.

Regenerate `status.json` if EITHER:
- `currentOpenIssueCount` differs from `sourceSnapshot.openIssueCount`, OR
- `currentNewestUpdatedAt` is strictly later than `sourceSnapshot.newestIssueUpdatedAt`.

Otherwise skip — the page is current.

If regenerating, follow `init.md` Step 7's `status.json` procedure and
write a fresh `sourceSnapshot`.

### Step 4 — Check `overview.mdx` freshness

Read `docs/content/overview.mdx`'s frontmatter. Extract the
`sourceFiles` array.

For each entry, run `git hash-object <path>` and compare to the
recorded `sha`. If any differ, the overview is stale and should be
regenerated.

Also refresh unconditionally if `generatedAt` is older than 90 days —
the README may not have changed but the phrasing should get a fresh
pass occasionally.

If regenerating, follow `init.md` Step 7's `overview.mdx` procedure and
write fresh `sourceFiles` entries.

### Step 5 — Check `changelog.mdx` freshness

Run:

```bash
git tag --sort=-creatordate | head -1       # most recent tag
git rev-parse --short HEAD                  # current HEAD
```

Read `docs/content/changelog.mdx`'s frontmatter `sourceCommit`.

Regenerate (via `release` mode's procedure for the newest tag, combined
into this PR) if either:
- A tag exists that has no entry in `changelog.mdx`, OR
- `sourceCommit` differs from current HEAD AND the newest tag is newer
  than the most recent entry's date.

### Step 6 — If nothing changed, exit

If `status.json`, `overview.mdx`, and `changelog.mdx` were all left
alone, do nothing. Report "Docent: nothing to update" and exit without
opening a PR. This honors SKILL.md invariant 5 (idempotency).

Running `update` immediately after `init` on an unchanged repo MUST be
a no-op PR-wise. The anchors are the mechanism.

### Step 7 — Otherwise, commit and open PR

Branch: `docent/update-$(date -u +%Y-%m-%d)`.

PR title: `Docent: update {YYYY-MM-DD}`.

PR body: bullet list of what changed and why (which anchor drifted).

```bash
git checkout -b docent/update-$(date -u +%Y-%m-%d)
git add docs/content/
git commit -m "Docent: update status and content"
git push -u origin HEAD
gh pr create --title "Docent: update $(date -u +%Y-%m-%d)" --body "..."
```

The `Docent:` commit-subject prefix is load-bearing — Step 2's
hand-edit detection uses it to distinguish machine-authored commits
from human ones. All Docent-authored commits must start with that
prefix.

## Exit conditions

- PR opened with the content changes that were actually stale, or
- No-op reported if everything was fresh.

## Never write to theme.json

`docs/content/theme.json` is init-only (SKILL.md invariant 7). This
mode must not read it, write it, or regenerate it. If a maintainer
wants to re-theme, they run a separate "re-analyze design" path — not
wired into update.
