# Rules for files in `tech-stack/`

Applies to `data-engineering-glossary.md` and any other file in this folder. Read before creating or editing.

## Audience

Write for someone who has never heard of the technology and is reading it for the first time. If a beginner would have to Google a word to follow the sentence, explain the word.

## Scope

- Technology only (tools, languages, file formats, architectures, patterns). No SDLC, soft skills, or stakeholder topics; those belong in other folders.
- One entry per technology, placed under the category of its main function. Do not duplicate an entry across categories; cross-reference instead.

## Entry format

```
**Name** (full name if it is an acronym)
What it is: 1-2 plain sentences, everyday analogy welcome.
Why it exists: the problem it solves, in one or two sentences.
How it works: only if essential (see depth rule below).
Example in a DE project: one concrete scenario, told step by step.
```

Optional: `Don't confuse with:` when two tools are commonly mixed up (e.g. Kafka vs Kinesis, Iceberg vs Hudi).

## Writing style

- Plain, simple words. Short sentences. Active voice.
- Define every technical term the first time it appears in an entry, or point to the entry that defines it (e.g. "see Parquet").
- Expand every acronym on first use, even ones that feel obvious: CDC (Change Data Capture), API (Application Programming Interface). Full form goes in brackets right after the acronym.
- Lead with what the tool does for the user, not with its internals.
- Analogies are good when accurate (e.g. "a warehouse is a library for analytics"); never let the analogy replace the real explanation.
- No marketing language, no filler, no unexplained buzzwords.

## Technical depth

- Default is simple. Add depth only where it is absolutely essential: something a reader must know to use the tool correctly, or a concept interviewers commonly probe (e.g. partitioning in Spark, ACID in Iceberg).
- Put depth in a short `Going deeper:` line or bullet list after the simple explanation, so beginners can stop before it.
- Explain depth in the same plain style; a beginner should still follow it.

## Examples

- Exactly one example per entry, concrete and realistic: name the data, the source, the destination, and what the tool did.
- Tell it as a sequence (first, then, finally), not a single dense sentence.
- Keep examples generic to the tool. Do not put employer-specific details here; those belong in `project-stories/`.

## Accuracy

- Do not state features, versions, or limits you are not sure of. Verify or leave it out.
- Keep every claim true for the tool in general, not only for one vendor's edition.

## Editing

- Keep the entry format and category order consistent across the file.
- When adding a new category or tool, update `README.md` in this folder if it changes what the folder contains.
- When editing an entry, re-check that it still meets the audience, style, and example rules above.
