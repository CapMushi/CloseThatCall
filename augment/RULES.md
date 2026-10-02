# Rules for `augment/`

Read before creating or editing any file here.

- This folder holds optional, JD (Job Description)-specific extensions to a pipeline in `project-stories/` — technology a particular interview emphasizes that isn't part of that project's core, deterministic story. This keeps the base story universal and reusable across interviews, while letting prep go deep on whatever stack one specific JD calls out.
- One subfolder per augment topic, named after the technology/theme (e.g. `snowflake-cortex-genai/`), not after the company or the JD.
- Every augment file opens with which project-story and which layer/file it attaches to, and why it attaches there (for example: "Attaches on top of the Gold layer in `project-stories/fintech/01-pipeline-overview.md`, giving finance and risk natural-language access to the same certified tables"). Never assume the reader already has the base pipeline in mind.
- Split a large topic into multiple files the same way `project-stories/` splits by pipeline layer — one file per major capability, numbered in a logical reading order — rather than one giant file. Start with a `00-overview.md` per subfolder: the two-sentence framing, the attachment point, a short diagram, and a roadmap of the files that follow.
- Written the same first-person, decision-focused way as `project-stories/`: what was chosen, why, trade-offs — as if this capability had genuinely been built on top of that pipeline for one engagement.
- Audience and jargon rules follow `tech-stack/RULES.md`: explain every term plainly on first use, even ones that feel obvious — this is usually newer territory than the base pipeline. Expand every acronym in brackets on first use per file.
- Be concrete: name real product features (for example Cortex Analyst, Cortex Search), not generic placeholders like "an AI layer." Give at least one worked example per major concept.
- Every number or illustrative claim is marked "(confirm)" until matched to real experience — same convention as `project-stories/fintech/RULES.md`.
- Diagrams use Mermaid, fenced directly in the markdown file. Render and visually check any diagram (scratchpad, `@mermaid-js/mermaid-cli`) before calling the file done, same as `project-stories/fintech/RULES.md`.
- Never edit any file under `project-stories/` for an augment, not even a short pointer line. The base pipeline stays completely untouched and stack-agnostic; the connection is documented only from the augment side (stating which project-story and layer it attaches to).
