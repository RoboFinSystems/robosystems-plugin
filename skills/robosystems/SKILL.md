---
name: robosystems
description: >-
  Orientation for the RoboSystems MCP server — accounting and financial-reporting
  knowledge graphs (SEC EDGAR XBRL filings, RoboLedger general ledgers, RoboInvestor
  portfolios). Use whenever "RoboSystems", "RoboLedger", "RoboInvestor", the `sec`
  graph, or a `kg…` graph id is mentioned, to understand what the connection is
  scoped to, which tool family answers an intent, and how to explore a graph
  (schema → example queries → GraphQL or Cypher). For SEC company analysis use the
  sec-filing-analysis skill; for a month-end close use roboledger-close.
---

# RoboSystems

RoboSystems turns financial data into knowledge graphs that an agent can query
directly over MCP. Every graph is a LadybugDB property graph carrying XBRL-grade
financial semantics, with a typed GraphQL surface and read-only Cypher on top.

## What you are connected to

One authorization is **one graph**. The consent screen picks it, and the server's
`instructions` field tells you which kind you got:

| Graph | What it is | Tools |
|---|---|---|
| `sec` (shared repository) | Public-company XBRL filings from SEC EDGAR. Read-only. | ~12 analytical tools |
| A RoboLedger graph (`kg…`) | A customer's general ledger: fiscal calendar, journal entries, schedules, reports, forecasts, documents, memory. Reads and role-gated writes. | ~85 tools |
| A RoboInvestor graph | Portfolios, securities, positions alongside the ledger. | ledger set + portfolio tools |
| A subgraph (`kg…_name`) | A workspace hanging off a parent graph; writable Cypher lives here. | parent's set, no memory |

To work on a different graph, reconnect and pick it — there is no switch tool.
Shared repositories never accept writes; on a customer's graph, writes are limited
by the role the user already holds.

Connection is OAuth 2.1 at `https://api.robosystems.ai/v1/mcp`; the client runs
the consent flow the first time a tool is called. An account with no graph yet is
told to create one at robosystems.ai first.

## Which tool family, given an intent

- **A company's financial statements** → `financial-statement-analysis` (SEC:
  by ticker; a ledger graph after materialization: by report). Current books on
  a ledger graph → `live-financial-statement`.
- **Compare specific metrics across periods or companies** → `build-fact-grid`.
- **Find the XBRL element for a concept** ("revenue", "total debt") → `resolve-element`.
- **Narrative: MD&A, risk factors, policies, the tenant's own procedure docs** →
  `search-documents`, then `get-document-section` / `get-document`.
- **Typed reads of ledger data** (fiscal calendar, entries, agents, mappings) →
  `query-graphql` after `get-graphql-schema`.
- **Anything the typed surface doesn't cover** → `read-graph-cypher` after
  `get-graph-schema` and `get-example-queries`.
- **Month-end close** → call `get-close-playbook` first, then follow the
  roboledger-close skill.
- **Chart-of-accounts mapping** → `get-unmapped-elements`, `suggest-mapping`,
  `list-mapping-structures`, `get-mapping-summary`.
- **Durable notes across sessions** → `recall` before re-deriving context;
  `remember` to persist a fact or decision; `forget` to retire one.

## Exploring a graph

1. `get-graph-schema` — node labels, relationship types, properties.
2. `get-example-queries` — working Cypher for *this* graph's schema. Copy and
   adapt; don't write a traversal from scratch.
3. `query-graphql` for typed reads; `read-graph-cypher` for raw traversal.

Cypher rules that matter on these graphs:

- `read-graph-cypher` is read-only; `CREATE`/`SET`/`DELETE`/`MERGE` are refused.
  `write-graph-cypher` exists only on writable subgraphs, never a parent graph or
  a shared repository.
- Put every relationship from one node in a **single** `MATCH` with comma-separated
  patterns. Chained `MATCH` clauses on the same node can time out.
- **Anchor on the selective node first** (an `Entity`, a `Report`) and reach broad
  nodes like `Structure` last — tens of thousands of filings share a
  `Structure.canonical_type`, and leading with it scans them all.
- Consolidated totals are `Fact {has_dimensions: false}`; segment and geography
  breakdowns are `has_dimensions: true`.
- `Period.period_type` is `instant` / `duration` / `forever`; for durations,
  `duration_type` is `quarterly` / `semi_annual` / `nine_months` / `annual` / `other`.
  `Element.period_type` is the *expected* type for that concept — a different thing.

## Costs and limits

Database reads — Cypher, GraphQL, schema, search, statements, fact grids — consume
no credits. Only server-side AI operations do. Shared repositories carry per-plan
rate limits on MCP calls; batch your questions rather than polling.
