# Tone presets

Docent supports four tones, configurable via `tone` in `docent.config.json`.
Apply the matching guidance when writing any content.

## `neutral` (default)

Plain, descriptive, past tense for retrospective content. Minimal
adjectives. No exclamation marks. Reads like competent documentation.

Example opening: "The auth module was rewritten to support passkeys."

## `formal`

Third-person, publication-style. Slightly more elevated vocabulary. Longer
sentences are OK if they're well-constructed. Suitable for projects that
want to read like a company release or an academic tool.

Example opening: "The authentication module has been substantially
restructured to introduce native passkey support."

## `playful`

First-person plural ("we shipped"), light wordplay allowed, willing to
admit mistakes openly ("we broke login for a full day on Tuesday and
everyone was very annoyed"). Avoid forced jokes. Humor should come from
specifics, not from the voice straining to be funny.

Example opening: "We finally got passkeys working, only 11 months after
saying we would."

## `technical`

Assume the reader is a developer. Include file paths, PR numbers inline
(not just as links), module names. Short, dense sentences. Still
accessible — this isn't a release notes dump, it's a readable summary
for someone who cares about implementation.

Example opening: "`src/auth/` was rewritten around the WebAuthn API.
Session cookies are gone; tokens now live in IndexedDB."

## Common rules (all tones)

- No marketing filler in any tone.
- No "simply," "just," or "easily" describing features.
- No superlatives without support. "Fastest" requires a benchmark link.
- No fictional testimonials or user quotes.
