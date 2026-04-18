# Docent — build status

**As of 2026-04-18:** skill, Astro template, and the plugin-marketplace
packaging are done. Docent ran on itself end-to-end; the dogfood site is
live at [calumjs.github.io/docent](https://calumjs.github.io/docent/).

## What's done

- [x] `README.md` — project pitch and install instructions (plugin-based)
- [x] `SPEC.md` — full technical specification
- [x] `skills/docent/SKILL.md` — skill entry point
- [x] `skills/docent/modes/*.md` — all five modes (init, update,
      digest, release, triage)
- [x] `skills/docent/prompts/*.md` — overview, journal, status,
      theme system prompts; tone presets
- [x] `skills/docent/schemas/*.json` — config, status, frontmatter,
      theme schemas
- [x] Design inheritance (SPEC §5.5): `theme.json`, four vibe presets
      (editorial / technical / clinical / expressive), logo-analysis
      flow wired into `init` Step 7.5; init-only invariant in SKILL.md
- [x] `skills/docent/templates/workflows/docent-deploy.yml` —
      Pages deploy (primary path)
- [x] `skills/docent/templates/workflows/docent-update.yml` and
      `docent-digest.yml` — optional CI fallbacks (SPEC §6.7)
- [x] SPEC §6 rewritten around Claude Code Routines as the primary
      scheduler; GitHub Actions reduced to the Pages deploy
- [x] **Plugin-marketplace packaging** (SPEC §7.1): `.claude-plugin/`
      metadata, `skills/docent/` at the plugin root, install reduces to
      `/plugin marketplace add calumjs/docent` + `/plugin install docent@docent`
- [x] **Dogfood:** ran `init` on this repo, fixed three real bugs the run
      surfaced (off-by-one import paths in the site template; missing
      `package-lock.json`; main/master substitution in the deploy
      workflow), deployed the site.

## What's not done

- [ ] **Re-analyze-design path.** SPEC §5.5.4 reserves
      "Docent, re-analyze the design" as a future trigger to re-run the
      theme analysis. Not yet wired into any mode file.
- [ ] **Verify the four vibe stylesheets in a browser** across
      light/dark mode. Only `editorial` has been rendered.
- [ ] **Commit-hook for content refresh.** Routines are daily; a README
      or tag push doesn't reflect in the site until the next polling
      run. Could add a small Action that fires the update routine via
      the `/fire` API — needs a PAT.
- [ ] **Re-install test on a second repo.** The dogfood repo was
      bootstrapped from a raw skill drop, not from
      `/plugin install docent@docent` — so the marketplace path hasn't
      been exercised yet end-to-end.

## How to install

In any Claude Code session inside your target repo:

```
/plugin marketplace add calumjs/docent
/plugin install docent@docent
```

Then say "set up Docent for this repo." The `init` mode runs end-to-end:
config generation, template copy via `${CLAUDE_PLUGIN_ROOT}`, content
generation (overview / status / changelog / inaugural journal), theme
analysis (Step 7.5), and opens a PR.

After merging, enable Pages and set up the two routines — details in the
PR description.

## Suggested next steps

1. Install Docent on a second repo via the marketplace and verify the
   `${CLAUDE_PLUGIN_ROOT}` template copy works as expected end-to-end.
2. Close the commit-hook gap — small Action that fires the update
   routine on `README.md` pushes and `v*` tag pushes.
3. Render the three unused vibe stylesheets in the browser by temporarily
   editing `docs/content/theme.json`. Find anything that breaks.
4. Tag v0.1.0 once the above feels stable.
