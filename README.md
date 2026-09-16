# @pipeworx/reginfo

The Unified Agenda of Federal Regulatory and Deregulatory Actions — every RIN
(Regulation Identifier Number) a federal agency has told OIRA it intends to
propose, finalize, or has completed, published biannually since Fall 1995.
This is the LEADING record ahead of `ecfr`/`federal-register`'s trailing one:
an agency's "planning to propose X" shows up here, edition by edition, often
years before the same rule actually publishes or takes effect.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `reg_agenda_search(agency?, keyword?, stage?, edition?, limit?)` — search RINs by agency, free-text keyword, and/or rulemaking stage. Defaults to the newest edition on file.
- `reg_rin(rin)` — full detail for one RIN across EVERY edition it has appeared in, showing how its stage/status changed over time.
- `reg_agency_agenda(agency, edition?, limit?)` — one agency's full regulatory agenda for an edition, grouped by rule_stage.
- `reg_agenda_coverage()` — which editions are currently loaded (oldest/newest, total rows).

## Auth

Keyless — no operator key needed. Backed by `reginfo_rins`, refreshed
monthly by `workers/data-pipeline` (dataset `reginfo`).

## Data sources

- <https://www.reginfo.gov/public/do/XMLReportList> — the Unified Agenda
  report index.
- <https://www.reginfo.gov/public/do/XMLViewFileAction?f=REGINFO_RIN_DATA_202504.xml>
  — one biannual edition's full XML (~16.5MB, ~3,800 `RIN_INFO` records).
  Editions run `REGINFO_RIN_DATA_199510.xml` .. present, April(04)/October(10)
  each year.

**A missing edition answers HTTP 200, not 404.** reginfo.gov serves a ~50KB
HTML "no such report" page (`Content-Type: text/html`) for an edition that
was never published — a real edition answers `Content-Type: application/xml`.
The ingest HEAD-probes content-type before ever GETting a file, so a not-yet-
published edition (or a genuine off-cycle gap in the pre-2000 grid) costs a
cheap HEAD request rather than leaking an HTML page into the data.

**One row PER (rin, publication_id), not per RIN.** A RIN's `rule_stage`
changes edition to edition ("Proposed Rule Stage" → "Final Rule Stage" →
"Completed Actions") — that history is the whole point of this dataset, so a
newer edition never overwrites an older one. `reg_rin` returns every edition
a RIN has appeared in, newest first.

**Coverage grows in gradually, not all at once.** The backfill drains in
edition by edition rather than loading all ~60 editions back to 1995 in one
pass; `reg_agenda_coverage` reports exactly which editions are loaded right
now, so a caller can tell "not backfilled yet" apart from "genuinely never
published".

**Agency matching covers name, acronym, AND code**, and matches on either the
direct agency or its PARENT — asking for "Interior" also surfaces its
sub-agencies' (e.g. Fish & Wildlife) rules, since the source records both
levels per RIN.

**The abstract is HTML wrapped in CDATA in the source XML.** The ingest
strips tags and decodes entities before writing the `abstract` column, so
callers only ever see plain text.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "reginfo": {
      "url": "https://gateway.pipeworx.io/reginfo/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/reginfo/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "reginfo": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-reginfo"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-reginfo
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Reginfo data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
