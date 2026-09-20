# RoboSystems plugin

Accounting and financial-reporting knowledge graphs for coding agents. This plugin wires the [RoboSystems](https://robosystems.ai) MCP server into Claude Code, Cursor, Grok Build, and any agent that reads the Claude or Cursor plugin format, and ships four skills that teach the agent how to use it well.

**What the server exposes** — one authorization is one graph, chosen at consent:

| Graph | What it is | Tools |
|---|---|---|
| `sec` — SEC EDGAR Filings | Public-company XBRL filings as a read-only knowledge graph | analytical set |
| Your RoboLedger graph | A general ledger with XBRL-grade reporting: fiscal calendar and period close, journal entries, live statements, forecasts, chart-of-accounts mapping, documents, memory. Connect QuickBooks Online and your books become the graph. | full set; writes limited by your role |
| Your RoboInvestor graph | Portfolios, securities and positions alongside the ledger | ledger set + portfolio tools |

**Skills**

| Skill | Use it for |
|---|---|
| `robosystems` | Orientation — what the connection is scoped to, which tool family answers an intent, how to explore a graph (schema → example queries → GraphQL or Cypher) and the Cypher rules that matter |
| `sec-filing-analysis` | Statements by ticker or CIK, cross-period and cross-company comparisons, concept → XBRL element, full-text search over 10-K/10-Q narrative |
| `roboledger-close` | Month-end close: orient, clear blockers, draft schedule-driven entries, review, close, verify — and how to set up schedules for a first close |
| `roboledger-board-pack` | A board presentation from your ledger: pull the closed period, verify it against its guard rails, render a deck to HTML and PDF, then revise the operating plan from what the board decides |

## Install

**Claude Code**

```bash
claude plugin marketplace add RoboFinSystems/robosystems-plugin
claude plugin install robosystems@robosystems
```

The first tool call opens the browser for the OAuth consent screen, where you sign in to RoboSystems and pick the graph. To work on a different graph later, run `/mcp` and reauthenticate.

**Grok Build** — install from the [xAI plugin marketplace](https://github.com/xai-org/plugin-marketplace) once listed; the plugin is the same directory.

**Cursor** — install from the [Cursor Marketplace](https://cursor.com/marketplace) once listed (**Customize → Plugins**), or test it locally by copying this repository to `~/.cursor/plugins/local/robosystems` and running **Developer: Reload Window**. The `.cursor-plugin/` manifest points at the same server and skills.

**Any MCP client** — the server is `https://api.robosystems.ai/v1/mcp` (Streamable HTTP, OAuth 2.1 with PKCE, discovery via RFC 9728 / RFC 8414). Per-graph URLs with an API key are also available; see the [MCP guide](https://github.com/RoboFinSystems/robosystems/wiki/AI-Operators-and-MCP).

## Requirements

A RoboSystems account (sign up at [robosystems.ai](https://robosystems.ai)). To query SEC EDGAR you need a subscription to the shared SEC repository; to work on your own books you need a RoboLedger graph. Database reads over MCP consume no credits.

## Security

The plugin contains no hooks, no commands, and nothing that runs on its own — an MCP server URL, four Markdown skills, and one HTML deck template. The only network endpoint it reaches is `api.robosystems.ai`, over OAuth; it reads no environment variables and no secrets. Every tool carries MCP `readOnlyHint` / `destructiveHint` annotations so the client can confirm before a write.

Two things `roboledger-board-pack` does that the other skills do not, both visible to you before they happen: it writes the deck into your working directory, and it asks your agent to run local headless Chrome to print that file to PDF. It builds the deck from reads alone; filing the pack back to your ledger, remembering the board's decisions, and revising the operating plan are each offered separately and need your explicit yes.

## License

Apache-2.0 © 2026 RFS LLC. RoboSystems itself is open source: [github.com/RoboFinSystems/robosystems](https://github.com/RoboFinSystems/robosystems).
