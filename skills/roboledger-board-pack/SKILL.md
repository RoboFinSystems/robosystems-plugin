---
name: roboledger-board-pack
description: >-
  Build a board presentation from a RoboLedger graph over the RoboSystems MCP
  server — pull the closed period's statements, KPIs, working capital and
  operating plan, verify every figure against its guard rails, render a
  self-contained HTML deck and print it to PDF, then revise the plan from what
  the board decides. Use for "board deck", "board pack", "board meeting",
  "investor update", "quarterly business review", "QBR", "make the slides for
  the board", or any request to present a RoboLedger graph's results.
---

# Board pack from a RoboLedger graph

A board pack is the closed books, presented. This skill turns a RoboLedger graph
into two files next to each other — `<entity>-<period>-board-pack.html` (the
source) and `.pdf` (what goes to the board) — and then closes the loop by
revising the operating plan from what the board decides.

**You author numbers, not slides.** The deck is one JSON object inside
`assets/board-pack.html`; the renderer draws every slide from it. Read
`reference/deck-contract.md` before you write the first one — it carries the
slide kinds, the formats, the measured capacity limits, and the two commands
that gate and print the deck.

## Orient before you pull anything

1. `get-fiscal-calendar` — this is also the check that you are on a ledger graph
   at all. Read `closed_through`: **the pack reports a closed period.** If the
   period the user wants is not closed, say so and offer the close
   (`roboledger-close`) before the pack, or a clearly-labelled flash pack off
   live data. Read `reconcilingItemCount` and `syncStaleDays` too — both go on
   the cover.
2. `recall` — a prior pack, the board's standing asks, last quarter's decisions.
   Continuity is most of what makes the second pack better than the first.
3. `search-documents` for the company's own "board pack" / "board reporting"
   document. A tenant that has one has already decided what its board sees; it
   takes precedence over the default arc below.
4. Then ask the user, in **one** turn: which period, who is in the room, and the
   two or three decisions they want out of the meeting. The asks shape the pack
   — a deck that does not end in a decision is a status report.

## The pull

One pass, then build. Every row is a real capability, which is why the pack
exercises the whole platform rather than one statement endpoint.

| Slide | Tool | Notes |
|---|---|---|
| Cover attestation | `get-fiscal-calendar`, `get-graph-sync-status` | Closed through, close receipt, source freshness, unresolved reconciling items. This is the pack's audit trail — put it on page 1, not in an appendix. |
| Entity, ledger scale | `query-graphql` → `entity`, `summary` | Legal name for the cover; `transactionCount` / `entryCount` / `earliestTransactionDate` for the provenance appendix. |
| KPI scoreboard | `list-information-blocks` (`block_type='metric'`) → `get-information-block` | Metric blocks carry the standing series, one FactSet per period, so the deltas are the ledger's own, not arithmetic you did in your head. No metric blocks yet? Derive from the statements and say so in the source line. |
| P&L, balance sheet, cash flow | `live-financial-statement` | `statement_type` is `income_statement` \| `balance_sheet` \| `cash_flow_statement`; pass explicit `period_start`/`period_end` for the closed period. Returns current + prior. **Read the `validation` block** (see below). |
| Actuals against plan, and the trend | `get-information-block` with `series: true` and `scenario_id` | One column per period, crossing the actuals→forecast seam; forecast columns carry `periods[].forecast = true`. This is the single most useful call in the pack — it gives you the P&L-versus-plan slide and the forward view from one read. Window it with `series_history` / `series_forecast` on an old ledger. |
| Multi-period trend for a chart | `build-fact-grid` | Graph-backed, so it needs the ledger materialized. On a graph that is not, take the trend from the statement series instead. |
| Prior filed reports | `query-graphql` → `reports`, `statement(reportId:, blockType:)` | Ties the pack back to what was presented last quarter. |
| AR / AP concentration | `query-graphql` → `agents(agentType:"customer"){ id name openReceivable { openBalanceCents openEventCount } }` | Counterparties are REA agents. Use this rather than `openReceivablesByAgent`, which returns `agentId` with no name. **`openBalanceCents` is cents.** `agent-activity` drills one counterparty to its originating events. |
| Operating plan | `list-information-blocks` (`block_type='forecast'`) → `get-information-block` | The plan is a forecast block with a driver cascade; its `scenario_id` is what the statement series is read against. |
| Controls and data quality | `get-period-close-status`, `get-mapping-summary`, `get-unmapped-elements` | Close status by schedule, CoA→GAAP coverage, what is still unmapped. A board that is told the coverage number trusts the rest of the pack more, not less. |
| Narrative, commentary | `search-documents`, `get-document-section` | The tenant's own memos, policies and procedures — cite the section. |
| Anything unshaped | `get-graphql-schema` → `query-graphql`; `get-graph-schema` + `get-example-queries` → `read-graph-cypher` | Typed reads first; Cypher when the typed surface has no answer. |

## Verify before a number reaches a slide

The pack is the one artifact where a wrong number is expensive, so this step is
not optional.

- **`live-financial-statement` returns `validation`.** `status` is `passed` |
  `failed` | `inconclusive`. A `failed` statement does not foot or does not
  balance — do not put it on a slide as though it does. Either fix it or state
  it on the slide. A warning naming *'Other operating capital, net'* means the
  cash flow was made to foot by a reconciling plug: say so rather than
  presenting the figure as clean.
- **Foot the pack against itself.** Net income on the P&L, the equity movement
  on the balance sheet and the top of the cash flow are the same number. If the
  KPI card and the statement disagree, one of them is reading a different period
  or a different scenario.
- **Units, once.** Cents from the agent balances; dollars from the statements.
  Convert on the way in. A slide mixing the two is the most likely error in the
  whole build.
- **Scenario discipline.** Variance is against a *named, dated* plan. Say which
  one in the subhead, and never compare actuals to a scenario that was revised
  after the period closed.
- **Say the period out loud.** The closed period, not "the latest data".

## Build, gate, print

1. Copy `assets/board-pack.html` to `<entity>-<period>-board-pack.html` in the
   user's working directory.
2. Replace the `<script id="deck">` JSON with the deck. Follow
   `reference/deck-contract.md`; stay inside the measured capacity limits.
3. Run the overflow gate (the `--dump-dom | grep` command in the contract).
   **Empty output is the pass.** A flagged slide is drawing over its own footer:
   move content, re-run. Do not print a flagged deck — it carries a red
   `OVERFLOW` banner onto the page.
4. Print to PDF with the contract's Chrome command, then tell the user both
   paths and walk them through the arc in a few lines.

The default arc, when the tenant has no shape of its own: cover · where we stand
· scoreboard · revenue against plan · margin and opex · balance sheet · cash and
runway · working capital · the operating plan · controls and close · risks ·
the asks · provenance appendix. Vary the slide kinds — several `statement`
slides in a row is a spreadsheet, not a pack.

## Then offer the two write-backs

The build is entirely reads. Once the deck exists, offer — do not assume:

- **File the pack in the ledger.** `create-document` with the pack's narrative
  and its figures, so next quarter's `search-documents` finds what this board
  was told.
- **Remember the asks.** `remember` the decisions requested and the commitments
  made, so the next pack opens against them.

## After the meeting — revise the plan

This is the half that makes the pack a loop instead of a document. When the user
comes back with what the board decided:

1. **Capture the decisions first**, in their words, and `remember` them. A
   decision that only exists in a slide revision is lost.
2. **Move the levers, not the outputs.** The plan is a forecast block driven by a
   lever cascade: `get-information-block` to read the current levers,
   `update-information-block` to change the ones the board moved (the hiring
   plan, the price, the ramp), and `compute-forecast` to walk the cascade
   forward. Re-typing a target into a slide is not a revised plan — nothing
   downstream would ever see it.
3. **Read the revision back** with `get-information-block` (`series: true`,
   `scenario_id` of the revised block) and confirm the new numbers with the user
   before they go on a slide.
4. **Add the revised-plan slide** — a `chart` whose second series is the revised
   scenario with `projected: true`, plus a `split` naming each lever that moved
   and its effect. Keep the original plan visible: the board needs to see what
   changed, not just the new line.
5. **Re-gate, and print to `…-v2.pdf`.** Never overwrite the PDF that was
   presented; the record has to match what the board saw.

Run a revision as its own confirmed step. `update-information-block` and
`compute-forecast` write to the customer's graph — say exactly which levers will
move and get an explicit yes first.

## What not to do

- Do not hand-author slide HTML. If the deck needs a shape the contract has not
  got, say so rather than escaping the renderer.
- Do not present an open period as closed, or a figure whose `validation`
  failed as though it passed.
- Do not put a number on a slide you could not trace to a tool call. The
  provenance appendix is the promise that you can.
- Do not build a pack on a shared repository such as `sec` — there is no ledger
  there. Public-company comparatives are a separate connection: one
  authorization is one graph, so a comps slide needs a `sec`-scoped connection
  alongside this one, and is skipped when there is not one.
