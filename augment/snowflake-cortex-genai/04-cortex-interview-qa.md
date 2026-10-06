# Cortex Interview Q&A

**Attaches to:** the story in `01`–`03` of this folder, built on `project-stories/fintech/`. This file covers what the JD (Job Description) asks about that the story doesn't use, plus short answers to the questions the story invites. Each answer points back to the story where one exists.

## The 60-second version

"On the fintech platform, Gold in Snowflake already held certified definitions of revenue, loans and customers. I added three things on top with Snowflake Cortex:
- **Cortex Analyst**, over a semantic view, so finance asks questions in plain English and gets the certified numbers.
- **A document pipeline** that parses signed loan agreements with `AI_PARSE_DOCUMENT`, extracts the terms with `AI_EXTRACT`, and flags every loan where Mambu disagrees with the contract. About 0.8% did.
- **Cortex Search** over those agreements, and one **Cortex Agent** that combines both, so a reviewer can ask for numbers and contract wording in a single question.

We're migrating the front end from a Streamlit app to Snowflake CoWork, and every change has to pass an evaluation set before it ships."

## Cortex basics

**What is Snowflake Cortex?**
Snowflake's built-in AI (Artificial Intelligence) features. LLMs (Large Language Models), search, and agents run inside Snowflake, next to the data, under Snowflake's normal permissions.

**Why Cortex instead of an outside LLM API (Application Programming Interface) plus a separate vector database?**
- The data never leaves Snowflake. That matters for our card and KYC (Know Your Customer) data.
- Existing roles and column masks apply automatically to every AI call.
- There's no second system to secure, sync, or pay for.
- The trade-off: you're limited to the models and features Snowflake offers.

**What are Cortex AI Functions?** AI tasks you call inside ordinary SQL (Structured Query Language), like any other function.

| Need (JD wording) | Function |
|---|---|
| General prompt ("Cortex Complete") | `AI_COMPLETE`, formerly `SNOWFLAKE.CORTEX.COMPLETE`; you choose the model |
| Summarization | `AI_SUMMARIZE` (one input), `AI_SUMMARIZE_AGG` (across many rows) |
| Classification | `AI_CLASSIFY` (into categories you define) |
| Translation | `AI_TRANSLATE` |
| Sentiment | `AI_SENTIMENT` |
| Information extraction | `AI_EXTRACT` (fields from text or files), `AI_PARSE_DOCUMENT` (PDF → text) |
| Yes/no filtering | `AI_FILTER` |
| Embeddings / similarity | `AI_EMBED`, `AI_SIMILARITY` |
| Removing personal data | `AI_REDACT` (strips PII (Personally Identifiable Information) from text) |

Story use: `AI_PARSE_DOCUMENT` and `AI_EXTRACT` in `02`; `AI_COMPLETE` in the old Streamlit Loan Review tab.

**What access do users need?**
- **AI Functions:** the `USE AI FUNCTIONS` account privilege plus the `CORTEX_USER` or `AI_FUNCTIONS_USER` database role.
- **Cortex Analyst:** `CORTEX_ANALYST_USER`.
- **Agents:** `CORTEX_AGENT_USER`.
- **Always:** `SELECT` on the underlying tables. Cortex never bypasses normal permissions.

## Cortex Analyst

**What is it?** A service that turns plain English into SQL using a semantic view: a business dictionary of tables, joins, metrics, synonyms, and verified example queries. See `01`.

**How do you improve its accuracy?**
- **Define the business logic:** put metrics in the semantic view as formulas, give columns clear descriptions, and add synonyms for the words people actually type.
- **Pin the joins:** define relationships so Analyst never invents a join.
- **Teach by example:** add verified queries copied from certified dashboards.
- **Measure it:** keep a test set of real questions with known answers and rerun it on every change.
- *Example:* our `active_customers` bug in `01`.

**Why not just ask an LLM to write SQL from table names?** It would guess what "revenue" means and how tables join. That's exactly the three-different-answers problem. The semantic view removes the guessing.

## RAG and Cortex Search

**What is RAG (Retrieval-Augmented Generation)?** Before the model answers, you retrieve the relevant passages from your own documents and tell the model to answer only from them. You get answers grounded in your facts, not the model's memory, and you can cite sources. See search step 4 and Example B in `02`.

**What are embeddings and cosine similarity?**
- **Embedding:** text converted into a list of numbers (768 with Snowflake's default model), so texts with similar meaning get similar lists.
- **Cosine similarity:** measures how closely two of those lists point in the same direction. Near 1 means very similar meaning.
- *Example:* "early payoff fee" and "prepayment penalty" share few words but score high.

**What is high-dimensional indexing?** Comparing a question against millions of 768-number lists one by one is too slow. A vector index groups similar vectors so search checks only the most promising neighbours. That's approximate nearest-neighbour search: a little accuracy traded for a lot of speed. Cortex Search builds and maintains this index for you.

**Which chunking strategies are there?**
- **Fixed size:** cut every N characters. Simple, but it splits clauses.
- **Recursive:** what we used. Split at the most natural boundary first (headings, then paragraphs, then sentences) until pieces fit.
- **Semantic / section-based:** split where the topic changes.
- **The trade-off:** smaller chunks retrieve more precisely, but each piece has less context. Overlap stops a clause being lost at a boundary.
- *Ours:* 1,500 characters with 200 of overlap (`02`). Snowflake suggests at most about 512 tokens per chunk.

**What is hybrid search, and why rerank?**
- **Hybrid search:** keyword search (exact terms, IDs) and vector search (meaning) run together.
- **Reranking:** a second step reorders the combined results by true relevance before the top few are used.
- Cortex Search does both automatically. See `02`.

**What is context-window optimisation?**
- **The context window** is the maximum text a model can read in one call.
- **More isn't better:** extra text costs more, runs slower, and can bury the relevant passage.
- **What we do:**
  - send only the top 5 chunks (`max_results: 5`);
  - filter to the named loan first;
  - keep chunks small.

**Cortex Search versus a vector database (Pinecone, pgvector)?** Cortex Search is managed and hybrid by default. It refreshes from a table on a schedule (`TARGET_LAG`) and obeys Snowflake roles. A separate vector database gives more control over indexing, but you have to run the embedding pipeline and access control yourself.

## Agents and CoWork

**What is Cortex Agents?**
A managed agent loop inside Snowflake: plan, use tools, reflect, repeat until done. See `03`.

**What is "Planner–Tool–Executor–Refiner"?**
A common way to describe agent design. It maps to Cortex Agents like this:
- **Planner** = plan.
- **Tool / Executor** = use tools.
- **Refiner** = reflect.
- *Example:* the re-search for `L-2290` in `03`.

**What tools can a Cortex Agent use?**
- Cortex Analyst (structured data).
- Cortex Search (documents).
- Custom tools (your stored procedures or UDFs (User-Defined Functions)).
- Code execution (a Python sandbox; preview as of August 2026).
- Web search.
- Charts.
- MCP (Model Context Protocol, a standard way to connect AI to outside systems such as Jira or Salesforce) connectors.

**What is Snowflake CoWork?**
- **What it is:** Snowflake's ready-made chat app where business users talk to Cortex Agents. It was named Snowflake Intelligence until it was renamed at Snowflake Summit 2026.
- **Use case:** a CFO (Chief Financial Officer) opens CoWork and asks "net revenue in March by product." The agent routes the question to Cortex Analyst and returns a table and chart.
- **What it provides:** agents shared by role, guardrails, and monitoring, with no front-end code to maintain.
- **In the story:** we're migrating from Streamlit to it (`03`).

## Prompt engineering

**What are system prompts, and what's a prompt library?**
- **A system prompt** holds the standing instructions a model always follows. In our agent, that's the `instructions` block in `03`.
- **A prompt library** is reusable, versioned prompt text kept in Git and reviewed like code. Our `AI_EXTRACT` questions in `02` are part of it, and any change reruns the hand-checked set before it ships.

**How do you optimise a prompt?**
- **Be specific:** "nominal annual interest rate, not the APR" fixed a 6% false-mismatch rate (`02`).
- **Constrain the output:** a structured `responseFormat` instead of free text.
- **Use the smallest model that passes the test set,** which cuts cost and wait time.
- **Send fewer tokens:** see context-window optimisation above.

## Evaluation and Responsible AI

**What is LLM-as-a-Judge?**
A second LLM grades the first one's answer against a rubric and gives a score from 0 to 1 with an explanation. It lets you evaluate thousands of answers without a human reading each one. Spot-check the judge against human grades now and then.

**Which metrics matter?**

| Metric | Question it answers |
|---|---|
| Groundedness / faithfulness | Is every statement supported by the retrieved passages? |
| Answer relevance | Does the answer address the question? |
| Context relevance | Were the retrieved passages the right ones? |
| Correctness | Does it match a known right answer (needs ground truth)? |
| Hallucination | Did it state anything with no source? (low groundedness) |
| Toxicity | Is the language harmful or inappropriate? |

**How do you evaluate an agent, not just an answer?**
Snowflake's Cortex Agent evaluations use the GPA (Goal–Plan–Action) framework. They score whether the agent understood the goal, planned sensibly, and executed each tool call correctly, so you see where it failed. See `03`.

**Which Snowflake tools do this?**
- **AI Observability:** built on TruLens, an open-source evaluation library. It runs your app against a test set, stores traces, and computes LLM-as-a-Judge scores. Events are stored in the `AI_OBSERVABILITY_EVENTS` table.
- **Cortex Agent evaluations:** for agents specifically.

**What are Cortex AI Guardrails?**
- **What they do:** runtime protection against prompt injection (instructions hidden in input or in documents the agent reads) and jailbreaks (attempts to get around safety rules).
- **How they're enabled:** account-wide by `ACCOUNTADMIN` via the `AI_SETTINGS` parameter.
- **Where they're logged:** `CORTEX_AI_GUARDRAILS_USAGE_HISTORY`.
- **Status:** generally available for Cortex Agents and CoWork since May 2026.
- **Why it matters for us:** borrower addenda inside loan PDFs (`03`).

**What Responsible AI practices did you apply?**
- **A human decides.** Mismatches go to credit ops; low-confidence extractions go to review.
- **Grounded answers with citations,** never contract terms from memory.
- **Least-privilege access:** contracts are visible only to credit ops and compliance.
- **Logging:** every request's plan and tool calls are logged.
- **`AI_REDACT`** is available wherever free text with personal data has to leave a restricted role.

## Deployment, governance, and cost

**How do you do CI/CD (Continuous Integration / Continuous Deployment) for Snowflake AI?**
- **Everything is code in Git:** the semantic view, the agent specification, the search-service SQL, and the dbt models.
- **Separate environments:** a pipeline deploys to dev, then test, then prod databases.
- **The gate:** the evaluation sets (120 Analyst questions, 60 agent questions) must pass before promotion.
- **Tools:** SQL runs through Snowflake CLI (`snow`), and models through dbt.

**How do you manage model lifecycle?**
- **`orchestration: auto`:** picks up model upgrades automatically, but behaviour can shift. So every model change reruns the evaluation set, and production is watched more closely for a week after.
- **Pinning a model:** more stable, but you have to schedule upgrades yourself.

**What do you monitor?**
- **Usage and cost:** usage views such as `CORTEX_ANALYST_USAGE_HISTORY` and `CORTEX_AI_GUARDRAILS_USAGE_HISTORY`.
- **Agent behaviour:** agent request monitoring (plan, tool calls, timing).
- **Answer quality:** scores in AI Observability.
- **Weekly review:** the slowest and failed requests.

**What drives cost, and how do you control it?**

| Piece | Billed by | Our control |
|---|---|---|
| Cortex Analyst | Per successful question (tokens when used inside an agent) | Verified queries cut retries |
| AI_PARSE_DOCUMENT / AI_EXTRACT | Pages / tokens | Incremental models: each PDF processed once |
| Cortex Search | Embedding tokens + indexed data size + refresh warehouse | Incremental refresh, small chunks |
| Agents | Orchestration and tool tokens | `max_results: 5`, focused instructions |
| SQL execution | Warehouse time | Extra-small warehouse, 60-second auto-suspend |

## Technical skills in the JD

- **Snowpark:** write DataFrame code in Python (a table-like object you filter and join in code) that runs inside Snowflake instead of on your laptop.
- **Python UDFs (User-Defined Functions):** your own Python function, callable from SQL like a built-in one. A stored procedure can also become a custom agent tool.
- **Streamlit:** a Python library for simple web apps. It was our first front end (`01`, `03`).

## Sources

- [Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [CREATE AGENT](https://docs.snowflake.com/en/sql-reference/sql/create-agent)
- [Use Cortex Search with Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-agents)
- [Build agents for Snowflake CoWork](https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork/build-agents)
- [Streamlit in Snowflake: migrating between runtime environments](https://docs.snowflake.com/en/developer-guide/streamlit/migrations-and-upgrades/runtime-migration)
- [Snowflake CoWork explained (Atlan)](https://atlan.com/know/snowflake/snowflake-cowork/)
- [Cortex Analyst](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [CREATE SEMANTIC VIEW](https://docs.snowflake.com/en/sql-reference/sql/create-semantic-view)
- [Cortex Search overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
- [Cortex AI Functions](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql)
- [AI_EXTRACT](https://docs.snowflake.com/en/sql-reference/functions/ai_extract)
- [AI_PARSE_DOCUMENT](https://docs.snowflake.com/en/sql-reference/functions/ai_parse_document)
- [SPLIT_TEXT_RECURSIVE_CHARACTER](https://docs.snowflake.com/en/sql-reference/functions/split_text_recursive_character-snowflake-cortex)
- [Cortex AI Guardrails](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-ai-guardrails)
- [Evaluate AI applications (AI Observability)](https://docs.snowflake.com/en/user-guide/snowflake-cortex/ai-observability/evaluate-ai-applications)
- [Cortex Agent evaluations](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-evaluations)
