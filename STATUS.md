# Docent — build status

**As of this handoff (2026-04-18):** skill, prompts, schemas, and the
Astro site template are all written. Docent has not yet been run against
a real repo — dogfooding on this one is the next step.

## What's done

- [x] `README.md` — project pitch and install instructions
- [x] `SPEC.md` — full technical specification
- [x] `.claude/skills/docent/SKILL.md` — skill entry point
- [x] `.claude/skills/docent/modes/*.md` — all five modes (init, update,
      digest, release, triage)
- [x] `.claude/skills/docent/prompts/*.md` — overview, journal, status,
      theme system prompts; tone presets
- [x] `.claude/skills/docent/schemas/*.json` — config, status,
      frontmatter, theme schemas
- [x] Design inheritance (SPEC §5.5): `theme.json`, four vibe presets
      (editorial/technical/clinical/expressive), logo-analysis flow wired
      into `init` Step 7.5; init-only invariant added to SKILL.md
- [x] `.claude/skills/docent/templates/workflows/docent-deploy.yml` —
      Pages deploy (primary path)
- [x] `.claude/skills/docent/templates/workflows/docent-update.yml` and
      `docent-digest.yml` — optional CI fallbacks (SPEC §6.7)
- [x] SPEC §6 rewritten around Claude Code Routines as the primary
      scheduler; GitHub Actions reduced to the Pages deploy

## What's not done

- [ ] **Dogfood on this repo.** Install the skill into this repo and run
      `init`. Docent should generate a decent site for itself; if it
      can't, that's a signal the prompts or template need work.
- [ ] **Re-analyze-design path.** SPEC §5.5.4 reserves
      "Docent, re-analyze the design" as a future trigger to re-run the
      theme analysis. Not yet wired into any mode file.
- [ ] **Verify the four vibe stylesheets in a browser** across light/dark
      mode. They're written but haven't been rendered.
- [ ] **Package install tested?** No — `npm install` has not been run on
      the template yet. Version pins (`astro@^5`, `@astrojs/mdx@^4`) are
      best-effort; may need to be adjusted once a real install happens.

## How to try the init flow

1. Drop `.claude/skills/docent/` into a test repo (or this repo).
2. Run Claude Code in the repo and say "set up Docent."
3. Watch the `init` mode run end-to-end: config generation, template copy,
   content generation (overview / status / changelog / inaugural journal),
   and theme analysis (Step 7.5).
4. Review the PR it opens.

## Suggested next steps

1. Dogfood on this very repo. If Docent can't generate a decent site for
   Docent, the prompts need work before the template does.
2. Iterate on the `init` prompts based on what the dogfood run produces.
   The theme classifier is the highest-stakes prompt — if it picks a bad
   vibe, the whole site feels wrong.
3. Once the dogfood site looks right, verify all four vibes render well
   by temporarily swapping the `vibe` field in `theme.json`.
