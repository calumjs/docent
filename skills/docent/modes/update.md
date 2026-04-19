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
hand since Docent last wrote it. **Two signals must BOTH agree that the
file is machine-owned** — a single spoofable signal is not enough:

**Signal 1: working tree clean + no non-Docent commits touching the file.**

```bash
git diff --quiet HEAD -- docs/content/{file}             # 1 = dirty
git log --pretty=format:"%s" -- docs/content/{file} \
  | grep -v "^Docent:" | head -1                         # any human commit
```

Skip regeneration if either the working tree is dirty or ANY commit in
the file's history has a subject that does NOT start with `Docent:`. We
check the full history, not just the last commit — a single human edit
anywhere in the past permanently moves the file to "co-owned."

**Signal 2: body hash matches what Docent wrote.**

Compute the SHA-256 of the file body (everything after the closing `---`
of the frontmatter; or the whole file for JSON). Compare to the
recorded `bodyHash` (MDX) or `sourceSnapshot.issueSetHash`-validated
content (status.json).

For MDX files:
```bash
awk 'BEGIN{n=0} /^---$/{n++; next} n>=2 {print}' docs/content/overview.mdx | \
  sha256sum | awk '{print $1}'
```

If the current body hash differs from the recorded `bodyHash`, the file
was edited — skip regardless of what the commit log says. This catches
the case a human accidentally (or deliberately) used a `Docent:` commit
subject: the hash won't match.

**Both signals must pass** for machine-ownership. Either failing → skip.
Defense in depth: commits can be subject-spoofed, but you can't spoof
the hash unless you carefully re-hash and rewrite the frontmatter,
which is well past "accidental."

### Step 2b — Regeneration writes fresh anchors

When Docent regenerates a file, it must:
1. Write the new body.
2. Compute the body's SHA-256.
3. Write the full file with the hash in frontmatter (`bodyHash` for MDX,
   inline in JSON's `sourceSnapshot`). This means writing twice or
   buffering — a minor implementation cost for a load-bearing check.

### Step 3 — Check `status.json` freshness

Read `docs/content/status.json`'s `sourceSnapshot.issueSetHash`. Fetch
the current state from GitHub:

```bash
gh issue list --state open --limit 200 \
  --json number,title,body,labels,updatedAt,url,assignees,state
```

Filter out excluded labels. Then compute the current issue-set hash:

1. For each remaining issue, produce a tuple
   `{number, updatedAt, state, labels[sorted]}`.
2. Sort the tuple list by `number`.
3. Serialize as canonical JSON (UTF-8, sorted keys, no whitespace).
4. SHA-256 of the serialization.

Compare to `sourceSnapshot.issueSetHash`. **Regenerate if and only if
the hashes differ.** Do NOT gate on `openIssueCount` and
`newestIssueUpdatedAt` alone — those can collide across different issue
sets (one issue closing while another opens with no newer timestamp
leaves both unchanged but the groupings stale).

The hash is the authoritative signal. The `openIssueCount` top-level
field is retained for the page template and humans, not for freshness
detection.

If regenerating, follow `init.md` Step 7's `status.json` procedure and
write a fresh `sourceSnapshot` with the new hash.

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

Full tag-set reconciliation — not just the newest tag. Older tags can
be added mid-history (backported releases, annotated tag fixes), and
we must catch them all or the changelog silently drifts incomplete.

```bash
git tag --sort=-creatordate                 # all tags, newest first
```

Extract the set of tags already represented in `changelog.mdx` (each
entry opens with `## <tag> — <date>` per SPEC §4.4). Let:

- `allTags` = set from `git tag`
- `changelogTags` = set parsed from the file

**Regenerate if `allTags \ changelogTags` is non-empty** — any missing
tag triggers a run.

For each missing tag, run the `release` mode procedure (not opening a
separate PR; fold the entries into this update PR, inserted in correct
chronological position based on tag date).

Update the frontmatter `sourceCommit` to the current short HEAD after
writing. This records the commit Docent reconciled against; a future
run comparing `sourceCommit` to HEAD is a cheap first-pass check
before doing the expensive tag-set diff.

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
