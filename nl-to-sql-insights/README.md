# Natural-Language-to-SQL and Automated Insights Service: A Technical Case Study

A design write-up of a production service I built that lets non-technical business users query a relational database in plain English and get plain-language insights from tabular data.

> **About this document.** This write-up describes architecture, design decisions and lessons learned. The organization, product names, hostnames, database schema, table and column names, and all business data have been removed or replaced with generic examples. Code snippets are simplified illustrations written for this document, not the original source.

---

## Contents

1. [Overview](#1-overview)
2. [Tech stack](#2-tech-stack)
3. [System architecture](#3-system-architecture)
4. [Part 1: Natural language to SQL](#4-part-1-natural-language-to-sql)
5. [Part 2: Automated insight generation](#5-part-2-automated-insight-generation)
6. [Cross-cutting concerns](#6-cross-cutting-concerns)
7. [Lessons learned](#7-lessons-learned)
8. [Scaling and future work](#8-scaling-and-future-work)

---

## 1. Overview

### The problem

Business users in a B2B distribution company (sales managers, regional heads) needed answers from a SQL Server database. Typical questions were "total sales by distributor for March" or "top 10 products by quantity last quarter". Each answer meant a report request and a wait for someone who knows SQL.

Several things made automation hard:

- **Legacy schema.** The database had no foreign key constraints and inconsistent naming. Some tables also had malformed column names.
- **Readability.** Users want names ("Acme Traders"), not surrogate keys ("distributor_id 4471").
- **Safety.** An LLM writes SQL that runs against a production database, and that database also holds user credential data.
- **Continuous improvement.** Accuracy had to improve from real usage without letting anyone poison the system with bad "corrections".
- **Scale of insights.** Some reports have hundreds of thousands of rows, far more than fits in an LLM context window.

### What the service does

1. **Natural language to SQL.** It retrieves the relevant tables and similar past examples, generates T-SQL with an LLM, validates it with deterministic code, executes it read-only, and logs every attempt for feedback.
2. **Automated insights.** It turns any table into a short business summary. It can summarize the result of a generated query, or any rows sent by another internal system, such as a reporting module.

### Guiding principle

**Treat every LLM output as untrusted.** The prompt asks the model to behave, and deterministic code enforces the rules regardless of what the model actually does.

---

## 2. Tech stack

| Layer | Technology | Role |
|---|---|---|
| API | FastAPI, Uvicorn | REST endpoints, dependency-injected auth, Pydantic validation, OpenAPI docs |
| LLM orchestration | LangChain (core, community, classic, ollama) | Prompt templates, few-shot example selection, SQL chain, callbacks |
| LLM runtime | Ollama | Serves the generation model and the embedding model |
| LLM | `gpt-oss` family (20B local) | SQL generation and insight narration |
| Embeddings | `nomic-embed-text` (768 dimensions) | Semantic retrieval of tables and examples |
| Vector store | FAISS (flat L2 index, persisted to disk) | Two small indexes: table schemas and example questions |
| Database | Microsoft SQL Server via SQLAlchemy and pyodbc | Business data and the query feedback log |
| Data processing | pandas | Result serialization and dataset aggregation |
| Config | pydantic-settings, INI file | Typed environment settings and a table allowlist |
| Auth | PyJWT (HS256) | Verifying tokens issued by the company's existing SSO |
| Resilience | tenacity | Exponential-backoff retry for transient model-server errors |
| Packaging and hosting | cx_Freeze, IIS with URL Rewrite and ARR | Windows executable behind an IIS reverse proxy |
| Test tooling | Streamlit, a Python checklist runner | Manual test UIs and automated question runs |

---

## 3. System architecture

### 3.1 Style

- **A modular monolith.** It's one stateless REST service with clear internal layers, deployed as a single process behind a reverse proxy.
- **NL-to-SQL uses retrieval plus generation plus a guard.** Retrieve schema and examples, generate SQL, validate it with code, execute it, then log it for human-in-the-loop learning.
- **Insights uses compute-then-narrate.** pandas computes exact aggregates over every row. The LLM only narrates the figures.
- **Offline and online paths are split.** An operator script builds the vector indexes. The API only reads them.

### 3.2 Context diagram

```
 +------------------+   login    +--------------------------+
 | Web app users    |----------->| Company SSO (issues JWT) |
 +--------+---------+            +--------------------------+
          | HTTPS + Bearer token
          v
 +------------------+            +-----------------------------------------------+
 | Reverse proxy    |----------->|  NL-to-SQL & Insights API (FastAPI, stateless) |
 | (IIS + ARR)      |  localhost |  verifies JWT with a shared HMAC secret        |
 +------------------+            +---+------------+-------------+------------+---+
          ^                          |            |             |            |
 +--------+---------+                v            v             v            v
 | Other internal   |        +-----------+  +----------+  +-----------+  +-----------+
 | modules (reports)|        | SQL Server|  | FAISS    |  | Ollama    |  | Email /   |
 | send rows for    |        | data (RO) |  | indexes  |  | LLM +     |  | webhook   |
 | insights         |        | + log (RW)|  | on disk  |  | embeddings|  | alerts    |
 +------------------+        +-----------+  +----^-----+  +-----------+  +-----------+
                                                 |
                                   +-------------+-------------+
                                   | Offline index build script |
                                   +---------------------------+
```

### 3.3 Layers inside the service

```
ENTRY            app factory, lifespan (create log table, background model warm-up), CORS, routers
API              query router (/query, /feedback, /review, /query/{id}/insights)
                 insights router (/insights); router-level auth dependency; error-to-status mapping
SERVICES         query orchestration, SQL validator + executor, insights summarizer, alerting
KNOWLEDGE        SQL prompt, insights prompts, verified join graph, sensitive-column terms, seed examples
INFRASTRUCTURE   settings, DB engine, LLM client, embeddings client, retry policy, JWT verification,
                 vector index reader (online), vector index builder (offline), feedback store
```

The index **writer** and index **reader** share only a paths module, and neither imports the other. It's a small CQRS-style split. The standalone insights path has **no database dependency**, so it keeps working during a database outage.

**Patterns used:**

| Pattern | Where | Why |
|---|---|---|
| Lazy singleton (`functools.lru_cache`) | LLM client, embeddings client, vector indexes, prompts | One instance per process, created on first use |
| Adapter / decorator | Retrying embeddings wrapper, custom example selector | Add retry to a third-party client; plug FAISS into LangChain's selector interface |
| Dependency injection | Router-level auth dependency | Cross-cutting auth with no handler changes |
| Pipeline | LangChain Expression Language chain | Composable prompt, model and parser steps |
| State machine | Feedback and review status columns | Explicit lifecycle for corrections |
| Fail closed | Empty auth secret or CORS origin list | Misconfiguration denies access instead of opening it |

### 3.4 Deployment

```
Windows Server
  IIS application
    URL Rewrite "(.*)" -> http://localhost:<port>/{R:1}   (ARR reverse proxy)
    request body limit raised for large report payloads
    ARR proxy timeout raised above the slowest insights call
  Frozen executable (cx_Freeze) running Uvicorn on a local port
    working directory holds settings files and the vector index folder
  Ollama daemon (local, or a shared instance behind its own proxy)
SQL Server on a separate host
```

### 3.5 Request flow: a question

```
client -> proxy -> API: verify JWT
  embed question -> FAISS schema index -> top-5 tables
  add lookup tables from the verified join graph (deterministic)
  embed question -> FAISS examples index -> top-3 similar question/SQL pairs
  prompt = rules + narrowed table definitions + join graph + examples + question
  LLM -> SQL text
  validator: single read-only SELECT/WITH, no blocked keywords, no sensitive columns
  execute with pandas on SQLAlchemy -> JSON (table orient, keeps column types)
  insert a row in the feedback log -> return {id, sql, result}
```

### 3.6 Offline versus online paths

| | Offline (index build) | Online (serving) |
|---|---|---|
| Trigger | Operator script after schema or example changes | Every request |
| Reads | Live schema, seed examples, reviewer-approved corrections | Vector indexes, business data |
| Writes | Vector index files, "promoted" flags | Feedback log rows |
| Model calls | One embedding per table or example | Two embeddings and one generation per question |
| Why split | Slow and has side effects | Must stay fast and read-only for the indexes |

---

## 4. Part 1: Natural language to SQL

### 4.1 Table retrieval with a vector index

Sending the whole schema in every prompt wastes tokens, slows inference and distracts the model with irrelevant columns.

- **Offline:** one document per allowlisted table, containing its `CREATE TABLE` definition. The documents are embedded and saved as a FAISS flat L2 index. Exact search is fine at a dozen vectors; approximate indexes only pay off at much larger scale.
- **Online:** the question is embedded and the five nearest tables are returned. The index is loaded once and cached. If it's missing, the service tells the operator to run the build script instead of building it during a request.
- **Design note:** LangChain's `SQLDatabase` includes a few sample rows per table in its schema text by default. Sample rows help the model with value formats. They also send real data to the model and into the persisted index. For tables with sensitive data, set the sample-row count to zero or filter columns before rendering.

### 4.2 Deterministic expansion with lookup tables

Results had to show names, not IDs. A prompt rule and a matching few-shot example still produced raw IDs.

**Root cause:** for a question like "show 10 sales records", vector search never ranked the lookup tables (distributors, customers, items) in the top five. The model never saw their columns, so it couldn't join to them, whatever the prompt said.

**Fix:** after retrieval, a deterministic step walks a hand-maintained relationship graph. For every selected table with a known display-name relationship, it adds the table that holds the name column.

```python
def expand_with_lookup_tables(selected: list[str], relationships: list[dict]) -> list[str]:
    result = set(selected)
    lowered = {t.lower() for t in selected}
    for rel in relationships:
        source_table = rel["left"].split(".")[0]
        if source_table.lower() in lowered and rel.get("display_name"):
            result.add(rel["display_name"].split(".")[0])
    return list(result)
```

**Lesson:** check what actually reaches the model's context before blaming the model.

### 4.3 A verified join graph, because there were no foreign keys

With no foreign keys, the schema text never shows how tables join. As more tables were added, the model began guessing joins from similar column names. One example was joining `distributor.id = region.id` instead of `distributor.region_id = region.id`.

The fix was a relationship graph in code. Each entry records:
- the two columns;
- whether the join was **verified against real data** (overlap of sampled distinct values) or only inferred from naming;
- a short note;
- the display-name column on the lookup side, when one is confirmed.

The graph is rendered into the prompt with explicit markers such as `[unverified]` and `[no confirmed display-name column - leave the raw id]`. The prompt says to use only these joins and to prefer an existing denormalized column over guessing a join. Saying "there is no name column" explicitly stops the model from inventing one.

### 4.4 Prompt design

A `FewShotPromptTemplate` with these parts, in order:

1. **Role:** "You are a Microsoft SQL Server expert. Return only safe SELECT statements."
2. **Security rule:** never select, filter by or reference a column whose name contains a sensitive term, even if asked. Return a query over the non-sensitive columns instead, and return SQL only.
3. **The narrowed schema** for the selected tables.
4. **The verified join graph** and "only join using these relationships".
5. **Readability rule:** when an ID column has a display-name relationship, join and select the name unless the user asked for the ID.
6. **T-SQL rules:** `TOP N` instead of `LIMIT`, square brackets for unusual names, `ISNULL`, T-SQL date functions, `GROUP BY` with aggregates, `ORDER BY` with `TOP`.
7. **Pattern hints**, such as "top 5 X" maps to `SELECT TOP 5 ... ORDER BY ... DESC`.
8. **Three dynamically selected examples**, then the question.

**Dynamic few-shot selection.** The examples index embeds only the example **question**; the SQL is stored as metadata. A custom `BaseExampleSelector` returns the three most similar pairs. Three relevant examples beat a dozen fixed ones: fewer tokens, better relevance, and the set can grow without making the prompt longer.

**`create_sql_query_chain` details worth knowing:**
- It requires the prompt to declare `input`, `top_k` and `table_info`.
- The `table_names_to_use` input restricts the schema text to the pre-selected tables.
- Its `k` argument means "rows per SELECT", not "number of tables". Keep the two settings separate, or the prompt will tell the model to return at most five rows.

**Model settings:**

| Setting | Value | Reason |
|---|---|---|
| Temperature | 0.2 | Near-deterministic SQL |
| Context window (`num_ctx`) | 16,384 tokens | Prompt and response share it (see 5.6) |
| Max output (`num_predict`) | 2,500 tokens | Bounds latency and runaway generation |
| Reasoning mode | Off | Faster. The task doesn't need extended reasoning |
| Keep-alive | 30 minutes | Avoids cold model loads |

### 4.5 The SQL validator

The first version was a keyword blacklist using substring matching. It missed `EXEC`, stacked statements and comment-hidden payloads, and it false-positived on columns like `UpdatedDate`. It was replaced with an **allowlist**:

```python
COMMENTS = re.compile(r"--.*?$|/\*.*?\*/", re.MULTILINE | re.DOTALL)
STARTS_OK = re.compile(r"^\s*(SELECT|WITH)\b", re.IGNORECASE)
BLOCKED = re.compile(r"\b(INSERT|UPDATE|DELETE|DROP|ALTER|TRUNCATE|MERGE|GRANT|REVOKE|DENY|CREATE|"
                     r"EXEC|EXECUTE|INTO|OPENROWSET|OPENQUERY|OPENDATASOURCE|BACKUP|RESTORE|"
                     r"SHUTDOWN|BULK)\b", re.IGNORECASE)
SENSITIVE = re.compile("|".join(map(re.escape, SENSITIVE_TERMS)), re.IGNORECASE)

def is_safe(sql: str) -> bool:
    if not sql or not sql.strip():
        return False
    body = COMMENTS.sub(" ", sql).strip()
    body = body[:-1] if body.endswith(";") else body   # allow one trailing semicolon
    if ";" in body:                                     # anything else is a stacked statement
        return False
    return bool(STARTS_OK.match(body)) and not BLOCKED.search(body) and not SENSITIVE.search(body)
```

**Why each rule exists:**
- **Comments are stripped first**, so nothing can hide inside them.
- **One statement only.** A semicolon inside a string literal is rejected too. That's acceptable because it fails safe.
- **`INTO` is blocked** because `SELECT ... INTO new_table` is a write disguised as a read.
- **`OPENROWSET`, `OPENQUERY` and `OPENDATASOURCE`** reach external sources.
- **Keywords match whole words** (`\b`), so `UpdatedDate` no longer triggers `UPDATE`.
- **Sensitive terms match as plain substrings**, not whole words. Regex treats letters and underscores as word characters, so `\bpassword\b` misses `PasswordHash` and `pwd_hash`. Short terms that collide with ordinary words were left out (for example `pin` matches `Opinion` and `ShoppingCart`).

**Two real catches:**
- A security test asked for "user logins and passwords". Before the fix, it returned the password column. It was fixed at two layers: a prompt rule and the deterministic column block. Afterwards the model either omitted the column or refused. A refusal isn't SQL, so the validator blocked it either way.
- "Show me everything" made the model emit several stacked `SELECT *` statements. The single-statement rule blocked them.

**Limits of this approach:** regex isn't a SQL parser. The stronger design adds AST validation (for example with `sqlglot`), a statement timeout, a row cap and, as the final guarantee, a **read-only, least-privilege database login**.

### 4.6 Execution and serialization

- The query runs through `pandas.read_sql_query` on a SQLAlchemy engine.
- Results are serialized with `DataFrame.to_json(orient="table")`. That format carries a type schema with the data, so the client or the insights service can restore integer, float and datetime types exactly.
- Blocked or failed queries return a warning message plus the logged attempt id. That way they still appear in the feedback loop.

### 4.7 Feedback, review and learning without fine-tuning

Every attempt is logged with:
- the question, the SQL and the selected tables;
- the model name, success flag, error, row count and execution time;
- feedback and review fields.

```
logged           feedback=pending   review=not_required
user accepts  -> feedback=accepted
user rejects  -> feedback=rejected  review=pending_review  (+ optional suggested SQL, alert sent)
reviewer      -> review=approved with reviewer-confirmed SQL
index job     -> approved pairs embedded into the examples index, rows marked promoted
```

- **Anti-poisoning:** a user's suggested SQL is stored but never trusted. Only reviewer-approved SQL becomes a few-shot example.
- **Learning through retrieval, not training.** Approved pairs are retrieved as examples for similar future questions. That's cheap, immediate, reversible and auditable compared with fine-tuning.
- **Alerts** (email over STARTTLS, Slack-style webhook) are optional and best-effort. A failed alert never fails the feedback request.
- **Database portability gotcha:** SQLAlchemy's `.is_(False)` renders as `IS 0` on SQL Server, which T-SQL rejects. `== False` renders `= 0`, which works. It was caught by running against the real database.

### 4.8 Testing

- **A tiered question checklist of about 85 questions** in markdown: single-table queries, filters, aggregates, dates, two-table and multi-table joins, top-N, grouping, edge cases (non-existent entities, future dates, vague requests) and a security section.
- **A runner script:**
  - parses the checklist and calls the live API;
  - flags only objectively checkable expectations, such as expected-empty results, sensitive columns in the SQL, or a missing row cap on vague questions;
  - writes a timestamped report;
  - in interactive mode, submits real accept/reject feedback through the API.
- **Semantic correctness is judged by a human.** An item is ticked only when the SQL is verified against the schema, not merely because it ran.
- **A typical run:** zero request failures, average round trip about 9 seconds, minimum about 4 seconds. Cold starts with retry backoff took over a minute.

---

## 5. Part 2: Automated insight generation

### 5.1 Problem

A table of 900 rows, or a report of nearly half a million, isn't an answer for a manager. Constraints:
- the data doesn't fit in a context window;
- LLMs are unreliable at arithmetic over long tables;
- other modules need insights on their own data, with no SQL involved;
- long calls must survive a reverse proxy.

**Core idea:** **compute in code, narrate with the LLM.**

### 5.2 Two entry points, one core

```
POST /query/{id}/insights  -> re-run the logged SQL (re-validated) -> DataFrame
POST /insights {data, context} -> row guard -> DataFrame.from_records
                    \                         /
                     summarize_dataframe(df)          (shared, bounded)
                               |
                   question-aware prompt  |  generic prompt with caller context
                               |
                        LLM (retry on transient errors) -> {insights, row_count, generation_time_ms}
```

- **Re-running the SQL instead of storing results** keeps large blobs out of the log table and always reflects current data. The trade-off is extra database load.
- **The standalone endpoint has no database dependency.** A caller sends rows plus optional context ("Monthly sales by distributor group, Jan to Dec").
- **Error semantics:** empty data returns 422, too many rows returns **413** (a dedicated input-error exception), and model or infrastructure failures return 500.

### 5.3 The summarizer

```python
def summarize_dataframe(df) -> str:
    if df.empty:
        return "The dataset has no rows."
    parts = [f"Dataset: {len(df)} rows, {df.shape[1]} columns ({', '.join(df.columns)})."]
    num = df.select_dtypes(include="number")
    if not num.empty:
        parts.append("Numeric stats over ALL rows:\n" + num.agg(["sum", "mean", "min", "max"]).round(2).to_csv())
    for col in df.columns.difference(num.columns):
        counts = df[col].value_counts(dropna=True)
        if len(counts) <= 30:
            parts.append(f"'{col}' full breakdown:\n" + counts.to_csv())
        else:
            top = counts.head(30)
            parts.append(f"'{col}': {len(counts)} distinct; top 30 cover {top.sum()}/{len(df)} rows:\n" + top.to_csv())
    parts.append("Illustrative sample (stats above already cover every row):\n"
                 + truncate_cells(df.head(50), 200).to_csv(index=False))
    text = "\n\n".join(parts)
    return text if len(text) <= 12_000 else text[:12_000] + "\n...(truncated)"
```

**Design choices:**
- **Every row is aggregated.** An early version looked only at the first few hundred rows. The rewrite gives a 100-row and a 100,000-row dataset equally complete statistics.
- **Coverage is explicit.** "Top 30 cover X of Y rows" tells the model and the reader how representative a breakdown is.
- **The sample is labeled as illustration**, so the model doesn't generalize from 50 rows.
- **There are three bounds:** 200 characters per cell, 50 sample rows, and 12,000 characters total. No single dimension (rows, columns, cell length) can blow the prompt. Sections are ordered so truncation removes the sample first.
- **Characters, not tokens.** It's simpler, but numeric CSV tokenizes at roughly two to three characters per token. That fact drove the context-window change in 5.6.

### 5.4 Prompt design and its evolution

The generic prompt went through three versions:

1. **v1:** a short "summarize the key insights" instruction.
2. **v2:** a long rulebook. No jargon, numbers in grasp-able form ("about 1 in 4"), never invent units, hedge causes, a fixed output format.
3. **v3 (current):** shorter and more effective:
   - XML-style `<context>` and `<figures>` tags delimit the inputs;
   - **"The figures are already calculated. Use them as given; you may round, but do no new calculations."**;
   - an exact four-part output: **The big picture**, **What stands out** (3 to 5 bullets, each "finding with a number, then why it matters"), **What you could do next**, and **How much to trust this**;
   - a one-line style example on deliberately different data, which teaches tone without leaking content;
   - only numbers and units present in the figures may be used, and empty data gets a fixed sentence.

**Why shorter worked better:** long rule lists dilute attention and cost tokens on every call. A concrete format and one example steer output more reliably, and moving the arithmetic into code removed the need for many accuracy rules.

The query-linked prompt also includes the user's question and the SQL, so the model knows which filters and groupings produced the figures.

### 5.5 Limits, aligned across layers

| Layer | Limit | If exceeded |
|---|---|---|
| Client | 10-minute timeout | Client error |
| Reverse proxy body size | Raised from the ~30 MB default | Proxy rejects before the app sees it |
| Reverse proxy timeout | Raised above the slowest insights call | Gateway error while the app is still working |
| Request schema | List of objects | 422 |
| Row guard | Configurable (default 1,000,000) | 413 "pre-aggregate or split" |
| Summary | 200 characters per cell, 12,000 total | Truncated with a marker |
| Model | 16k context, 2,500 output tokens | Truncation warning logged |

The row guard and the proxy body limit must be raised together. Raising only one just moves which layer rejects a large request. The row guard is a memory and CPU guard, not a fidelity trade-off, because every row is aggregated anyway.

### 5.6 The context-window lesson

Ollama's `num_ctx` is **shared by prompt and output**. With an 8k window, an insights prompt of about 6,200 tokens left under 2,000 for output. The response was cut off with `done_reason="length"` well below the output cap.

The fix was a 16k window. The bigger fix was **visibility**. `llm.invoke()` discards Ollama's per-call statistics, so:
- direct calls use `.generate()` and log `generation_info` (prompt tokens, output tokens, tokens per second, load time, `done_reason`);
- chain calls use a LangChain callback handler with `on_llm_end`;
- a warning is logged whenever generation stops because of length.

### 5.7 Known limitations of the summarizer

- Sums and means are computed for every numeric column, including ID-like columns and rates, where totals are meaningless.
- Numbers sent as strings are treated as categories.
- Dates are treated as categories instead of ranges or trends.
- There are no top-N or bottom-N rows by value.

Next steps: column-role hints from callers, type coercion, date-aware stats, `nlargest`/`nsmallest`, and deterministic data-quality warnings passed to the prompt.

---

## 6. Cross-cutting concerns

### 6.1 Security model (defense in depth)

```
browser CORS (pinned origins, no credentials)
reverse proxy (TLS, body size limit)
JWT verification (HS256, expiry checked, algorithm pinned)
table allowlist (the model only sees configured tables)
prompt rules (read-only, no sensitive columns, verified joins only)
deterministic validator (single SELECT/WITH, blocked keywords, sensitive-term block)
least-privilege read-only database login   <- the layer that should be the final guarantee
human review before any correction is learned
```

**Authentication is verify-only.** The company's SSO issues tokens and this service only verifies them with a shared secret. There's no second credential store.
- The first version issued its own tokens. It was redesigned once the real integration contract was clear.
- **Gotcha:** PyJWT rejects any token carrying an `aud` claim unless an expected audience is supplied. The SSO's tokens carried a placeholder audience, so audience verification had to be configured explicitly.
- `HTTPBearer` registers a real OpenAPI security scheme, which gives Swagger UI a global **Authorize** button.

**Configuration fails closed.** A missing JWT secret or an empty CORS list denies access instead of opening it.

**Design guidance for similar systems:**
- keep secrets out of version control from day one;
- use an asymmetric signing algorithm (RS256 with JWKS) when the issuer supports it;
- use token claims for role-based access (for example, only reviewers can approve);
- never rely on the prompt for authorization.

### 6.2 Reliability

- **Retry with exponential backoff** (5 attempts, waits of 2, 2, 4 and 8 seconds) on transient model-server errors. Model runners can briefly refuse connections while loading, so retries usually succeed.
- **Background warm-up at startup** in a worker thread, so the server accepts traffic immediately. Plus a 30-minute keep-alive to avoid repeated model loads.
- **Non-fatal startup.** If the database is down, the insights endpoint and health check still work.
- **Best-effort alerts** never break requests.

### 6.3 Concurrency

Handlers are synchronous `def` functions. FastAPI runs them in its threadpool, which is correct because the database driver, pandas and the model client all block. Declaring them `async def` with blocking calls inside would stall the event loop.

### 6.4 Packaging

- **cx_Freeze misses lazy and conditional imports**, for example a LangChain chain module imported only when called, or a pandas optional dependency. The fix was listing whole packages in the build options.
- **`uvicorn.run("main:app", reload=True)`** needs an importable module, which a frozen executable doesn't have. So the app object is passed directly when reload is off.

---

## 7. Lessons learned

1. **Check the context before blaming the model.** The "names, not IDs" bug was a retrieval gap. Prompt changes couldn't fix it.
2. **Prompts shape behavior; code enforces it.** The password case needed both a prompt rule and a deterministic block.
3. **Allowlists beat blacklists** for anything an LLM generates.
4. **Encode domain knowledge explicitly.** A small, data-verified join graph removed a whole class of hallucinated joins.
5. **The LLM should narrate, not calculate.** Exact aggregates in pandas made insights both scalable and accurate.
6. **Make limits visible and aligned.** Context windows, row guards, proxy body limits and proxy timeouts interact. Log `done_reason`.
7. **Test against the real target.** ORM dialect quirks and token-claim quirks only showed up against the real database and the real SSO.
8. **Human-in-the-loop with a review gate** gives continuous improvement without poisoning risk, and the log doubles as an accuracy metric.

---

## 8. Scaling and future work

### 8.1 Target architecture

```
clients -> API gateway / load balancer (TLS, rate limits, auth offload)
        -> N stateless API replicas (containers, health and readiness probes)
             -> Redis: question->SQL cache (keyed by schema version), insights cache (keyed by data hash)
             -> job queue -> insights workers (202 Accepted + polling, or streaming over SSE)
             -> model gateway -> pool of model servers (timeouts, fallback model, token metering)
             -> read replica with a read-only login, statement timeout, row cap
             -> separate log database; shared or versioned vector index
             -> structured logs with request ids, metrics, traces
```

### 8.2 Capacity reasoning

The model is the bottleneck by an order of magnitude. Illustrative math: with four parallel generation slots and about 8 seconds per question, throughput is 4 / 8 = 0.5 questions per second, about 30 per minute. Insights at about 50 seconds each gives roughly 5 per minute. So scaling means more model capacity, caching, and queueing long jobs. Python-level tuning doesn't move the needle.

### 8.3 Roadmap

1. **Safety:** AST-based SQL validation, statement timeout, row cap or pagination, read-only database principal.
2. **Correctness:** an error-repair loop (feed the database error back to the model once), structured output (`{"sql": ...}`), and execution-accuracy evaluation on a golden question set in CI.
3. **Latency:** asynchronous insights jobs or streaming, plus caching.
4. **Conversation:** follow-up questions ("now only for the North region") through question rewriting with session context.
5. **Retrieval quality:** human-written table descriptions, the embedding model's task prefixes, hybrid keyword and vector search.
6. **Access control:** role-based permissions from token claims, and row-level security so users only see their own region's data.
7. **Index lifecycle:** schema fingerprinting to trigger rebuilds automatically, versioned indexes with hot reload, and incremental example updates.
8. **Observability:** structured logging, latency percentiles, token cost per request, and acceptance rate by model version.
