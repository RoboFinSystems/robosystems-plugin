---
name: roboledger-close
description: >-
  Run or set up a month-end close on a RoboLedger graph over the RoboSystems MCP
  server — orient on the fiscal calendar, clear sync and reconciling-item blockers,
  draft schedule-driven adjusting entries, review, close the period, and verify the
  receipt. Use for "close the books", "close July", "what's blocking the close",
  "set up depreciation schedules", "month-end", "period close", "close Maple
  Court", or any request to post, review, or reopen a fiscal period on a
  RoboLedger graph, for the group parent or for one subsidiary of a reporting group.
---

# Month-end close on RoboLedger

The close turns a synced general ledger into a locked, reported fiscal period.
Recurring adjusting entries (depreciation, amortization, prepaid roll-off) are
modeled as **schedules** that emit per-period obligations; closing a period
drafts those entries, posts them, and advances the calendar.

**Always call `get-close-playbook` first.** It is version-locked to the server you
are connected to — the exact tool sequence, parameter shapes, and gotchas for
*this* build. This skill is the map; the playbook is the territory. Then run
`search-documents` for the company's own "close procedures" / "month-end"
document: it captures tenant-specific accounts and quirks the generic playbook
cannot, and it takes precedence where they differ.

## Which company

A graph is one **reporting group**: the group parent and the subsidiaries under
it, each with its own books, chart of accounts, calendar and close. The server's
`instructions` name them at the handshake, and `get-graph-info` lists them with
their ids. Every close tool (`get-fiscal-calendar`, `get-period-close-status`,
`list-period-drafts`, `promote-obligations`, `close-period`, `reopen-period`)
and every ledger write take `entity_id`; omitted, they act on the group parent.
To close a subsidiary, pass its `entity_id` on **every** call of the sequence —
its siblings stay open, and a close with no `entity_id` closes the parent. When
the user names a company, resolve it to its id once and carry the id through;
a graph with one entity needs none of this. `create-entity` adds a subsidiary
(name, legal form, the share the parent holds, an optional ticker); it gets a
chart of accounts from a template and a calendar on the group's cadence.

## Orient (every close)

1. `get-fiscal-calendar` (with `entity_id` for a subsidiary) — read
   `closed_through` (the watermark), `close_target`, `closeable_now`,
   `blockers`, `catch_up_sequence`. The period you close is exactly
   `closed_through + 1`; closes run in order, per entity.
2. `get-period-close-status` (`period_start`/`period_end` as `YYYY-MM-DD`) —
   which schedules are pending / drafted / posted, and the amounts.
3. `get-graph-sync-status` — freshness of the source connection (QuickBooks) and
   the last sync result.

## Clear the blockers

| Blocker | Meaning | Action |
|---|---|---|
| `sync_stale` | Source sync is older than the period end | `sync-connection` (incremental; pass `since_date` if edits are older than ~60 days), then poll `get-graph-sync-status` / `get-fiscal-calendar` until it clears. Prefer this over `allow_stale_sync=true`. |
| reconciling items > 0 | Transactions edited upstream after they synced — normal, not an alarm | `list-event-blocks` with `is_reconciling_item=true`; for each, `preview-reconciling-item`, agree a disposition with the user, then `resolve-reconciling-item`: **RESTATE** (regenerate in place; prior months change; default inside the fiscal year), **CATCH_UP** (post the difference in an open period; prior statements stand), or **ACKNOWLEDGE** (already handled by an entry you authored). Never close over unreviewed items. |
| `pending_obligations` / `stranded_obligations` | Matured schedule obligations not yet drafted | `promote-obligations` with `dispatch_handlers=true`; if they belong to a pre-watermark month, the schedule's `closed_through` was set wrong. |
| `sequence_violation`, `period_incomplete`, `calendar_not_initialized` | Out of order, month not over, no calendar | Close the earlier period, wait, or initialize the calendar. |

Schedules ending this period (asset sold, prepaid cancelled) are terminated
**before** promoting obligations: `terminate-schedule` (no entry) or
`create-event-block` with `event_type='asset_disposed'` (also posts the disposal
entry). `delete-information-block` is not a termination — it erases history.

## Draft, review, approve, close

1. `promote-obligations` (`dispatch_handlers=true`) — drafts every matured
   schedule's closing entry for the period in one idempotent sweep. There is no
   `create-closing-entry` tool.
2. `create-event-block` with `event_type='journal_entry_recorded'` for manual,
   one-off adjustments. `source='manual'` publishes to QuickBooks at close;
   `source='system'` stays local; set `metadata.publish_to_source` to decide
   explicitly. A catch-up entry mirroring an upstream edit must stay local.
3. `list-period-drafts` (`period` as `YYYY-MM`) — every draft with DR/CR detail,
   `all_balanced`, and the QuickBooks outbox split (`will_publish_to_qb`).
   Expect one draft per active schedule; investigate any gap.
4. Summarize to the user — totals, balance check, per-schedule amounts, what will
   publish to QuickBooks — and get **explicit approval**. The close is a co-pilot
   step, never autonomous.
5. `close-period` (`period` as `YYYY-MM`) — atomic: posts drafts, runs the
   balance-sheet equation check, advances `closed_through`. If it returns
   `status='in_progress'` with an `operation_id`, the close is running on the
   worker: poll `get-period-close-status` (or `get-fiscal-calendar` for
   `has_close_receipt`) every ~10 s and **never re-call `close-period`**.

## Verify

Read the receipt back to the user: `entries_posted` equals the reviewed draft
count (with the published/local split), `statements_stamped=true` with the
statement rules passing, every schedule shows posted. The close marks the graph
stale for the analytical rebuild — poll `get-graph-sync-status` until
`sync_status=fresh` before running statements or fact grids against the new
month. If the tenant runs forecasts, re-run `compute-forecast` for each forecast
block; the actuals/forecast seam advances by itself.

A subsidiary's close stamps that subsidiary's statements. The group's view is
`live-financial-statement` on the group parent with `consolidated=true`: every
entity's books summed per concept under the parent's Reporting Style —
**combined, not consolidated**, nothing is eliminated between the companies —
and a company whose chart is not mapped yet is left out of it.

## Setting up the first close (schedules)

- Discover the recurring entries two ways: derive amounts from a prior posted
  close (`read-graph-cypher`, `financial-statement-analysis`, `list-period-drafts`
  on a closed period) **and** interview the user for the parameters history
  cannot give — cost basis, useful life, method, roll-off.
- One schedule = **one debit element + one credit element**. A multi-line entry
  is several schedules.
- Element ids are chart-of-accounts ids from `get-unmapped-elements` /
  `get-graph-schema`, never taxonomy qnames. Discover them; don't invent them.
- Amounts are **integer cents** (`$194.83` → `19483`). `period_start..period_end`
  is the schedule's whole life. `schedule_metadata` (basis, residual, custom
  curve) is optional and can come later.
- Set the schedule's `closed_through` to the last day of the calendar's
  `closed_through` month, or the first close is blocked by every pre-watermark
  obligation — and re-drafting an already-closed month duplicates entries.
- Write a per-tenant "Month-End Close Procedures" document with
  `create-document` once the schedules exist; future closes start by reading it.

## Shapes to get right

- `list-period-drafts` and `close-period` take `period='YYYY-MM'`;
  `get-period-close-status` takes `period_start`/`period_end='YYYY-MM-DD'`;
  schedule payload dates are `YYYY-MM-DD`.
- `entity_id` is the entity's id from `get-graph-info` (`ent_…`, or
  `entity_<graph_id>` for a parent created with its graph), never its name or
  ticker. One entity per call: a schedule, a draft and a close each belong to
  exactly one company.
- Only `schedule` and `rollforward` are constructible via
  `create-information-block`; statements are built with `create-report`.
- On a shared repository none of this exists — there is no ledger to close.
