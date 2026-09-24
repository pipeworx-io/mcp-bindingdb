# @pipeworx/bindingdb

Measured protein/small-molecule binding affinities from BindingDB — Ki, Kd,
IC50 and EC50 values in nM, each curated out of a published paper and carrying
the PubMed ID it came from.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `bindingdb_ligands_by_uniprot(uniprot, cutoff?, limit?)` — every ligand
  BindingDB has measured against a UniProt target, sorted by potency. Answers
  "what compounds bind this protein and how tightly".
- `bindingdb_targets_by_smiles(smiles, similarity?, limit?)` — the protein
  targets a compound (or a structurally similar one) has been measured against,
  with species. Answers compound-to-target and off-target questions.
- `bindingdb_by_pdb(pdb, limit?)` — affinities for the ligands associated with
  a PDB structure entry, linking a solved co-crystal to published potency.

## Auth

Keyless. No registration, no header.

## Data sources

- <https://bindingdb.org/rest/getLigandsByUniprots> — `uniprot=` (comma-separated
  accessions), `cutoff=` (affinity in nM), `response=application/json`.
- <https://bindingdb.org/rest/getTargetByCompound> — `smiles=`, `cutoff=`
  (Tanimoto similarity 0-1, NOT nM).
- <https://bindingdb.org/rest/getLigandsByPDBs> — `pdb=` (one 4-character ID).

Things that will otherwise cost you an afternoon:

- **The base path is `/rest/`, not `/rwd/bind/rest/`.** The latter is in older
  docs and 404s from Tomcat with an HTML body. `/axis2/services/BDBService/...`
  is also dead.
- **The JSON envelope key does not match the endpoint name.** All three respond
  under `getLindsByUniprotsResponse` / `getLindsByUniprotResponse` /
  `getLindsByPDBsResponse` — note "Linds", a typo that is part of the contract.
  The compound endpoint additionally prefixes every field with `bdb.`.
- **`getLigandsByPDBs` answers HTTP 500 with a SQL error in the body when it
  holds no data for that PDB ID** — 2RH1 and 1ZZ1 both do this, 3EML works.
  That is an absent-data signal wearing a server-error costume; the pack says so
  rather than letting it read as an outage.
- **Response size scales hard with `cutoff`.** P24941 at cutoff 1 is 387 KB, at
  10 is 1.1 MB, at 100 is 2.2 MB; P00533 at 100 is 5 MB. The default here is 10.
- **An affinity of `"0.000"` means unreported, not infinitely potent.** Values
  arrive as strings and may carry a qualifier (`<1`, `>10000`, `" 348000"`), so
  the pack splits them into `affinity_nm` + `qualifier` and nulls the zeros
  rather than ranking them first.
- `getLigandsByUniprots` returns rows for related targets as well as the exact
  accession you asked for — the `query` field on each row says which target the
  measurement is actually against.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bindingdb": {
      "url": "https://gateway.pipeworx.io/bindingdb/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bindingdb/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/bindingdb_ligands_by_uniprot \
  -H 'Content-Type: application/json' \
  -d '{"uniprot":"P24941","cutoff":1,"limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/bindingdb_ligands_by_uniprot`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "bindingdb": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bindingdb"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-bindingdb
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bindingdb data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
