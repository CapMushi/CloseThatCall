# Rules for `project-stories/fintech/`

Read before creating or editing any file here.

- One universal fintech Data Engineering story, told by the lead/senior data engineer who owned the architecture. First person, decision-focused: what was chosen, why, and what the trade-offs were.
- Split by pipeline layer, one file per layer, numbered in pipeline order: 00-problem-statement, 01-pipeline-overview, then one file per layer (ingestion, processing, orchestration, storage/warehouse, serving, monitoring), ending with challenges-tradeoffs. No extra files (no swap-note files, no summaries, no clutter).
- Diagrams use Mermaid, in a fenced ```mermaid code block directly inside the markdown file — never a separate diagram file or an exported image. `01-pipeline-overview.md` holds one diagram of the whole pipeline, start to end, showing only the main building blocks (one box per layer/system) so a reader gets the overall shape before any layer file goes into detail. A layer file may add its own, more detailed diagram of just that layer if it earns its place.
- Never put a node inside a subgraph if another node outside that subgraph sits between it and a fellow member in the actual flow (e.g. Bronze storage → Processing → Silver storage, with Processing outside the storage box) — Mermaid's auto-layout renders that as tangled, overlapping arcs. Keep such nodes as plain, unboxed nodes instead; only group nodes in a subgraph when nothing external needs to sit between them.
- After writing or editing a Mermaid diagram, render it before calling the file done: extract the ```mermaid block, run it through `@mermaid-js/mermaid-cli` (`npx --yes @mermaid-js/mermaid-cli -i diagram.mmd -o diagram.png -b white -s 2`, using the scratchpad directory, not the repo), and actually look at the output image. Parsing without errors is not enough — check the layout is clean, with no crossing/overlapping boxes, before considering it finished.
- Build each layer as a self-contained component with a clear input and output, so a tool can be replaced without rewriting neighbouring files. Do not add "swap notes"; the separation itself is what allows swaps.
- Use mainstream, in-demand tools only. Explain them in plain language, following `tech-stack/RULES.md`.
- The problem must be explainable in two sentences.
- Be concrete, leave no gaps. Never write a generic placeholder like "a lending system", "an ERP", "a CRM", or "some API". Name the real product (for example Mambu, NetSuite, Salesforce, Adyen, Persona) and say what it holds, how data is pulled from it, and roughly how much and how often.
- Name the actual mechanism, not just the category: CDC from the Postgres log, hourly REST API pull, nightly SFTP file, and so on.
- Give a specific example wherever a claim could be vague (for example which systems produce which conflicting revenue figure), and use tables when comparing several items.
- When a detail is missing, choose a realistic named default and mark it "(confirm)". Do not leave it blank or generic.
- Write the full form in brackets the first time any acronym or abbreviation appears, even ones that feel obvious (for example CDC (Change Data Capture), API (Application Programming Interface), SOX (Sarbanes-Oxley Act), PCI-DSS (Payment Card Industry Data Security Standard), KYC (Know Your Customer), AML (Anti-Money Laundering)). Every later use in the same file can stay short.
- Every number and claim is illustrative until the user confirms it against real experience. Mark them "(confirm)". Never present invented specifics as fact.
- Cover compliance and security where relevant (PCI-DSS, SOX, KYC/AML, access control).
