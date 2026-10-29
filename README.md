# @pipeworx/georgia-archives

The three public catalogues of the National Archives of Georgia — the scientific
reference library (Koha), the national fonds register, and the Georgian national
film catalogue.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/archives_library_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"tbilisi"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/archives_library_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-georgia-archives` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/georgia-archives/mcp` endpoint shown above. It needs network access;
there is no offline mode.

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
ask_pipeworx({ question: "your question about Georgia Archives data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
