# Forbidden English

This file lists writing habits that Ceili does not use. It supports the "Writing
Standards" section of `CLAUDE.md`, and it governs every English output: chat
replies, code comments, commit messages, user-facing strings, and Markdown docs.

Maintain this file. When a new habit appears, add it here with a bad example and a
fix.

## The standard

Write in ASD-STE100 Simplified Technical English:

- One word carries one meaning. Use the same word for the same thing every time.
- Write short sentences. Put one idea in each sentence.
- Use active voice and present tense where they fit.
- Keep the articles `the` and `a`.
- Choose a plain, common word over a rare or decorative one.

Follow Zinsser's four principles:

1. Simplicity: remove every word that does no work.
2. Brevity: say it in fewer words.
3. Clarity: make the meaning plain on the first read.
4. Humanity: keep a warm, readable voice. Plain writing is still good writing.

## Banned constructions

Each entry has a definition, the reason it is banned, a real bad example from the
docs, and a fix.

### 1. Staccato pairs

A staccato pair is two or three very short clauses or fragments placed side by side
for rhythm or punch. It often uses sentence fragments.

Reason: it reads as a slogan. It adds beat, not information.

- Bad: "No click. No leaving the page."
- Fix: "The page shows the clip inline, so you do not click or navigate away."

### 2. Antithesis reframe (negative parallelism)

This is the "not X, but Y" or "it is not A; it is B" shape. It sets up a foil so
the real point sounds profound.

Reason: the foil is filler. State the point directly.

- Bad: "This isn't documentation written for the AI. It's documentation that
  happens to be in a file the AI reads automatically."
- Fix: "This documentation is for human contributors. The AI reads it too, because
  it sits in a file the AI loads."
- Bad: "The AI is at its best not when it is trusted, but when it is instrumented
  to distrust itself."
- Fix: "The AI works best when it checks its own work."

### 3. Isocolon metaphor-pairs

This is two clauses of equal length and parallel structure, often metaphorical,
used as a closing flourish.

Reason: the symmetry decorates the sentence. It hides the plain fact.

- Bad: "A pink and black checkerboard is a bug report. An invisible surface is an
  afternoon."
- Fix: "The checkerboard shows a bad material name at once. An invisible surface
  would hide the error and cost debugging time."
- Bad: "Claude Code builds the engine; the engine ships with Claude inside it."
- Fix: "Claude Code helps build the engine, and the engine ships with an embedded
  Claude agent."

### 4. Backward-references

A backward-reference points at earlier text instead of naming the thing. Examples:
"the former", "the latter", "as noted above", "recall that", "as we saw", "that is
the whole design in one sentence".

Reason: it makes the reader look back. Name the thing where you use it.

- Bad: "the former O(selected) gizmo walk"
- Fix: "the O(selected) gizmo walk"
- Bad: "That is the whole design in one sentence."
- Fix: Delete the meta-comment. Let the design statement stand on its own.

### 5. Superlative self-ranking

A superlative self-ranking claims that one feature, subsystem, or decision is the
best, the most important, or the standout in the engine. It forces emphasis onto
one topic at the expense of the others, dates quickly, and reads as marketing. The
same applies to a negative superlative (the most questioned, the worst, the
hardest): it still ranks one thing against all the others without support.

Present each subject on its own technical merits and let the reader judge how it
compares. A specific, sourced comparison is acceptable, such as a measured number
or a named trade-off. Avoid an unsupported claim that one feature outranks the
rest.

- Bad: "This is the most distinctive single feature in the engine, so it is worth
  seeing in full."
- Fix: "It is worth seeing one in full."
- Bad: "This is the single most questioned decision, so here is the trade-off."
- Fix: "This decision gets questioned often, so here is the trade-off."
- Bad: "Materials | The standout: materials authored as Lua programs ..."
- Fix: "Materials | Materials authored as Lua programs ..."

## How to apply

- Read a sentence back and ask what each word does. If a word does nothing, cut it.
- If a sentence sounds clever, check it against the four constructions above.
- Fix violations as you touch a file. Do not run a separate cleanup pass unless
  asked.
