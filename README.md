# @pipeworx/wipo-pearl

Scientific, technical and IP legal terminology — search a term and get back
validated equivalents across 10 languages, each with a reliability rating,
defining context, usage note and source. Platform key, with a small keyless
fallback.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1843+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `pearl_search_terms(term, source_language?, target_languages?, subject_field?, abbreviation?, exact_search?, limit?)`
  — search for a term; returns matching concept records (concept id, matched
  term text, subject field).
- `pearl_concept(concept_id, lang_code?)` — the full database record for one
  concept id (from `pearl_search_terms`): equivalents grouped by language,
  each with a reliability rating, defining context, usage note and source.
- `pearl_subject_fields()` — reference list of the 29 subject-field codes and
  10 language codes the other two tools accept. No arguments.

## Auth

Platform key (`PLATFORM_WIPO_KEY`) with BYO override via `_apiKey`. Unlike
most platform keys, the stored value is a JSON STRING —
`{"client_id":"...","client_secret":"..."}` — not a bare token, because the
WIPO Business Partner Portal issues an application as a client_id/secret pair
rather than a single key. Each call JSON.parses it and exchanges the pair for
a short-lived Bearer JWT via OAuth2 client_credentials; the token is not
cached across calls.

**Activation is two gates, not one — registering an application is not the
same as being able to call the API.** The Business Partner Portal
(b2b.wipo.int) issues the client_id/client_secret immediately on
registration, but the application also has to be *subscribed* to the "WIPO
Pearl API" product/plan, and that subscription is reviewed and approved by
WIPO before it's usable — it is not instant and not guaranteed. A credential
that authenticates fine at the OAuth2 token endpoint can still get a 403 from
the Pearl API itself if the subscription step hasn't been approved yet; this
pack's error message says exactly that rather than a generic "forbidden", so
it doesn't read as our outage. If no credential (platform or BYO) is present
at all, calls fall back to WIPO's own tiny keyless allowance — its own portal
describes this as "5 requests per day" before you're expected to register —
which is only useful for a one-off smoke test, never production traffic
through a shared gateway.

## Data sources

- World Intellectual Property Organization (WIPO), WIPO Pearl terminology
  database — 250,000+ validated terms in Arabic, Chinese, English, French,
  German, Japanese, Korean, Portuguese, Russian and Spanish, across 29
  subject fields.

## Notes for the next person

- **The two tools are NOT equally reachable without a credential.**
  `pearl_search_terms`'s underlying search route answers keyless requests
  with real 200s (subject to the "5 requests/day" allowance above).
  `pearl_concept`'s underlying detail route answered a flat `500 Internal
  Server Error` on every keyless attempt made while building this pack —
  it appears to require the OAuth2 credential, and the upstream throws
  rather than returning a clean 401/403 when it's missing. Expect this
  route specifically to look "down" if ever called without a working
  credential; it isn't — it's just gated differently than the sibling route.
- **The keyless allowance is per calling IP, not a single global budget** —
  confirmed by exhausting it from one network path (a 429 "Rate limit
  exceeded! You reached the limit of 5 requests per 1440 minutes") while an
  identical call from a different egress still returned 200. Don't read one
  exhausted path as the whole API being down.
- **The product page is a Gravitee developer-portal SPA with no endpoint in
  its rendered HTML.** The live entrypoint, the OAuth2 token endpoint, the
  full request/response contract and the subject-field/language tables all
  came from the portal's own backend REST API and documentation pages (its
  `assets/config.json`, `/apis/<id>`, `/apis/<id>/pages` and the portal-wide
  FAQ pages) — see the file header in `src/index.ts` for exactly which pages.
  There is no dedicated subject-fields or languages endpoint; those two
  tables are from the product's own "Data Models" documentation page, not an
  API call, which is why `pearl_subject_fields` is static rather than a proxy.
- **The response shape of an authenticated `/records/{id}` call is inferred
  from the product's OpenAPI spec, not empirically observed** — building this
  pack had no working credential to test with. `pearl_concept` raises a loud
  `upstream_parse` error (naming the URL and body length) rather than a
  silent empty success if a real response doesn't match the documented
  shape, so a mismatch will surface on the first real call instead of sitting
  unnoticed.
- **`hits`/`count` in the search response are JSON strings** (`"70"`, not
  `70`) despite the spec declaring them as integers — parsed with `Number()`.
- **The upstream's own `limit`/`count` query parameters were unreliable in
  testing**: a request with neither set returned the full hit set (45-70+
  records) rather than the spec's documented 5-record default. This pack
  passes both anyway but also truncates the result client-side, so the
  response size is bounded regardless of whether the upstream honors either
  parameter.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "wipo-pearl": {
      "url": "https://gateway.pipeworx.io/wipo-pearl/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/wipo-pearl/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1843+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/pearl_search_terms \
  -H 'Content-Type: application/json' \
  -d '{"term":"satellite","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/pearl_search_terms`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "wipo-pearl": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-wipo-pearl"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-wipo-pearl
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Wipo Pearl data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
