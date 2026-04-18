# Docent

A Claude Code plugin that turns any GitHub repo into a public-facing, self-maintaining website.

Docent reads your code, commits, issues, and releases, and produces a static site that tells your repo's story to non-developer visitors — what the project is, what's been happening, what's broken, what's coming.

The site lives in your repo under `/docs` and deploys to GitHub Pages. No hosted service, no API keys, no backend. Inference runs through your Claude Code subscription via a weekly/daily Routine. Forms submit by opening pre-filled GitHub issues in a new tab.

## What it produces

A live site at `username.github.io/project` with:

- **Overview** — an AI-written explanation of what the project is and who it's for
- **Journal** — blog posts summarizing recent activity on a configurable cadence
- **Status** — open issues grouped and summarized, known bugs, in-progress work
- **Report a bug / Request a feature** — forms that open pre-filled GitHub issues
- **Changelog** — human-readable release notes derived from tags and merged PRs

## Installing it in a repo

Docent is distributed as a Claude Code plugin. In any Claude Code session inside your target repo:

```
/plugin marketplace add calumjs/docent
/plugin install docent@docent
```

Then say:

> **set up Docent for this repo**

The skill runs its `init` procedure — asks about tone, cadence, domain; scaffolds `/docs` from the plugin's bundled Astro template; generates an overview, a status page, a changelog, an inaugural journal post; analyzes your repo for design signals (logo, badges, metadata) and picks one of four visual vibes; opens a pull request against your default branch.

After merging the PR, two manual steps only you can do:

1. **Enable GitHub Pages with Actions as the source**:
   ```
   gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow
   ```
   (Or via the web UI: **Settings → Pages → Source → GitHub Actions**.)

2. **Set up the two Routines** that keep content fresh:
   ```
   /schedule update Docent daily at 10:17 — prompt: "Docent: run update mode."
   /schedule digest Docent weekly on Mondays at 9:17 — prompt: "Docent: run digest mode."
   ```
   Routines use your Claude Code subscription tokens — no `ANTHROPIC_API_KEY` required.

You can also invoke Docent manually anytime: **"Docent, write a journal post about the auth refactor."**

## What's in this repo

```
docent/
├── .claude-plugin/
│   ├── marketplace.json          # marketplace metadata
│   └── plugin.json               # plugin metadata
├── skills/
│   └── docent/                   # the skill itself — what gets installed
│       ├── SKILL.md              # triggering description + mode dispatch
│       ├── modes/                # one file per operation mode
│       ├── prompts/              # prompt fragments for each content type
│       ├── schemas/              # JSON shapes for structured content
│       └── templates/            # Astro site + deploy workflow
├── docs/                         # Docent's own site (dogfood)
├── SPEC.md                       # full technical specification
└── README.md                     # this file
```

## Status

The skill, Astro template, and dogfood site are in place. Docent's own site is live at [calumjs.github.io/docent](https://calumjs.github.io/docent/). See `SPEC.md` for the full design and `STATUS.md` for what's done vs. outstanding.

## License

MIT.
