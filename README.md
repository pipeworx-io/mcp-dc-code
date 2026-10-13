# @pipeworx/dc-code

District of Columbia Code (D.C. Code — DC statutes) — full section text by
citation, and topic/chapter search over official headings.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1784+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `dc_statute(citation)` — full current text of one D.C. Code section (e.g.
  `"22-2101"`, murder in the first degree), with the official heading,
  disposition (in force / repealed / transferred / reserved), legislative
  History annotations, Prior Codifications, and Cross References.
- `dc_search(query?, title?, limit?)` — topic/keyword search over every
  section's official heading (e.g. `"eviction notice"`, `"concealed carry
  permit"`), optionally scoped to one `title`. Pass `title` alone (no
  `query`) to list every section under that title — a chapter-list/browse
  mode. Not full-text search — the DC Council does not publish one.

## Auth

Keyless.

## Data sources

- `https://raw.githubusercontent.com/DCCouncil/law-xml-codified/{branch}/us/dc/council/code/titles/{title}/sections/{citation}.xml`
  — one section's current XML (its heading, body text, and annotations).
  `dc_statute` calls this live, per request, pinned to the branch baked
  below.
- Same repo's `us/dc/council/code/index.xml` and
  `titles/{title}/index.xml` — structure only (title/chapter/subchapter
  headings + section citation lists, no catchlines). Used only by
  `scripts/bake-index.mjs`.
- `https://api.github.com/repos/DCCouncil/law-xml-codified/branches` — used
  only by `scripts/bake-index.mjs` to resolve the current publication
  branch (see below).

## Licence — public domain, and why this pack is allowed to bake an index

code.dccouncil.gov's own footer: *"The codes and laws on this website are in
the public domain."* The same footer: *"Please do not scrape. Instead, bulk
download the HTML or XML"* from the Council's GitHub organization. This pack
follows that instruction literally — it never touches code.dccouncil.gov,
only `raw.githubusercontent.com` content from `DCCouncil/law-xml-codified`.
Public-domain content plus the publisher's own explicit redistribution
channel means baking a catchline index from this repo needs no separate
grant (root CLAUDE.md, "mirror-needs-grant-proxy-does-not" — the
licence-permits-it branch, not the silent/forbidden one). Statutory text
itself is also never baked — `dc_statute` always fetches it live, per call.

## The default-branch trap

`GET /repos/DCCouncil/law-xml-codified` reports `default_branch:
"publication/2021-10-18"` — real git metadata, frozen at a 2021
codification, five years stale, even though the repo is actively published:
406 branches existed at bake time, the newest a
`publication/{base-date}.{amendment-date}` pair updated within days.
GitHub's "default branch" setting is unrelated to which branch is current.
`scripts/bake-index.mjs` lists every branch (paginated, ~5 unauthenticated
requests) and picks the lexicographically-latest name — safe because every
name is an ISO-8601 date or a dotted pair of them, so string order is
chronological order. The resolved name is baked as `DC_CODE_BRANCH` and is
what `dc_statute`'s live fetch pins to.

## Section XML shape

A short section is one flat `<text>` element; a section with
lettered/numbered subsections nests `<para>` elements, each with its own
`<num>` ("(a)") and `<text>`, repeatedly — sometimes nested one level
deeper. A `reason` element reading "Repealed" (or Reserved/Transferred)
marks a non-substantive stub whose body text is literally "Repealed." —
surfaced as `disposition` rather than left for the caller to infer. A
trailing `annotations` block (History, Prior Codifications, Cross
References, Editor's Notes) is excluded from the body-text walk and returned
as its own grouped fields.

One parsing trap worth knowing: a `num` or `heading` element can carry a
NESTED inline element (e.g. Title 7 Chapter 2B's chapter number carries an
embedded effective-date annotation tag inside its own `num` element) — a
tight "no-nested-tag" regex silently drops that chapter's number instead of
erroring. `scripts/bake-index.mjs` strips any nested tag content rather than
assuming `num`/`heading` are always tag-free.

## Capabilities — two available, two are not

- **Citation lookup** — `dc_statute`. Direct URL construction from the
  citation; no index lookup needed (title is the citation's own prefix
  before its first `-`).
- **Topic/chapter search** — `dc_search`, over official headings only (see
  below). Matches 24,198 sections' real headings, baked once per
  `scripts/bake-index.mjs` run — see its header for why per-section fetches
  were unavoidable (the index tree carries no catchlines, unlike South
  Dakota's or Oklahoma's siblings).
- **Amendments/enactment history** — PARTIALLY AVAILABLE. Every amended
  section's `annotations` block carries one or more `type="History"`
  entries naming the enacting/amending Act; `dc_statute` returns these as
  `history`. There is no single canonical "effective date" field separate
  from these citations.
- **Historical version** (the text as it read before an amendment) — NOT
  AVAILABLE. `DCCouncil/law-xml-codified` publishes only the CURRENT
  codified text, no per-amendment archive and no endpoint returning a prior
  version's text.

## Refresh path

DC Council publishes a new amendment branch roughly every few days and a new
base codification a few times a year. Re-run
`node scripts/bake-index.mjs > src/dc-index-data.ts` whenever a caller's
`index_live_mismatch` or a stale `data_as_of` suggests the baked index has
fallen behind the live branch — nothing else in the pack needs to change;
`dc_statute` always fetches section text live, pinned to whatever branch the
bake most recently resolved. No scheduled workflow exists for this (same as
every sibling state-code pack in this family) — it is a manual re-run.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "dc-code": {
      "url": "https://gateway.pipeworx.io/dc-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/dc-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1784+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/dc_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"22-2101"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/dc_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "dc-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-dc-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-dc-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Dc Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
