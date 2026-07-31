# Exact Online AI Connect (Claude Code plugin)

Connects Claude Code to your Exact Online administration through the hosted MCP server at
`https://mcp-exactonline.iwebdevelopment.com`. You authorize in your browser on first use, so
there are no tokens to paste into a config file.

```
/plugin marketplace add iwebdevnl/exact-online-ai-connect
/plugin install exact-online-ai-connect@iwebdevnl-exact
/reload-plugins
/mcp
```

The last command signs you in. Full documentation and the client compatibility matrix are in the
[repository README](https://github.com/iwebdevnl/exact-online-ai-connect).

## What is in this plugin

| Component | Contents |
|---|---|
| MCP server | One hosted HTTP server (`.mcp.json`), OAuth in the browser |
| Commands | 4 slash commands, prefixed `exact-` |
| Skills | 10 skills that load automatically when the question matches |

The plugin ships `defaultEnabled: false`, so it stays off until you enable it. That is deliberate:
the MCP server is a paid subscription and the tools reach your live administration.

## Commands

Command names carry an `exact-` prefix so they do not collide with the Twinfield and AFAS
plugins, which offer the same four reports.

| Command | Does |
|---|---|
| `/exact-balans [periode]` | Balance sheet and general ledger totals |
| `/exact-omzet [jaar]` | Revenue per customer per month |
| `/exact-openstaande-debiteuren [administratie]` | Outstanding receivables |
| `/exact-ouderdomsanalyse` | Aging analysis of receivables |

## Skills

Skills are instructions Claude loads by itself when your question matches. You do not invoke them.
All skill content is in Dutch, matching the accounting vocabulary of Exact Online.

| Skill | Loads for |
|---|---|
| `btw-aangifte-assistent` | Checking or preparing a VAT return, ICP |
| `cashflow-analyse` | Cash flow statement, working capital KPIs, Excel deliverable |
| `creditcard-aflettering` | Processing a credit card or PSP statement end to end |
| `debiteurenbeheer` | Chasing receivables, escalation, reminder drafts |
| `exact-afletter-logica` | Finding and matching unreconciled entries |
| `grootboek-anomalie-detectie` | Bookkeeping quality checks, suspense accounts, duplicates |
| `management-informatie` | Commercial steering information, dashboards, KPIs |
| `periodeafsluiting` | Preparing a month or quarter close |
| `reporting` | Which tool or endpoint to use for a given question |
| `resultatenrekening-analyse` | A formal profit and loss statement (BW2 Titel 9) |

## Requirements

- An Exact Online AI Connect subscription. Trial and Analytics plans include the analysis tools
  (`analyze_data`); the Essentials plan does not, and the skills say so where it matters.
- `cashflow-analyse` generates an Excel workbook with a bundled Python script. That step needs
  `python3` with `openpyxl` installed (`pip install openpyxl`). Every other skill works without it.

## Support

Questions about the product go to <support@iwebdevelopment.com>. Bugs in this plugin go to the
[issue tracker](https://github.com/iwebdevnl/exact-online-ai-connect/issues). Security reports:
see [SECURITY.md](https://github.com/iwebdevnl/exact-online-ai-connect/blob/main/SECURITY.md).

MIT licensed. Exact and Exact Online are trademarks of Exact Group B.V. This plugin is built by
iWebDevelopment and is not affiliated with or endorsed by Exact.
