# Docent

A Claude skill that turns any GitHub repo into a public-facing, self-maintaining website.

Docent reads your code, commits, issues, and releases, and produces a static site that tells your repo's story to non-developer visitors — what the project is, what's been happening, what's broken, what's coming.

The site lives in your repo under `/docs` and deploys to GitHub Pages. There is no hosted service, no API keys, no backend. Inference runs through your local Claude Code or a GitHub Action. Forms submit by opening pre-filled GitHub issues in a new tab.

## What it produces

A live site at `username.github.io/project` with:

- **Overview** — an AI-written explanation of what the project is and who it's for
- **Journal** — blog posts summarizing recent activity on a configurable cadence
- **Status** — open issues grouped and summarized, known bugs, in-progress work
- **Report a bug / Request a feature** — forms that open pre-filled GitHub issues
- **Changelog** — human-readable release notes derived from tags and merged PRs

## Installing it in a repo

Docent is a Claude skill. To use it in your project:

1. Copy `.claude/skills/docent/` into the root of your repo (or symlink it, or submodule it).
2. Open Claude Code in your repo and say: **"set up Docent for this repo."**
3. The skill will scaffold `/docs`, generate initial content from your repo, and open a PR.
4. Review the PR, merge it, then in your repo go to **Settings → Pages → Source: GitHub Actions**.
5. The site will build and deploy automatically.

After that, two Claude Code Routines (set up once via `/schedule`) keep the site current — a daily update run and a weekly journal digest. Both use your Claude Code subscription, so there's no API key to configure. You can also invoke Docent manually any time: **"Docent, write a journal post about the auth refactor."**

## What's in this repo

```
docent/
├── .claude/skills/docent/        # the skill itself (this is what users install)
│   ├── SKILL.md                  # triggering description + pipeline overview
│   ├── modes/                    # one file per operation mode
│   ├── prompts/                  # prompt fragments for each content type
│   ├── schemas/                  # JSON shapes for structured content
│   └── templates/                # site template + workflows (copied on init)
├── docs/                         # Docent's own site (dogfood)
├── SPEC.md                       # full technical specification
└── README.md                     # this file
```

## Status

Early. The skill scaffold exists; the site template is minimal; modes are partially implemented. See `SPEC.md` for the full design and `.claude/skills/docent/SKILL.md` for what's actually built.

## License

MIT.
