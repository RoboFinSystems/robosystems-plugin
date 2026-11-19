---
name: roboinvestor-receive-report
description: >-
  Load a report a portfolio company sent as a PDF or a spreadsheet onto a
  RoboInvestor graph over the RoboSystems MCP server: create the company as an
  issuer, read the statements out of the file value by value, name each line with
  a library concept, dry-run until every total foots, write it, and read it back.
  Use for "load this board package", "ingest Acme's Q2 financials", "add this
  portfolio company's report", "read this PDF into the fund", "here is their
  monthly reporting package", or any request to put a company's financial
  statements from a file onto a RoboInvestor graph.
metadata:
  lane: use
---

# Receive a portfolio company's report from a file

A company that keeps its books elsewhere sends a workbook or a PDF. This turns
it into a report on the fund's graph: every number a fact with the cell or page
it came from, every total checked against the lines beneath it, readable beside
the fund's holdings in that company.

**The server says how the tools work.** Its instructions at the handshake have
a RECEIVING REPORTS section, and `ingest-report` describes its own request.
Read both first: they match the build you are connected to. This skill is the
order of work and the judgement around it.

The connection must be scoped to a RoboInvestor graph. On any other graph
`ingest-report` is not in the tool list, and there is nothing to do here.

## 1. The company

A report hangs on an entity. A company that shares from its own RoboLedger
graph already has one. A company that does not is an **issuer**, made once
with `create-issuer`.

- Look before you create: `list-reports` and the graph's entities
  (`query-graphql`) show who is already there. `create-issuer` refuses a second
  issuer with the same LEI, CIK or tax id, but two issuers with the same name
  and no identifier are two entities.
- Keep the `id` it returns. `ingest-report` takes it as `entity_id`, and a
  security the fund holds in the company takes it as `entity_id` too
  (`create-security` or `update-security`).

## 2. Read the document before you map anything

Decide these, and tell the user what you decided:

- **Which report.** One `period_end` and one `kind` (`monthly-package`,
  `quarterly-package`, `annual-report`; free text). A second ingest for the
  same entity, period end and kind replaces the first.
- **The units.** A header may say `$'000s` over values that are whole dollars.
  Judge from a figure you can sanity-check (revenue, cash) and say which you
  chose. Send the value as the number it means, with `decimals` for its
  precision (`0` whole dollars, `2` cents, `-3` rounded to thousands).
- **The periods.** A balance is an instant: `end` only. A flow is a duration:
  `start` and `end`, both inclusive. A quarter-to-date and a year-to-date
  column are different periods of the same concept; load both if both are
  printed.
- **Which columns.** Load what the company reports as actual. Leave out
  variance and percentage-change columns: they are arithmetic on values you
  are already loading. Budget and forecast columns have no place yet; leave
  them out and say so.
- **Every printed line**, totals and subtotals included, and zeros. A total is
  only checked against lines that were sent.

Never retype a number. Read each value from its cell or its place on the page,
and give its `source`: `package.xlsx#sheet=BS&cell=B10` or
`package.pdf#page=3`. Send `require_source: true` so a value without one is
refused.

## 3. Name each line

Each line needs a concept. Use the library's wherever one fits.

- `search-library-elements` with `whole_library: true` and `brief: true`. Use
  the words of the line ("accounts payable"); the closest name comes first.
  Check the row's balance (debit or credit) and period type (instant or
  duration) against the line before you take it.
- **The document's wording goes in `labels`**, not in a new concept:
  `"rs-gaap:LaborAndRelatedExpense": "Personnel Expense"`. The report reads
  back in the company's words and stays comparable with every other report
  that uses the concept.
- **A value carries the sign of its concept, not of the page.** Accumulated
  depreciation printed as `(79,193)` under a credit-balance concept is sent as
  `79193`; the layout subtracts it. Printed negatives on contra lines are the
  usual reason a first dry run is refused.
- **Declare a concept of the company's own only when the library has none**
  (a cloud hosting cost line, a named loan, a KPI). Give it a prefix bound in
  `namespaces`, a `label`, a `period_type` and, for money, a `balance`. Two
  lines cannot share one concept, so two loans the library would both call
  notes payable need one of the company's own each.

## 4. Lay out the statements

- **Balance sheet and income statement:** name the section with its
  `block_type` and `layout: library`. The server arranges every library
  concept and checks every total. Give `lines` only for the company's own
  concepts, each with the `parent` it adds into and a `weight` (`1` adds, `-1`
  subtracts).
- **On a primary statement a concept of the company's own must add into a
  library concept's total**, directly or through another line that does. One
  that does not is refused. This keeps a statement's structure the library's.
- **Moving a library concept is a last resort.** If the company prints
  depreciation below its operating expense total, the library's arrangement
  will not foot as printed, and a line that moves the concept will. The report
  still loads, with `conforms_to_library: false` and each move listed in
  `departures`. Move a concept only when the document's own totals cannot foot
  otherwise, never to save a search.
- **Cash flow statement:** the library does not arrange it. Give every line.
  Use the library's totals (operating, investing, financing, and the change
  in cash) and the company's own concepts for the lines beneath them, signed
  as printed: positive increases cash.
- **A table of metrics** (ARR, headcount, backlog) is not a primary statement.
  Give it a heading concept (`abstract: true`) and its lines; no `block_type`.

## 5. Dry run until it is right

Send `dry_run: true` first, every time. Nothing is written.

A refusal names each failing total once: the period, what was reported, what
its lines sum to, and every line that was counted. Read the terms:

| What you see | What it usually is |
|---|---|
| The sum is off by twice a line's value | A contra sent with the page's sign |
| A line you sent is missing from the terms | The library files it under a different total; give it a line under the right one |
| "No section places these concepts" | A library concept this statement does not arrange; give it a line with its parent |
| `unanchored` | A concept of the company's own with no parent and weight on a primary statement |
| The terms are all right and the total still differs | A misread value, or the document itself does not foot |

**Never drop a line or change a number to make a total agree.** If the
document does not foot, stop and tell the user which total, in which period,
by how much. That is a finding about their report, not a problem to hide.

Nothing checks the cash flow statement against the balance sheet. Do it
yourself: opening cash plus the change in cash should equal closing cash on
the balance sheet, for each period. Report a difference.

## 6. Write it, then read it back

1. The same request without `dry_run`. Keep the `report_id`.
2. `list-reports` with the `entity_id` shows it; `disclosures` lists its
   sections; `information-block` reads one. Statement sections are named
   `Balance Sheet`, `Income Statement` and `Cash Flow Statement` whatever the
   document called them. Pass `periods` as the keys from the response's
   `columns` to see the columns you want.
3. Compare several values against the document, from different statements and
   periods, before you say it is done.

Then tell the user, plainly:

- how many values loaded, over which periods, and how many totals were checked
- whether the statements conform to the library, and each departure in words
- every judgement call: units, which columns were left out, any concept you
  declared and why
- anything the document itself got wrong

## A package with years of history

A single period fits in one tool call. Five years of quarters is thousands of
values and too large to write out by hand. Then:

- write a script that reads the cells and builds the request, so no number is
  retyped, and keep the row-to-concept mapping in it as the one thing a person
  reviews
- send it with the RoboSystems SDK (`ingest_report` on the investor client in
  `robosystems-client`, `ingestReport` in `@robosystems/client`), dry run
  first, exactly as above

A script is also how the same company's next package loads in minutes: the
mapping is already written.

## Afterwards

- **A correction** is a second ingest for the same entity, period end and
  kind. It replaces the first.
- **`delete-report`** removes the report, its facts, its sections, and the
  concepts it brought that no other report uses. The issuer stays.
- The stored record is a holon, a single file. `get-report-bundle` gives a
  download link, and the free viewer at xbrlkit.com opens it with no account.
- A company that can send a holon or a Tavi file needs none of the reading
  above: pass the file's text as `holon` or `tavi` instead of `report`. The
  same checks run. Values a file breaks out by a dimension (a segment, a
  product line) stay in the stored file and are counted in
  `dimensional_facts_not_loaded`; the graph holds the totals.
