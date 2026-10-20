# Rules for `first-round-story/`

Read before creating or editing any file here.

- **One content file only:** `overall-pipeline.md`, the whole pipeline from start to end, told as one story for a first-round interview. No other content files.
- **Same problem, added to only where it's natural.** Use the same company and problem statement as `project-stories/fintech/00-problem-statement.md`. Additions must grow out of problems already there (for example, "late data" becoming real-time monitoring). Never force in an unrelated domain (for example, IoT sensors in a fintech).
- **Swap only the JD's major technologies.** Replace a tool only when the JD (Job Description) names it, or when a tool needs the same cloud to make sense. Borrow everything else from `project-stories/fintech/`, and keep its order: sources → ingestion → Bronze/Silver/Gold → serving, with orchestration and governance alongside.
- **Never edit any file under `project-stories/` or `augment/`** for this story.
- **Audience:** zero to medium technical knowledge. Use simple, conversational English that the user can understand and repeat out loud. Explain every term the first time it appears, and expand every acronym in brackets on first use.
- **Be concrete:** use real product names, one worked example per major idea, and tables when comparing.
- Every invented number or incident is marked "(confirm)".
- Diagrams use Mermaid, fenced in the file. Render and look at them before calling the file done (scratchpad, `@mermaid-js/mermaid-cli`).
- **Keep code minimal:** only a few lines where they make an idea concrete.
