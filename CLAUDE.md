# Ceili Public Documentation

This repository holds the public documentation for the Ceili engine. The engine
source is a separate, private tree. Ground every technical claim in that source;
never commit engine code here.

## Writing standard

These rules apply to every Markdown page and to chat replies about them.

- **Write in ASD-STE100 Simplified Technical English.** One word carries one
  meaning. Short sentences, one idea each. Active voice and present tense where
  they fit. Keep the articles `the` and `a`. Prefer a plain, common word over a
  rare or decorative one.
- **Follow Zinsser's four principles: simplicity, brevity, clarity, humanity.**
  Cut every word that does no work. Say the thing directly. Keep the voice warm
  and readable.
- **Do not use the constructions banned in [forbidden_english.md](forbidden_english.md)**
  -- staccato pairs, antithesis reframe (negative parallelism), isocolon
  metaphor-pairs, backward-references, and superlative self-ranking. That file holds a bad example and a fix
  for each. Read it, and add to it when a new tic appears.

## House style

- **ASCII only. No em-dashes or en-dashes in prose.** Use a colon, a comma,
  parentheses, or two sentences. Use `->` for an arrow. The one exception is
  Mermaid edge syntax (`A -- "x" --> B`), which is not prose.
- **Ground every code fragment in real engine source.** Quote 5 to 20 lines
  exactly. Never invent an API. A quoted source comment stays verbatim, even when
  it breaks the writing standard, because it must match the source.
- **One Mermaid diagram per deep page** at least. GitHub renders Mermaid natively.
- **Cross-link by relative path**, and end each page with a "Next:" line to a
  sibling page and to `Docs/README.md`.
- **Name no competitor engines.** Use archetypes: "the commercial heavyweights",
  "the open-source peers", "the data-oriented ECS stacks".
- **Mark media with an HTML comment** that names the exact screenshot or clip to
  capture: `<!-- MEDIA: ... -->`. Store committed media under `Docs/Media/` (Git
  LFS). Embed a webp or image with an `<img width="...">` tag.

## Publishing

The whole site is one commit titled "Hello World!". To publish a change:

```bash
cd C:/dev/Instinct/ceili_public
git fetch -q origin
# STOP if HEAD != origin/main and inspect -- the owner web-edits on github.com.
git add -A && git commit --amend --no-edit --quiet
git push --force-with-lease origin main
```

Always compare `HEAD` against `origin/main` before you amend. Never use a plain
`--force`.
