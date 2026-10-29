# @pipeworx/new-jersey-code

New Jersey Statutes (N.J.S.A.) by citation — with amendment history where the
source lists one — plus full-text topic search. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nj_statute_section(section)` — full text of one N.J.S.A. section by citation
  (e.g. `"2C:11-3"`, `"39:4-50"`, `"46:8-19"`), with its `amendment_history`
  line parsed out when the section has been amended.
- `nj_statute_search(query, limit?)` — full-text search by topic/keyword
  ("security deposit interest", "landlord grounds for eviction"); returns
  ranked sections with citation + heading where the hit title parses as one.

## Auth

Keyless.

## Data sources

- <https://lis.njleg.state.nj.us/nxt/gateway.dll> — the New Jersey
  Legislature's statute-viewer subdomain (NXT4/Folio document publishing,
  IIS-hosted). This is a DIFFERENT legislature subdomain than the
  njleg.state.nj.us Next.js site most links point at — that one is
  bills/session focused and does not itself host statute text.

### Shape of this source, because it is not a REST API

The viewer is session-cookie based, not a clean lookup-by-id endpoint:

1. `f=xhitlist` is a full-text search. It sets a session cookie and renders
   its hit list as plain server-side HTML (`<a class="hit-doc-link"
   href="...">title</a>` rows) — no JS execution needed to read it.
2. A citation lookup queries the EXACT quoted phrase `"<citation> "`
   (trailing space, inside the search). That trailing space is load-bearing:
   an unquoted citation free-text-matches thousands of sections that merely
   contain the same digits, and even the quoted phrase WITHOUT the trailing
   space still matches dotted/lettered children ("39:4-50.17", "39:4-50c")
   because "50" is a literal prefix of "50.17". The space forces the match
   onto a token boundary.
3. Each hit's `href` is an `f=hitdoc$hitdoc_bm=...` link. Resolving it (GET,
   with the search's session cookie, `redirect: manual`) 302s to the
   document's permanent, COOKIE-FREE path:
   `/nxt/gateway.dll/statutes/<a>/<b>/<c>`.
4. That path is fetched directly for the actual statutory text.

This is a session-cookie flow across two ordinary GETs — not the ASP.NET
`__VIEWSTATE` postback pattern (there is no postback form anywhere in it),
and it carries no CAPTCHA or other anti-bot wall. It has been reachable from
a Cloudflare Worker at the rate one caller generates; it has not been load
tested.

### Amendment history is free; a prior point-in-time version is not

Every amended section's text ends with a trailing `amended 1979, c.178,
s.21; ...; 2017, c.150.` (or, for a section's original enactment, `L.<year>,
c.<n>, s.<n>.`) line. `nj_statute_section` parses that into
`amendment_history` rather than leaving it buried in `text` — an original,
never-amended section simply has none (`null`). This source does NOT expose
a point-in-time/prior-version snapshot of a section; `nj_statute_section`
says so explicitly in its `historical_version` field rather than silently
omitting the capability.

### Topic search quality

`nj_statute_search` ANDs every word in the query (this is the upstream
engine's own behaviour, not something this pack adds) — a long sentence
over-narrows; a few plain words ("landlord grounds for eviction") works
better than a full question.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "new-jersey-code": {
      "url": "https://gateway.pipeworx.io/new-jersey-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/new-jersey-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/nj_statute_section \
  -H 'Content-Type: application/json' \
  -d '{"section":"39:4-50"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nj_statute_section`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-new-jersey-code` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/new-jersey-code/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "new-jersey-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-new-jersey-code"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about New Jersey Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
