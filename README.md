# RoboSystems Plugin

**The agent plugin for [RoboSystems](https://github.com/RoboFinSystems/robosystems), the open-source knowledge graph for finance.** It connects your agent to the RoboSystems MCP server and ships five skills that teach it to use the server well. It installs in any agent that reads the Claude or Cursor plugin format; [Install](#install) covers each.

One authorization is one graph, chosen at consent:

| Graph | What it is | Tools |
|---|---|---|
| `sec` (SEC EDGAR Filings) | Public-company XBRL filings as a read-only knowledge graph | analytical set |
| Your RoboLedger graph | RoboLedger, the general ledger built on RoboSystems, with XBRL-grade reporting: fiscal calendar and period close, journal entries, live statements, forecasts, chart-of-accounts mapping, documents, memory. Connect QuickBooks Online, or keep the books here, fed by bank feeds | full set; writes limited by your role |
| Your RoboInvestor graph | Portfolios, securities and positions, and the reports portfolio companies send: shared from RoboLedger, or loaded from a PDF or a spreadsheet | portfolio tools and the tools for receiving and reading a report; the ledger set too when the graph also carries RoboLedger |

## Skills

- **`robosystems`**: orientation: what the connection is scoped to, which tool family answers an intent, how to explore a graph (schema → example queries → GraphQL or Cypher) and the Cypher rules that matter
- **`sec-filing-analysis`**: statements by ticker or CIK, cross-period and cross-company comparisons, concept → XBRL element, one disclosure section read whole, and search over 10-K/10-Q narrative and one filing's own text, 8-K earnings releases included
- **`roboledger-close`**: month-end close (orient, clear blockers, draft schedule-driven entries, review, close, verify) for the group parent or one subsidiary, then the group close, and how to set up schedules for a first close
- **`roboinvestor-receive-report`**: a portfolio company's report from a PDF or a spreadsheet onto a RoboInvestor graph: create the issuer, read the statements value by value with the cell or page each came from, name the lines with library concepts, dry-run until every total foots, write it and read it back
- **`roboledger-board-pack`**: a board presentation from your ledger: pull the closed period, verify it against its guard rails, render a deck to HTML and PDF, then revise the operating plan from what the board decides

## Install

Needs a RoboSystems account ([robosystems.ai](https://robosystems.ai)). To query SEC EDGAR you need a subscription to the shared SEC repository; to work on your own books you need a RoboLedger graph. Database reads over MCP use no credits.

**Claude Code**

```bash
claude plugin marketplace add RoboFinSystems/robosystems-plugin
claude plugin install robosystems@robosystems
```

The first tool call opens the browser on the OAuth consent screen, where you sign in to RoboSystems and pick the graph. To work on a different graph later, run `/mcp` and reauthenticate.

**Grok Build.** Install from the [xAI plugin marketplace](https://github.com/xai-org/plugin-marketplace) once listed; the plugin is the same directory.

**Cursor.** Install from the [Cursor Marketplace](https://cursor.com/marketplace) once listed (**Customize → Plugins**), or test it locally by copying this repository to `~/.cursor/plugins/local/robosystems` and running **Developer: Reload Window**. The `.cursor-plugin/` manifest points at the same server and skills.

**Any MCP client.** The server is `https://api.robosystems.ai/v1/mcp` (Streamable HTTP, OAuth 2.1 with PKCE, discovery via RFC 9728 / RFC 8414). Per-graph URLs with an API key are also available; see [AI Operators & MCP](https://robosystems.ai/docs/technical/ai-operators-and-mcp).

## Security

The plugin has no hooks, no commands and nothing that runs on its own: an MCP server URL, five Markdown skills and one HTML deck template. The only network endpoint it reaches is `api.robosystems.ai`, over OAuth; it reads no environment variables and no secrets. Every tool carries the MCP `readOnlyHint` / `destructiveHint` annotations, so the client can confirm before a write.

`roboledger-board-pack` does two things the other skills do not, both visible to you before they happen: it writes the deck into your working directory, and it asks your agent to run a local headless Chrome to print that file to PDF. It builds the deck from reads alone; filing the pack back to your ledger, remembering the board's decisions and revising the operating plan are each offered separately and need your explicit yes.

## Documentation

- **Product guides**: [robosystems.ai/docs/guides](https://robosystems.ai/docs/guides). For this plugin: [Skills and the plugin](https://robosystems.ai/docs/guides/skills-and-the-plugin) · [Connect an MCP client](https://robosystems.ai/docs/guides/connect-an-mcp-client) · [OAuth and graph scope](https://robosystems.ai/docs/guides/oauth-and-graph-scope)
- **Technical docs**: [robosystems.ai/docs/technical](https://robosystems.ai/docs/technical). For this plugin: [AI Operators & MCP](https://robosystems.ai/docs/technical/ai-operators-and-mcp)
- **Related**: [RoboSystems](https://github.com/RoboFinSystems/robosystems) · [MCP client](https://github.com/RoboFinSystems/robosystems-mcp-client)

## Support

- [Issues](https://github.com/RoboFinSystems/robosystems-plugin/issues) · [Discussions](https://github.com/orgs/RoboFinSystems/discussions) · [Projects](https://github.com/orgs/RoboFinSystems/projects)

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

Apache-2.0 © 2026 RFS LLC
