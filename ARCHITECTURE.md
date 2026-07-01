# Architecture & System Design

A deeper look at how the natural-language to SQL pipeline is built. This is just a
design document.

---

## Design goals

1. **Correctness on a messy, real-world schema** — hundreds of columns, cryptic
   names, values stored as text, and business rules that aren't in the schema.
2. **Safety** — a production database must only ever receive read-only queries,
   and that guarantee cannot depend on the model "being good."
3. **Privacy** — no data or schema leaves the machine.
4. **Transparency** — the user (or a reviewing analyst) can always see and
   verify the exact SQL that ran.
5. **Portability** — develop offline against a snapshot; run in production
   against enterprise databases with a different SQL dialect.

---

## Layered structure

The codebase is organized in clean layers with a one-directional dependency
flow, roughly Model-View-Controller with an AI/data core:

```
┌──────────────────────────────────────────────────────────┐
│  view/         Qt UI definitions, stylesheets, themes      │
│  controller/   wires UI events → the ask() pipeline        │
│  model/        table model + display formatting            │
├──────────────────────────────────────────────────────────┤
│  ai/           the NLP→SQL pipeline (the interesting part)  │
│     ├ agent      LLM agents, system prompts, validators     │
│     ├ context    dialects, business rules, guardrail regex  │
│     ├ schema     live schema reflection & catalog building  │
│     ├ templates  few-shot "golden query" retrieval          │
│     └ models     typed I/O contracts (Pydantic)             │
├──────────────────────────────────────────────────────────┤
│  data access   SQLAlchemy engines, query execution,        │
│                read-only enforcement, credential handling  │
└──────────────────────────────────────────────────────────┘
```

A single async entry point — `ask(question) → (SqlQuery, DataFrame)` — is all
the UI (or a CLI) needs to call. Everything below it is swappable.

---

## The pipeline in detail

### Stage 1 — Table selection (routing)

A wide database won't fit in a prompt, and shouldn't. The first step is a cheap
LLM call that receives a **compact catalog** — every table with just its column
*names*, capped per table — and returns the minimal set of tables needed to
answer the question.

- The catalog is built by reflecting the live database, so it never drifts from
  reality.
- The model's answer is validated against the real table list; anything it
  invents is dropped, and if it returns nothing usable the system falls back to
  a safe default.
- A few deterministic "join-repair" rules run afterward: if the selection
  includes a line-item table but not the order header needed to bridge to the
  customer table, the missing link is added automatically. This encodes join
  knowledge the model shouldn't have to rediscover every time.

**Why it matters:** this stage is the biggest accuracy lever. Reflecting only
the relevant tables keeps the generation prompt small, focused, and far less
likely to grab the wrong column.

### Stage 2 — Context assembly (deterministic)

Once the tables are chosen, the system builds the generation prompt by
reflecting **only those tables** — with a twist: the widest tables are pruned to
an allowlist of columns that actually get queried, so the model isn't buried in
audit/flag/internal fields.

On top of the schema, several context blocks are injected *conditionally*, based
on which tables are in scope:

- **The join graph** — foreign-key relationships the model can't infer from
  column names alone, marked explicitly as "join keys only, never display."
- **Business glossary** — domain terms ("internal" vs. "external" customers,
  what a "fiscal year" means) translated into concrete filters.
- **Mandatory filters** — e.g. "active customers only," always applied even when
  unmentioned, with documented exceptions for pre-filtered views.
- **Live value vocabularies** — for text category columns, the *actual distinct
  values* are queried from the database at request time and handed to the model,
  so "exclude prospects" maps to the exact stored string.
- **Pre-computed date ranges** — when the user names a fiscal year, Python
  computes the exact boundary dates and injects them as literals. LLMs are
  unreliable at date arithmetic; this removes the guesswork.
- **Few-shot examples** — see "The query cookbook" below.

### Stage 3 — SQL generation (typed LLM call)

The generation agent is configured to return a **structured object**, not free
text:

```
SqlQuery = { explanation: str, sql: str }
```

This is enforced with **grammar-constrained decoding** on the model server
(`response_format` / JSON schema), so the output is always a valid object — no
regex-scraping of a chat reply.

Two reliability features wrap the call:

- **Deterministic decoding.** Temperature 0, fixed seed → the same question
  always produces the same SQL. Reproducible and testable.
- **Output validators with retry.** Before the SQL is accepted, validators
  inspect it for known failure modes — for example, selecting an internal
  numeric join key where the user wanted a human-readable identifier, or
  referencing a column on the wrong table. A failed check raises a retry that
  sends a targeted correction message back to the model, which fixes it. A
  small model gets these details wrong often enough that prompt rules alone
  aren't enough; the validator is the safety net.

### Stage 4 — Guardrails (deterministic, non-negotiable)

Independent of anything the model did, the generated SQL passes through a
read-only enforcement gate before it can touch a database:

- Reject anything that isn't a single statement (no stacked queries).
- Reject anything that doesn't start with `SELECT` / `WITH`.
- Reject a blocklist of write/DDL/procedure keywords
  (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `EXEC`, …).
- Inject a **row-count cap** in the correct dialect (`LIMIT n` for
  SQLite/Postgres, `TOP (n)` spliced in for T-SQL).

This prompt sanitization is important and since this is plain deterministic logic, the read-only guarantee holds even if
the model is prompted adversarially or simply misbehaves.

### Stage 5 — Execution & display

The validated SQL runs through SQLAlchemy against the target engine, results
land in a pandas DataFrame, and the DataFrame is shown in a Qt table with
accounting-style formatting (zeros as `–`, negatives in parentheses, thousands
separators), sortable columns, and query history.

---

## Cross-cutting design decisions

### Dialect abstraction

A small `Dialect` value object captures everything that differs between
databases — prompt wording, the reflection schema name, and how to apply a row
limit. The pipeline is written once; SQLite (cache) and MS SQL Server (prod) are
just two `Dialect` instances. This is what makes "develop offline, deploy to
production" a configuration choice rather than a rewrite.

### The query cookbook (few-shot retrieval)

A folder of **parameterized `.sql` files** captures canonical, fully-correct
query patterns — each with the joins, mandatory filters, relationships and casts already
baked in, plus header comments describing the question it answers and matching
keywords.

- At request time, the templates most relevant to the question (by keyword
  overlap) are injected as few-shot examples, with placeholder parameters
  swapped for illustrative literals *only for the prompt*.
- The same templates can be executed **directly** — bypassing the model
  entirely — with safely bound parameters, for known high-frequency questions.

This gives two wins: better generations (the model adapts a proven pattern
instead of deriving joins from scratch) and an escape hatch to skip the model
for common queries.

### Convenience views

For the most common category of question, a database **view** pre-applies the
joins, the active-customer filter, the domain exclusions, and the text→number
casts. The model is steered to query this one flat view with a simple `SELECT`,
which dramatically cuts the room for error. The view's definition lives in a
`.sql` file and is (re)created idempotently at startup, with its exclusion list
generated from a single source of truth shared with the rest of the app.

### Provider-agnostic model layer

Because the local runtime speaks the OpenAI-compatible API and the app is built
on a typed agent framework, switching from the on-device model to a hosted model
is a one-line change. Local is the default (for privacy and cost); cloud is
available for a quality comparison or eventual adoption.

### Secrets handling

Database credentials are pulled from the OS keyring (with an `.env` fallback for
local dev) and wrapped in a typed secret type so the values are **masked in logs
and tracebacks**. Connection strings are assembled at runtime; no credential is
ever committed to source. The packaging config explicitly excludes secrets and
the multi-gigabyte model file from any build artifact.

---

## Data-flow summary

```
question ──▶ [table selector LLM] ──▶ minimal table set
                                          │
             live DB reflection ◀─────────┘
                    │
                    ▼
        [context assembly]  ── schema + join graph + business
                    │           rules + live vocab + date math
                    ▼           + few-shot templates
        [generator LLM] ──▶ {explanation, sql}  (grammar-constrained)
                    │
             [output validators] ──▶ retry on domain mistakes
                    │
             [read-only guardrails] ──▶ reject writes / cap rows
                    │
                    ▼
            [SQLAlchemy execute] ──▶ DataFrame ──▶ Qt table
```
