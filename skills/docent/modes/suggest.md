# Mode: `suggest`

File a feedback issue against `calumjs/docent` from inside the user's
Claude Code session, with their config and plugin version attached. The
goal: make it easier to say "I noticed something" than to not say it.

## Preconditions

- `gh` CLI is authenticated (`gh auth status` succeeds). If not, prompt
  the user to run `gh auth login` before continuing.
- The user explicitly invoked suggest — this mode never runs on
  schedule and never auto-opens issues.

## What feedback goes here

Feedback about **Docent itself** — its prompts, modes, schemas,
defaults, or procedure. Things like "the inaugural post came out
weird," "init Step 5 failed on my harness," "the editorial vibe
looked wrong on my repo."

Feedback about the **user's own project** (its bugs, its features) does
NOT go here. That's what the site's `/report-bug/` and
`/request-feature/` pages are for — they file against the user's own
repo, not against Docent.

If the user's request sounds like project feedback rather than Docent
feedback, redirect them before continuing:

> That sounds like feedback for {their-project}, not for Docent. Open
> the /report-bug/ or /request-feature/ page on your Docent site — it
> opens a pre-filled issue against your repo. Want me to do that
> instead?

## Procedure

### Step 1 — Short structured intake

Ask these in ONE message. Keep it to four questions; longer forms kill
feedback loops harder than no form.

1. **Kind** — enhancement, bug, prompt issue, procedure issue, or
   other? (Pick one.)
2. **Where did you notice it?** — which mode were you running (init,
   update, digest, release, triage, suggest), which file or step was
   involved.
3. **What happened?** What did you see, what did you expect instead,
   and what makes it matter to the maintainer?
4. **One-line title** (optional). If skipped, draft one yourself from
   the answer to #3.

Accept short answers. Users who want to write essays can; users who
want to fire off a one-liner per question should be able to.

### Step 2 — Gather environment

Collect context the maintainer will want when triaging, so they don't
have to ask:

```bash
# Docent config
cat docent.config.json 2>/dev/null

# Plugin version — inspect the installed plugin's plugin.json
cat "${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json" 2>/dev/null

# Repo identity (so maintainer can reproduce against a similar project)
gh repo view --json name,owner,primaryLanguage,repositoryTopics 2>/dev/null
```

If `docent.config.json` doesn't exist (user is filing feedback on the
`init` mode itself before it's run), record that as "no config — init
hadn't been run."

### Step 3 — Compose the issue body

Template:

```markdown
## What kind of feedback

{kind}

## Where

{where — mode, file, step}

## What happened

{what they told you, their words lightly edited for clarity but NOT
rewritten. If they wrote a one-liner, leave it short.}

---

<details>
<summary>Environment (auto-collected, click to expand)</summary>

**Docent plugin version:** {from plugin.json's version field}

**Source repo:** {owner}/{name} ({primaryLanguage}, topics: {topics})

**docent.config.json:**
```json
{contents, or "no config — init hadn't been run"}
```

</details>

---

_Filed via Docent `suggest` mode from {owner}/{name}._
```

### Step 4 — Redact

Before showing the body to the user, run a redaction pass. Replace
matches with `[REDACTED]`:

- GitHub tokens: `gh[oprs]_[A-Za-z0-9]{20,}`
- AWS access keys: `AKIA[0-9A-Z]{16}`
- Generic high-entropy 32+ char hex or base64 runs that appear next to
  a keyword like `token`, `key`, `secret`, `password`, `api_key`
- Private email addresses that show up in git config or commit
  messages inside `docent.config.json` (shouldn't happen, but check)

SKILL.md invariant 3 already forbids writing secrets into generated
*site content*; this step applies the same rule to generated *issue
content*. Same producer, same duty.

If redaction hits on anything, tell the user: "I redacted {N}
potentially sensitive strings before filing. Please review the preview
before confirming." Then show the redacted body.

### Step 5 — Confirm with the user

Show the full composed body and the target:

> I'm about to file this against `calumjs/docent`:
>
> **Title:** {title}
>
> {body}
>
> Confirm to file, or say what to change.

Wait for explicit confirmation. Never call `gh issue create` without
an affirmative response from the user in this turn — no silent
external writes.

If the user requests changes, apply them and re-confirm before filing.

### Step 6 — File

```bash
gh issue create \
  --repo calumjs/docent \
  --title "{title}" \
  --body "$(cat <<'EOF'
{redacted, confirmed body}
EOF
)" \
  --label "{kind-derived-label}"
```

Labels by kind:
- `enhancement` or other → `enhancement`
- `bug` → `bug`
- `prompt issue`, `procedure issue` → `documentation`
- `other` → no label (maintainer triages)

### Step 7 — Report back

Report the issue URL to the user. Thank them briefly and mention that
the maintainer reviews feedback when iterating on Docent's prompts.

## Exit conditions

- Issue filed, URL reported, or
- User declined to confirm — exit without filing. Do not retry
  automatically.

## Error handling

- **`gh auth status` fails**: prompt the user to run `gh auth login`.
- **`gh issue create` fails** (rate limit, network, permissions): show
  the full composed body and the title so the user can file manually
  at https://github.com/calumjs/docent/issues/new.

## Non-goals

- Not a general-purpose bug tracker. Feedback goes to `calumjs/docent`,
  not to the user's own repo.
- Not anonymous. Issues are filed under the authenticated `gh` user,
  same as any other `gh issue create`.
- Not a proxy for iterating on the user's project — use the site's
  `/report-bug/` or `/request-feature/` forms for that.
