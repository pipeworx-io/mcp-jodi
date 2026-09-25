# @pipeworx/jodi

Monthly oil and natural gas production, consumption, imports, exports and
closing stocks by country — the JODI-Oil and JODI-Gas World Databases,
self-reported to the Joint Organisations Data Initiative
(APEC/Eurostat/GECF/IEA/IEF/OLADE/OPEC/UNSD). Global coverage (~100+
countries), including every Gulf producer (Saudi Arabia, UAE, Kuwait, Qatar,
Oman, Bahrain, Iraq, Iran).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `jodi_oil_series(country, product?, flow?, months?, end_month?)` — one
  country's monthly oil series (crude/NGL/other/total-crude, or a refined
  product), with reporting gaps marked explicitly.
- `jodi_gas_series(country, flow?, months?, end_month?)` — same for natural
  gas (production, imports/exports split LNG vs pipeline, demand, stocks).
- `jodi_oil_compare(countries?, product?, flow?, months_back?, end_month?)` —
  rank countries by change in a series between the latest reported month and
  N months earlier. Defaults to the Gulf producer set. Answers both "which
  Gulf producers raised output most this quarter" (`months_back: 3`) and
  "Saudi exports vs a year ago" (`countries: ["Saudi Arabia"], months_back: 12`)
  — the same comparison with a different country list.
- `jodi_gas_compare(countries?, flow?, months_back?, end_month?)` — same for
  gas.
- `jodi_countries(commodity?)` — which countries JODI-Oil/Gas cover, and each
  one's earliest/latest reported month.

## Auth

Keyless. There is no upstream key to bring.

## Data sources

- <https://www.jodidata.org/oil/database/data-downloads.aspx> — JODI-Oil
  annual CSVs (primary: crude/NGL/other/total-crude; secondary: refined
  products), one file per year, 2002-present.
- <https://www.jodidata.org/gas/database/data-downloads.aspx> — JODI-Gas,
  one CSV covering its whole 2009-present history, resolved through JODI's
  own `https://api.publisher.jodidata.org/web/files/gas` JSON API rather than
  a hardcoded filename (the file has already been renamed once).

Ingested monthly by `workers/data-pipeline/src/datasets/jodi.ts` into
`jodi_oil_observations` / `jodi_gas_observations` (migration 161). Full
design notes — why a non-report is a missing ROW rather than a stored null,
and why only one of JODI's five published units survives ingest per flow —
live in the migration file and the dataset config; the short version:

- **A missing month is never a zero.** JODI is self-reported and coverage is
  uneven — some countries report late, some skip months, some never report a
  given series. Every tool here builds the full expected month range and
  marks gaps `{reported: false, value: null}` rather than only returning the
  rows that exist and letting the gap read as silence.
- **Units are pre-selected, not user-choosable.** JODI publishes every value
  in 5 units; this pack keeps KBD (thousand barrels/day) for oil rate flows,
  KBBL (thousand barrels) for the two level flows (closing stocks, stock
  change — JODI itself marks KBD "not applicable" for both), and M3 (million
  cubic metres) for everything in gas.
- Country names are ISO 3166-1 alpha-2 codes; JODI's raw files carry the code
  only, so this pack keeps its own code→name table (`COUNTRY_NAMES` in
  `src/index.ts`) covering every code seen in either database.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "jodi": {
      "url": "https://gateway.pipeworx.io/jodi/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/jodi/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/jodi_oil_series \
  -H 'Content-Type: application/json' \
  -d '{"country":"Saudi Arabia","flow":"exports","months":24}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/jodi_oil_series`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "jodi": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-jodi"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-jodi
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Jodi data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
