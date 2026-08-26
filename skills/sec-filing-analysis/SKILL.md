---
name: sec-filing-analysis
description: >-
  Analyze a public company from its SEC EDGAR XBRL filings on the RoboSystems `sec`
  graph — statements by ticker or CIK, cross-period and cross-company metric
  comparisons, concept-to-element resolution, and full-text search over 10-K/10-Q
  narrative. Use for questions like "show me NVIDIA's income statement", "compare
  gross margin across the last eight quarters", "which filers mention goodwill
  impairment", or any request that names a ticker, CIK, 10-K, 10-Q, or XBRL concept.
---

# SEC filing analysis on the `sec` graph

The `sec` graph is a curated, read-only knowledge graph of public-company XBRL
filings. There is no general ledger here and nothing to write; every tool is an
analytical read. Rate limits apply per plan, so prefer one well-chosen call over
many exploratory ones.

## The ladder — cheapest tool that answers

1. **`financial-statement-analysis`** — a whole statement for one company.
   `ticker` is required on `sec`; `statement_type` is `income_statement`,
   `balance_sheet`, `cash_flow_statement` or `equity_statement`. Without
   `report_id` it auto-resolves the latest matching filing (`period_type`
   `annual` → 10-K/20-F/40-F; `quarterly` → 10-Q; `instant` for balance-sheet
   facts). Returns deduplicated consolidated facts ordered by period, plus the
   `resolved_report`. Start here for "show me X's statement".
2. **`build-fact-grid`** — *specific* elements across any mix of periods and
   entities. Use `canonical_concepts` (`revenue`, `net_income`, `total_assets`,
   …) rather than raw qnames so cross-filer tagging differences collapse
   (`us-gaap:Revenues` vs `us-gaap:RevenueFromContractWithCustomerExcludingAssessedTax`).
   Set `period_type` for duration concepts, `entities` for multi-company
   comparisons, and raise `limit` when `truncated` comes back true. This is the
   tool for "compare A and B" and "trend over N periods".
3. **`resolve-element`** — map a concept in words to the qnames actually used,
   with fact counts and a ready-to-run `query_hint`. Call it before writing
   Cypher that filters on an element.
4. **`search-documents`** — BM25 keyword search across filing narrative and
   iXBRL disclosure sections; `semantic: true` adds vector ranking. Filter with
   `entity`, `form_type`, `fiscal_year`, `section` (`item_1a`, `item_7`, …) or
   `element` (find disclosures carrying a given XBRL fact). Follow a hit with
   **`get-document-section`** for the full text; iXBRL hits carry
   `xbrl_elements` you can hand to `resolve-element` — the bridge from narrative
   back to numbers.
5. **`read-graph-cypher`** — when the shaped tools don't fit. Always run
   `get-example-queries` first; the patterns below are the ones that matter.

## Cypher patterns that work here

Consolidated revenue for one company, cross-filer robust:

```cypher
MATCH (f:Fact {has_dimensions: false})-[:FACT_HAS_ELEMENT]->(e:Element),
      (f)-[:FACT_HAS_ENTITY]->(ent:Entity {ticker: 'NVDA'}),
      (f)-[:FACT_HAS_PERIOD]->(p:Period {duration_type: 'annual'})
WHERE e.canonical_concept = 'revenue' AND f.numeric_value IS NOT NULL
RETURN ent.ticker, e.qname, p.end_date, f.numeric_value AS revenue
ORDER BY p.end_date DESC LIMIT 10
```

A whole statement via the presentation structure — anchor on the entity, reach
`Structure` last:

```cypher
MATCH (ent:Entity {ticker: 'NVDA'})<-[:FACT_HAS_ENTITY]-(f:Fact {has_dimensions: false})-[:FACT_HAS_ELEMENT]->(e:Element),
      (f)-[:FACT_HAS_PERIOD]->(p:Period {duration_type: 'annual'}),
      (fs:FactSet)-[:FACT_SET_CONTAINS_FACT]->(f),
      (s:Structure {canonical_type: 'income_statement'})-[:STRUCTURE_HAS_FACT_SET]->(fs)
WHERE f.numeric_value IS NOT NULL
RETURN DISTINCT e.qname, f.numeric_value AS value, p.end_date
ORDER BY p.end_date DESC LIMIT 40
```

Segment breakdowns — flip the dimension flag:

```cypher
MATCH (f:Fact {has_dimensions: true})-[:FACT_HAS_ELEMENT]->(e:Element),
      (f)-[:FACT_HAS_DIMENSION]->(d:Dimension)
WHERE e.qname = 'us-gaap:Revenues' AND f.numeric_value IS NOT NULL
RETURN d.axis_uri, d.member_uri, f.numeric_value LIMIT 10
```

## Rules of the road

- **Consolidated vs dimensional.** `has_dimensions: false` is the consolidated
  total; `true` is a segment/geography/member breakdown. Mixing them double-counts.
- **Canonical concepts over qnames.** `Element.canonical_concept` normalizes the
  tag zoo; `resolve-element` tells you which qnames map to it for a given filer.
- **Periods.** Income-statement and cash-flow facts are `duration` with a
  `duration_type`; balance-sheet facts are `instant`. Filter accordingly or you
  get an empty or mixed result.
- **Anchor selectively.** Lead every `MATCH` with an `Entity` or `Report`; never
  with `Structure`, `Element` or `Period` alone. Always `LIMIT`.
- **One `MATCH`, comma-separated patterns** for multiple relationships from the
  same node.
- **Numbers vs prose.** Cypher and the shaped tools answer with numbers;
  `search-documents` answers with prose. A complete analysis usually needs both:
  the figure, then the disclosure that explains it.
- **Fiscal calendars vary.** NVIDIA's fiscal 2026 ended 2026-01-25. Use
  `fiscal_year` / `fiscal_period` filters from the report's own focus rather
  than calendar assumptions.

## Worked shape

"Compare NVIDIA's gross margin across its last eight quarters":

1. `build-fact-grid` with `entity: NVDA`, `canonical_concepts:
   [revenue, cost_of_revenue]`, `period_type: quarterly`, `limit: 40`.
2. Compute margin per period from the returned pairs; note any period where a
   10-K annual fact stands in for a quarter (annual filers report Q4 implicitly).
3. If the trend needs explanation, `search-documents` with `entity: NVDA`,
   `section: item_7`, query "gross margin" — and cite the section.
