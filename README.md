# @pipeworx/georgia-archives

The three public catalogues of the National Archives of Georgia — the scientific
reference library (Koha), the national fonds register, and the Georgian national
film catalogue.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `archives_library_search(query, index?, offset?)` — search the reference library
  by keyword / title / author / subject. Returns a **true** total (not a page
  count) plus 20 records, each with its permanent `biblionumber` and catalogue URL.
- `archives_library_record(biblionumber)` — one record as parsed MARC: the full
  field/subfield structure plus a flattened summary (title, authors, imprint,
  ISBN, languages, subjects).
- `archives_fonds_browse(page?)` — walk the fonds register: ~14,650 fonds over 586
  pages of 25. Each row carries the holding regional archive, the fond number, the
  fond name, the historical names it held with their date ranges, and the
  inclusive dates of its administrative and personnel records.
- `archives_film_search(name?, page?, locale?)` — the film catalogue's 652
  newsreels, documentaries and features, with directors and camera operators as
  `{id, name}` objects carrying the catalogue's own person ids.

## Auth

Keyless. Every endpoint answers without a credential.

## Data sources

- <https://library.archives.gov.ge/cgi-bin/koha/opac-search.pl?idx=kw&q=…&format=rss2>
  — Koha OpenSearch RSS. `<opensearch:totalResults>` is an honest total and
  `&offset=N` is honoured, so pagination has a real cursor.
- <https://library.archives.gov.ge/api/v1/public/biblios/{id}> — Koha public REST.
  `/api/v1/` itself is a keyless Swagger 2.0 spec, 129 paths, 13 of them public.
- <https://archival-services.gov.ge/fonds/?page=N> — the fonds register.
- <https://kinokatalogi.archive.gov.ge/{ka|en}/records?name=…> — the film
  catalogue, an Inertia/Laravel app that returns JSON when asked.

## Things the next person would otherwise rediscover the hard way

**Koha's biblio route rejects `Accept: application/json` with a 400.** It requires
a MARC media type — `application/marc-in-json` or `application/marcxml+xml`. Test
it with default headers and you will file a working endpoint as dead.

**The fonds register's trailing slash is load-bearing.** `/fonds?page=2` 301s to
`/fonds/` and drops the query, so you get page 1 back with a 200. Its search is
POST-only behind a `_token` CSRF field, and the same parameter names sent over GET
are *silently ignored* — page 1 again, which reads as a clean zero-result. There is
no JSON surface here at all (`main.js` greps out to jQuery internals, zero
endpoints). Browse-and-index is the only honest shape; do not build a search proxy
on it.

**The film catalogue ignores unknown parameters instead of rejecting them.**
`search=`, `keyword=` and `title=` all return the unfiltered 652 with a 200. Only
`name` filters. That is why this pack accepts exactly one filter and nothing else —
an accepted-but-ignored argument would hand the caller the whole catalogue dressed
as a search result.

**`X-Inertia-Version` is a deploy fingerprint, not a constant.** Hard-code it and
you get a 409 the next time they deploy. `currentInertiaVersion()` reads it out of
the page's `data-page` attribute on every call, which costs one extra request and
cannot go stale.

**OAI-PMH is not enabled on the Koha instance** (`/cgi-bin/koha/oai.pl?verb=Identify`
→ 404), so there is no bulk path; the REST + OpenSearch pair is the whole surface.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "georgia-archives": {
      "url": "https://gateway.pipeworx.io/georgia-archives/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/georgia-archives/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/archives_library_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"tbilisi"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/archives_library_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "georgia-archives": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-georgia-archives"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-georgia-archives
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Georgia Archives data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
