# Docent — Technical Specification

**Status:** Draft v0.1
**Last updated:** 2026-04-18

## 1. Overview

Docent is a Claude skill that generates and maintains a public-facing static website for a GitHub repository. The site explains what the project is, summarizes recent development activity, surfaces open issues and known bugs in human-readable form, and provides forms for visitors to report bugs or request features (submitted as pre-filled GitHub issues).

The skill is invoked through Claude Code — either locally by a maintainer, or in CI via a scheduled GitHub Action. It reads the repository's state (code, git history, GitHub issues, PRs, releases) and writes structured content files to `/docs/content/`. A vendored Astro site template in `/docs/src/` renders those content files into the deployed site.

### 1.1 Design principles

1. **Zero hosted infrastructure.** No backend to run, no SaaS to subscribe to, no API keys to manage beyond what Claude Code and GitHub already require.
2. **PR-gated by default.** The skill proposes changes by opening pull requests; a human reviews and merges. Nothing AI-generated goes live without human approval.
3. **Structured content, not raw HTML.** The skill writes MDX and JSON conforming to documented schemas; the site template owns all presentation. Theming and generation are independent.
4. **Same-repo, versioned with the code.** `/docs` lives in the project's own repo. Checking out a tag shows the docs at that tag. No drift between deployed site and described codebase.
5. **Claude Code as runtime.** The skill is mostly prompts and conventions. All file I/O, git operations, and GitHub API calls happen through Claude Code's existing tools (`bash`, `gh`, file edits). No custom binaries, no SDK dependencies.
6. **Static-first.** The deployed site is pure static files on GitHub Pages. All "interactive" elements (bug reports, feature requests) are implemented as pre-filled GitHub issue links that open in a new tab.

### 1.2 Non-goals

- Docent is not an API documentation generator. It does not replace Docusaurus, MkDocs, or Sphinx.
- Docent is not a replacement for GitHub Issues. It is a friendlier public surface on top of them.
- Docent is not autonomous. It proposes; the maintainer disposes.
- Docent does not run continuously. It runs on schedule, on demand, or on specific events.

## 2. Architecture

### 2.1 Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         User's repository                        │
│                                                                  │
│  ┌───────────────────┐        ┌──────────────────────────────┐  │
│  │  .claude/skills/  │        │  docs/                       │  │
│  │  docent/          │        │  ├── content/   (AI-owned)   │  │
│  │  (the skill)      │───────▶│  ├── src/       (human-owned)│  │
│  │                   │        │  ├── public/                 │  │
│  └───────────────────┘        │  └── dist/      (gitignored) │  │
│           │                   └──────────────────────────────┘  │
│           │                                   │                  │
│           ▼                                   ▼                  │
│  ┌───────────────────┐        ┌──────────────────────────────┐  │
│  │ docent.config.    │        │ .github/workflows/           │  │
│  │ json              │        │ ├── docent-update.yml        │  │
│  │                   │        │ └── docent-deploy.yml        │  │
│  └───────────────────┘        └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
           │                                   │
           ▼                                   ▼
  ┌───────────────────┐              ┌──────────────────────┐
  │   Claude Code     │              │   GitHub Pages       │
  │   (local or CI)   │              │   (deploy target)    │
  └───────────────────┘              └──────────────────────┘
```

### 2.2 Directory layout in a user's repo

```
user-repo/
├── src/                          # the user's project (untouched by Docent)
├── docs/
│   ├── content/                  # AI-owned; skill writes here
│   │   ├── overview.mdx
│   │   ├── status.json
│   │   ├── changelog.mdx
│   │   └── journal/
│   │       ├── 2026-04-13-weekly.mdx
│   │       └── 2026-04-06-weekly.mdx
│   ├── src/                      # human-owned site template
│   │   ├── layouts/
│   │   ├── pages/
│   │   ├── components/
│   │   └── styles/
│   ├── public/                   # static assets (favicon, images)
│   ├── astro.config.mjs
│   ├── package.json
│   └── .gitignore                # ignores /docs/dist and /docs/node_modules
├── docent.config.json            # project-level Docent settings
├── .claude/skills/docent/        # the skill itself
└── .github/workflows/
    ├── docent-update.yml         # scheduled: runs skill, opens PR
    └── docent-deploy.yml         # on push to main: builds & deploys
```

### 2.3 Separation of concerns

| Path | Owner | Changes when |
|---|---|---|
| `/docs/content/` | Docent (AI) | Repo activity, scheduled runs, manual invocations |
| `/docs/src/` | Human | User customizes theme |
| `/docs/public/` | Human | User adds assets |
| `/docs/astro.config.mjs` | Human (initial scaffold from skill) | Rarely |
| `docent.config.json` | Human | Settings changes |
| `.github/workflows/docent-*.yml` | Human (initial scaffold from skill) | Rarely |

The skill MUST NOT modify files outside `/docs/content/` (except during `init` mode, which scaffolds the entire `/docs` tree and workflow files). This boundary is enforced by convention in each mode's prompt.

## 3. The skill

### 3.1 Skill structure

```
.claude/skills/docent/
├── SKILL.md                      # triggering description + mode dispatch
├── modes/
│   ├── init.md                   # first-run scaffolding
│   ├── update.md                 # regenerate status + overview
│   ├── digest.md                 # write a journal post
│   ├── release.md                # release notes from a tag
│   └── triage.md                 # process new issues
├── prompts/
│   ├── overview-system.md        # how to write an overview
│   ├── journal-system.md         # how to write a journal post
│   ├── status-system.md          # how to summarize issues
│   ├── theme-system.md           # how to pick a vibe + accent color
│   └── tone-presets.md           # tone options for docent.config.json
├── schemas/
│   ├── status.schema.json        # shape of status.json
│   ├── config.schema.json        # shape of docent.config.json
│   ├── theme.schema.json         # shape of theme.json
│   └── frontmatter.schema.json   # MDX frontmatter conventions
└── templates/
    ├── site/                     # Astro template copied into /docs on init
    └── workflows/                # GitHub Actions copied into .github/workflows
```

### 3.2 SKILL.md (triggering description)

`SKILL.md` is Claude's entry point. Its description field must match when users say things like:

- "set up Docent for this repo"
- "update the Docent site"
- "write a journal post"
- "generate release notes for v1.2"
- "triage the new issues"

The body of `SKILL.md` contains:

1. **A high-level summary** of what Docent does.
2. **Mode dispatch**: a table mapping intent signals to mode files.
3. **Invariants**: rules that apply across all modes (PR-gated output, content directory boundaries, etc.).
4. **Configuration reference**: how to read `docent.config.json`.

Each mode file is self-contained: it describes what to read, what to write, what prompts to use, and when to stop.

### 3.3 Modes

#### 3.3.1 `init` — first-run scaffolding

**Trigger:** user says some variant of "set up Docent" in a repo that does not yet have `/docs/content/` or `docent.config.json`.

**Preconditions:** repo has a git history and a remote. (Docent can run in a fresh repo but will produce thin content.)

**Procedure:**

1. Detect repo metadata: owner, name, default branch, primary language, existence of `README.md`, license.
2. Prompt the user for a handful of choices if not inferable from context:
   - Tone preset (default: "neutral")
   - Journal cadence (default: "weekly")
   - Which sections to enable (default: all — overview, journal, status, changelog, bug report, feature request)
   - Custom domain? (default: none)
3. Write `docent.config.json` at the repo root with the user's choices.
4. Copy `templates/site/` into `/docs/`, substituting repo-specific values (name, owner, URL) into `astro.config.mjs` and `package.json`.
5. Copy `templates/workflows/` into `.github/workflows/`.
6. Generate initial content:
   - `overview.mdx` — read `README.md`, top-level code structure, `package.json`/`Cargo.toml`/`pyproject.toml` if present; produce an accessible explanation.
   - `status.json` — read current open issues via `gh issue list`; group and summarize.
   - `changelog.mdx` — read `git tag` output and the commits between tags; produce release notes.
   - `journal/{date}-inaugural.mdx` — a "project so far" post summarizing the repo's history.
7. Open a pull request titled `Docent: initial site scaffold` with all of the above. PR description explains what was created, what the user needs to do next (enable Pages, merge).

**Exit condition:** PR opened, URL returned to user, user given a short checklist of manual steps (enable Pages, merge PR, verify deploy).

#### 3.3.2 `update` — regenerate status and overview

**Trigger:** scheduled run, or user says "update Docent."

**Procedure:**

1. Read `docent.config.json`.
2. Read current repo state: open issues, recent commits on default branch, latest release tag.
3. Regenerate `status.json` from current open issues. If unchanged (hash matches last run), skip.
4. Check if `overview.mdx` should be refreshed. Heuristic: refresh if `README.md` has materially changed since last update, or if the last refresh is older than 90 days.
5. Regenerate `changelog.mdx` if a new tag has appeared since last run.
6. If any content changed, open a PR titled `Docent: weekly update {date}`.
7. If nothing changed, exit silently (in CI) or report "nothing to update" (locally).

#### 3.3.3 `digest` — write a journal post

**Trigger:** scheduled run (if cadence says so), or user says "write a journal post" / "write a digest."

**Procedure:**

1. Determine the time window: from the date of the most recent journal post to now. If this is less than the configured cadence, exit unless user explicitly requested.
2. Collect activity in that window:
   - Merged PRs (with titles, descriptions, labels)
   - Closed issues
   - New tags/releases
   - Commit volume and authorship
3. Identify coherent themes in the activity. The prompt in `journal-system.md` guides the model to group commits into narratives ("the auth refactor," "performance work," "bug sweep") rather than listing every change.
4. Write `journal/{date}-{slug}.mdx` with appropriate frontmatter.
5. Open a PR titled `Docent: journal post — {headline}`.

#### 3.3.4 `release` — release notes for a specific tag

**Trigger:** push of a new tag, or user says "generate release notes for v1.2."

**Procedure:**

1. Identify the tag and the previous tag.
2. Collect all commits and merged PRs between the two tags.
3. Group by conventional-commit prefix if present (`feat`, `fix`, `chore`, etc.), otherwise by heuristic.
4. Write release section to `changelog.mdx` (prepended, above existing entries).
5. Optionally write a journal post announcing the release (if `docent.config.json` has `journal.announceReleases: true`).
6. Open a PR titled `Docent: release notes for {tag}`.

#### 3.3.5 `triage` — process new issues

**Trigger:** user says "triage the new issues" (not typically run on schedule; too opinionated for default behavior).

**Procedure:**

1. Fetch untriaged issues (no labels, or a specific `needs-triage` label).
2. For each issue:
   - Suggest labels based on content.
   - Search existing issues for potential duplicates; flag if found.
   - Draft a clarifying comment if the report lacks reproduction steps or environment info.
3. Produce a summary report. Docent does NOT automatically comment on or label issues — it presents suggestions for the maintainer to apply manually. This is a safety boundary: the skill drafts, the human posts.

### 3.4 Invariants across modes

Every mode's prompt includes the following rules:

1. **Content boundary**: only write to `/docs/content/` (except in `init` mode).
2. **PR-gated**: never push directly to the default branch. Always commit to a branch and open a PR.
3. **Deterministic outputs**: prefer stable ordering (chronological, alphabetical) so diffs are readable.
4. **No secrets in content**: never include API keys, tokens, or private email addresses in generated content, even if they appear in commits or issues.
5. **Attribution**: generated files include a frontmatter field `generatedBy: "docent"` with a timestamp and mode name.
6. **Idempotency**: running the same mode twice with no repo changes should produce no PR.

## 4. Content schemas

### 4.1 `overview.mdx`

```mdx
---
title: "Project Name"
tagline: "one-line description"
generatedBy: "docent"
generatedAt: "2026-04-18T12:00:00Z"
mode: "init"
---

import { FeatureGrid } from '@/components/FeatureGrid.astro';

<!-- free-form MDX body: explanation of what the project is, who it's for, how to get started -->
```

Constraints:
- Frontmatter fields `title`, `tagline`, `generatedBy`, `generatedAt`, `mode` are required.
- Body should be 300–800 words.
- May import components from the site template's `components/` directory.

### 4.2 `status.json`

```json
{
  "generatedAt": "2026-04-18T12:00:00Z",
  "openIssueCount": 23,
  "groups": [
    {
      "label": "Bugs",
      "description": "Known issues affecting current behavior.",
      "issues": [
        {
          "number": 142,
          "title": "Login fails on Safari 17",
          "summary": "Users on Safari 17 see a blank page after submitting login.",
          "url": "https://github.com/owner/repo/issues/142",
          "updatedAt": "2026-04-15T09:00:00Z",
          "labels": ["bug", "browser-safari"]
        }
      ]
    },
    {
      "label": "In progress",
      "description": "Work that is currently being actively worked on.",
      "issues": [ ... ]
    },
    {
      "label": "Feature requests",
      "description": "Suggested improvements from users and contributors.",
      "issues": [ ... ]
    }
  ]
}
```

Constraints:
- Schema defined in `.claude/skills/docent/schemas/status.schema.json`.
- `summary` field is AI-written, kept under 200 characters.
- Issues are grouped by the skill based on labels and content, not by a fixed mapping. The skill may create appropriate group names for a given project.

### 4.3 Journal posts (`journal/{date}-{slug}.mdx`)

```mdx
---
title: "Week of April 13: auth refactor and a pile of bug fixes"
date: "2026-04-13"
summary: "Short subtitle visible in the index."
tags: ["weekly", "auth"]
generatedBy: "docent"
generatedAt: "2026-04-13T09:00:00Z"
mode: "digest"
commitRange: "abc123..def456"
---

<!-- free-form MDX body -->
```

Constraints:
- Filename format: `{ISO date}-{kebab-case-slug}.mdx`.
- `commitRange` field records the git range covered by the post, so subsequent runs know where to pick up.

### 4.4 `changelog.mdx`

Single file, release entries prepended (newest first). Each release entry follows a consistent template:

```mdx
## v1.2.0 — 2026-04-10

_A short human-readable summary of the release._

### Added
- ...

### Changed
- ...

### Fixed
- ...
```

### 4.5 `docent.config.json`

```json
{
  "$schema": "./.claude/skills/docent/schemas/config.schema.json",
  "project": {
    "name": "Docent",
    "owner": "username",
    "repo": "docent",
    "homepage": "https://username.github.io/docent"
  },
  "tone": "neutral",
  "sections": {
    "overview": true,
    "journal": true,
    "status": true,
    "changelog": true,
    "bugReport": true,
    "featureRequest": true
  },
  "journal": {
    "cadence": "weekly",
    "announceReleases": true,
    "minCommitsPerPost": 3
  },
  "status": {
    "groupStrategy": "auto",
    "excludeLabels": ["wontfix", "duplicate"]
  },
  "deploy": {
    "target": "github-pages",
    "customDomain": null
  }
}
```

Valid `tone` values: `"neutral"`, `"formal"`, `"playful"`, `"technical"`. These map to tone-preset prompt fragments in `prompts/tone-presets.md`.

## 5. The site template

### 5.1 Stack

- **Astro** as the static site generator. Chosen because: native MDX support, minimal JS by default, fast builds, mature ecosystem, straightforward to theme.
- **Plain CSS** with custom properties for theming. No Tailwind; the template is small enough that a stylesheet suffices and users can restyle without learning a utility system.
- **No client-side framework** by default. Astro components are server-rendered at build time. A future version may add selective islands (e.g., a search box) but v1 does not.

### 5.2 Pages

| Route | Source | Purpose |
|---|---|---|
| `/` | `overview.mdx` | Landing page |
| `/status` | `status.json` | Current open issues, grouped |
| `/journal` | `journal/*.mdx` | Index of journal posts |
| `/journal/[slug]` | `journal/{slug}.mdx` | Individual journal post |
| `/changelog` | `changelog.mdx` | Release history |
| `/report-bug` | static | Bug report form |
| `/request-feature` | static | Feature request form |

### 5.3 Bug report and feature request forms

Pure static HTML forms with client-side JS that constructs a pre-filled GitHub issue URL and opens it in a new tab. No data is transmitted anywhere except to GitHub when the user clicks the final link.

The form URL format:

```
https://github.com/{owner}/{repo}/issues/new?title={title}&body={body}&labels={labels}
```

This URL is constructed on submit. The user sees GitHub's new-issue page pre-filled with their description; they authenticate via GitHub (their existing session) and submit directly on GitHub. Docent never sees the data.

Form fields:
- Bug report: title, description, steps to reproduce, expected vs. actual behavior, environment
- Feature request: title, description, use case, alternatives considered

### 5.4 Configuration flow

The template reads `docent.config.json` at build time via Astro's content-loading APIs. Section visibility, project name, repo URL, and tone all flow from config into the template. Users should never need to edit the template's source to change these.

### 5.5 Design inheritance from the host repo

A Docent site should feel like it belongs to the project, not like a
generic template with the project's name pasted in. `init` mode analyzes
the host repo for visual signals and writes a theme that the template
consumes at build time.

#### 5.5.1 Signals the skill collects

In priority order — the first signal found wins; later signals fill gaps:

1. **Logo or brand asset.** Glob for `logo.{svg,png,webp}`, `banner.*`,
   `brand.*` in the repo root, `/assets/`, `/.github/`, `/docs/images/`,
   `/public/`. If found, Claude inspects the image and extracts the
   dominant color as the accent. The asset itself is copied to
   `docs/public/` for use as a hero image.
2. **README badges.** Parse shields.io URLs for `color=` or inline hex
   values. Useful as an accent fallback when there's no logo.
3. **Repo metadata.** `gh repo view --json description,repositoryTopics,
   primaryLanguage` — topics like `game-engine` or `machine-learning` and
   the primary language inform the vibe classification.
4. **README voice.** The README's prose density, formality, and domain
   vocabulary inform vibe classification when metadata is ambiguous.

#### 5.5.2 Vibes

The template ships four stylesheets. The skill picks one based on the
signals above, with a one-sentence justification saved for debuggability.

| vibe | typical fit | type | palette character |
|---|---|---|---|
| `editorial` | docs tools, prose-heavy libraries, the default "museum guide" | serif display + sans body | warm neutrals + one accent |
| `technical` | CLIs, systems tools, DX tooling, infrastructure | mono + sans | cool slate/blue + one accent |
| `clinical` | research, ML, data tools, academic projects | sans only | muted, high-whitespace |
| `expressive` | creative tools, games, design libraries | larger type scale | saturated accent |

When signals are ambiguous, the default is `editorial`.

#### 5.5.3 Output: `docs/content/theme.json`

```json
{
  "vibe": "editorial",
  "accent": "#b45309",
  "accentDark": "#f59e0b",
  "heroImage": "/logo.svg",
  "signals": {
    "source": "logo-analysis",
    "reasoning": "Warm serif logo and a README full of prose — editorial fits."
  }
}
```

Schema lives at `.claude/skills/docent/schemas/theme.schema.json`.

- `accent` and `accentDark` are both required; the template uses them for
  light and dark modes respectively. If only one source color is
  available, Claude produces a contrast-adjusted complement for the other.
- `heroImage` is a path under `/docs/public/` or `null` if no brand asset
  was found.
- `signals.source` records where the decision came from: `logo-analysis`,
  `readme-badges`, `metadata-heuristic`, or `default`.
- `signals.reasoning` is one short sentence, saved so a human reviewing
  the generated PR can understand (and override) the choice.

#### 5.5.4 Regeneration policy

Theme is chosen **once, at init**, and never automatically regenerated.
Scheduled `update` and `digest` runs MUST NOT modify `theme.json`. The
justification: visual identity drift is disorienting for return visitors,
and a theme that changes under the maintainer's feet is worse than one
that's occasionally out of date.

If the host repo rebrands, the maintainer explicitly invokes
"Docent, re-analyze the design" — a separate path that re-runs the
analysis step and opens a PR with the new `theme.json`. Out of scope for
v1; add in a later milestone.

#### 5.5.5 Template consumption

`theme.json` exposes CSS custom properties at build time:

```css
:root {
  --accent: var(--theme-accent);
  --accent-dark: var(--theme-accent-dark);
}
```

The vibe field selects which stylesheet `BaseLayout.astro` imports:

```astro
---
import theme from '../../content/theme.json';
---
<style is:global>
  @import `../styles/vibes/${theme.vibe}.css`;
</style>
```

This keeps the template itself generic — every vibe stylesheet targets
the same set of custom properties and semantic class names.

## 6. Scheduling and deployment

Docent has two jobs to automate: **regenerating content** (which needs
Claude) and **deploying the static site** (which doesn't). These use
different mechanisms:

| Job | Mechanism | Why |
|---|---|---|
| Regenerate content (status, overview, changelog, journal) | Claude Code Routines (`/schedule`) | Uses the user's Claude subscription tokens; no API key to manage; runs on Anthropic's infrastructure |
| Build and deploy the Astro site to GitHub Pages | GitHub Actions | Pure static build, no Claude in the loop; native to Pages |

A Docent install ends up with one GitHub Actions file (`docent-deploy.yml`)
and two Routines (`docent-update` and `docent-digest`).

### 6.1 Why Routines, not GitHub Actions, for content

- **No `ANTHROPIC_API_KEY` secret to configure.** The user's Claude Code
  subscription is the auth. Removes the biggest setup friction.
- **Usage comes out of the subscription plan**, not separate API billing.
- **Runs in a clean Anthropic-hosted environment** that clones the repo and
  opens PRs under the user's GitHub identity (established when they
  connected GitHub to Claude Code).
- **Conversational setup** (`/schedule ...`) is faster than editing YAML.

Limitations the design has to live with:

- **Cron-only, no webhooks.** Routines can't fire on GitHub issue events
  directly. The `update` routine polls daily; the tradeoff is a
  status page that lags by up to a day instead of updating within minutes.
- **Daily run-count cap** on Claude Code plans. Two routines per repo is
  well under typical caps even for small plans.
- **Creation is UI/CLI, not programmatic.** The skill can't call a tool to
  create routines. `init` mode outputs the exact `/schedule` commands for
  the user to run manually.

### 6.2 Routine setup (emitted by `init` mode)

At the end of `init` mode, the skill prints:

```
Docent is installed. To finish setup, run these two commands in Claude Code:

  /schedule update Docent daily at 10:17 — prompt: "Docent: run update mode."
  /schedule digest Docent weekly on Mondays at 9:17 — prompt: "Docent: run digest mode."

Or configure them in the Claude Code UI at claude.ai/code/routines.
```

Times use off-minute values (`:17`) to avoid the thundering-herd cost of
everyone's routines firing at `:00`. See `prompts/tone-presets.md` for the
analogous principle applied to content.

### 6.3 Update routine

- **Cadence**: daily, ~10:17 local.
- **Prompt**: `Docent: run update mode.`
- **What it does**: follows `modes/update.md` — regenerates `status.json`
  from current issues, refreshes `overview.mdx` if `README.md` changed or
  the file is >90 days old, and runs `release` mode for any tag newer than
  the most recent changelog entry. Opens a PR if anything changed,
  otherwise exits silently (the idempotency invariant).

Daily is cheap because most days produce no PR. The routine fetches
issues, hashes the result against the prior `status.json`, and bails early
if identical.

### 6.4 Digest routine

- **Cadence**: weekly, Mondays ~09:17 local.
- **Prompt**: `Docent: run digest mode.`
- **What it does**: follows `modes/digest.md` — finds the time window
  since the last journal post, gathers merged PRs and closed issues,
  identifies themes, writes one post, opens a PR.

The weekly cron is fixed; the **effective cadence** is controlled by
`journal.cadence` in `docent.config.json`. If set to `biweekly` or
`monthly`, `digest` mode's time-window check causes off-week runs to exit
silently. Users change cadence by editing config, not the routine.

### 6.5 `docent-deploy.yml` (the one GitHub Actions file)

Deploys the site to GitHub Pages on pushes to main that touch `/docs` or
the config. This is a pure static build — Claude is not involved.

```yaml
name: Docent deploy
on:
  push:
    branches: [main]
    paths: ['docs/**', 'docent.config.json']
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: docs/package-lock.json
      - run: cd docs && npm ci && npm run build
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: docs/dist
      - uses: actions/deploy-pages@v4
```

Uses the modern official-action flow: `configure-pages` +
`upload-pages-artifact` + `deploy-pages`. Requires **Settings → Pages →
Source: GitHub Actions** to be enabled (one-time manual step).

### 6.6 Local dev mode

While iterating on the skill itself, the Routine feedback loop is too slow
— you want to fire the skill against a repo and see results in seconds,
not wait a day. Use either:

- **`/loop /docent update`** — re-runs on each idle tick until cancelled.
  Good for tightening prompts.
- **`CronCreate`** — explicit cron in-session for timing-sensitive tests
  (e.g. "fire every 5 minutes while I fake-push commits").

Neither is a substitute for Routines in production — both require an
active Claude Code session and both auto-expire.

### 6.7 Appendix: GitHub Actions fallback for content

Users who prefer CI over Routines (e.g. organizations with strict
policies about hosted AI runners, or users not on a Claude Code plan) can
use GitHub Actions for content regeneration instead. The skill ships
`templates/workflows/docent-update.yml` and `docent-digest.yml` as
optional fallbacks. They use `anthropics/claude-code-action@v1` and
require an `ANTHROPIC_API_KEY` repository secret. `init` mode does NOT
copy them by default; users opt in by copying them manually from
`.claude/skills/docent/templates/workflows/` into `.github/workflows/`
and configuring the secret.

## 7. Install and first-run experience

### 7.1 Installation

The user installs Docent by copying `.claude/skills/docent/` into their repo. Three supported methods:

1. **Git submodule** (recommended for updates):
   ```bash
   git submodule add https://github.com/USERNAME/docent .claude/skills/docent
   ```
2. **Direct copy** (simplest):
   ```bash
   curl -L https://github.com/USERNAME/docent/archive/main.tar.gz \
     | tar xz --strip-components=1 -C .claude/skills/docent \
       'docent-main/.claude/skills/docent'
   ```
3. **Clone-and-copy** (for contributors):
   ```bash
   git clone https://github.com/USERNAME/docent
   cp -r docent/.claude/skills/docent .claude/skills/
   ```

### 7.2 First run

After installation, the user opens Claude Code in the repo and says "set up Docent." The skill's `init` mode runs as described in §3.3.1.

The PR the skill opens includes a checklist in its description:

> **Next steps** (do these after merging):
> 1. **Settings → Pages → Source → GitHub Actions** — enables the deploy workflow.
> 2. **Merge this PR.** The deploy workflow runs automatically.
> 3. **Set up the Routines** that keep content fresh. In Claude Code, run:
>    ```
>    /schedule update Docent daily at 10:17 — prompt: "Docent: run update mode."
>    /schedule digest Docent weekly on Mondays at 9:17 — prompt: "Docent: run digest mode."
>    ```
>    Or configure them at claude.ai/code/routines.

No `ANTHROPIC_API_KEY` is required — Routines use the user's Claude Code
subscription. The only repo-level configuration is the Pages setting in
step 1.

### 7.3 Ongoing use

- **Automatic**: Routines run on their configured cadence and open PRs.
- **Manual (local)**: open Claude Code in the repo and say "Docent, write
  a journal post about X." The skill runs in `digest` or `update` mode as
  appropriate.
- **Manual (remote)**: `/schedule run update-docent` (or the name given
  at creation) fires the routine ad hoc from any Claude Code session.

## 8. Open design questions

These are known unresolved questions, recorded so they're not forgotten:

1. **Plugin system?** Should third parties be able to add sections (e.g., "contributors," "dependencies," "benchmarks")? v1 says no; revisit after dogfooding.
2. **Multi-language sites?** Translation is possible (regenerate content per locale) but not planned for v1.
3. **Private repos?** Should work identically but requires GitHub Pro for Pages. Should the skill detect and warn?
4. **Monorepos?** Current design assumes one repo = one site. A monorepo with multiple projects might want one site per project or a unified site. Out of scope for v1.
5. **Rich embeds in journal posts?** Screenshots, videos, benchmark graphs. Current design is text-only; attachments would require somewhere to host images (could use the repo's `/docs/public/` but the skill would need to handle that deliberately).
6. **Content migration?** If the schema changes, how do we migrate existing content files? Needs a versioning scheme in frontmatter.
7. **Review comments vs. direct edits?** Should the skill accept PR review comments and iterate, or always start fresh? Probably the latter for simplicity.

## 9. Milestones

- **M0 — Skeleton (this commit)**: directory structure, SKILL.md stub, empty mode files, site template skeleton, workflow templates, this spec.
- **M1 — `init` mode works end-to-end**: someone can install the skill and generate a usable site on a real repo.
- **M2 — `update` and `digest` modes work**: scheduled runs produce useful PRs.
- **M3 — `release` mode works**: tag-triggered release notes.
- **M4 — `triage` mode works**: issue-processing suggestions.
- **M5 — Polish**: site theme refined, tone presets tuned, config validated, errors handled gracefully.
- **M6 — Publish**: public release, install instructions, example sites.

## 10. License

MIT.
