# datafordeler-mcp

> **Moved.** This project is now [bbr-mcp](https://github.com/JulianUmbhau/bbr-mcp), on PyPI as
> [`bbr-mcp`](https://pypi.org/project/bbr-mcp/). This repository is archived.

An [MCP](https://modelcontextprotocol.io) server for Danish buildings and addresses. It lets a model:

- find an address from free text,
- look up the address's details and coordinates,
- read the BBR buildings and units registered there: use, year built, areas, materials, heating installation and
  medium, housing type, rooms, and more. Code values come with their official Danish text.

It calls the official services live on each request:

| Data | Service | Credential |
|---|---|---|
| Addresses (DAR) | [Adressevælger](https://confluence.kds.dk/pages/viewpage.action?pageId=234782998), Klimadatastyrelsen's replacement for DAWA autocomplete | none |
| Buildings and units (BBR) | [Datafordeler GraphQL](https://confluence.kds.dk/display/DML/GraphQL), `BBR/v3` | `DATAFORDELER_API_KEY` |

It uses no DAWA (closed in 2026) and no legacy Datafordeler REST or username/password login (both end 15 January 2027).

## Tools

| Tool | Input | Returns |
|---|---|---|
| `address_search` | `text`, `level` (`adresse` or `husnummer`), `limit` | DAR ids and labels of the matching addresses |
| `address_get` | `id` of a unit address or a husnummer | full address, postcode, municipality and road code, husnummer id, coordinates |
| `bbr_buildings` | one of `husnummer_id`, `address_id`, `building_id`, `jordstykke_id`; `limit` | BBR buildings (`byg0xx` fields) |
| `bbr_units` | one of `address_id`, `building_id`, `unit_id`; `limit` | BBR units (`enh0xx` fields) |
| `bbr_buildings_in_area` | an area (`bbox`, `polygon_wkt`, `center` + `radius_m`, or `around_address_id` + `radius_m`), `crs` (`25832` or `4326`); filters `uses`, `heating`, `heating_medium`; `include_not_existing`; `limit` | a summary of the buildings (by use, heating installation, heating medium, supplementary heat and year built, total areas) and a compact list |
| `bbr_buildings_for_addresses` | `addresses` (free text) or `husnummer_ids`, at most 200; `include_units` | each address's match (with `exact` and alternatives) and buildings, plus a summary over all of them |

Area and list notes:

- **Size limits.** An area may be up to 25 km² and up to 10,000 buildings. A radius may be up to 2.5 km.
- **Coordinates.** WGS84 longitude/latitude is converted to EPSG:25832 locally, because BBR's spatial filter takes
  only its own coordinate system.
- **Filters.** BBR can filter on use and status itself. It has no heating filter, so `heating` and `heating_medium`
  apply to the fetched buildings. A building whose heating is registered on its units (code 99) doesn't match them.
- **Address matching.** Free-text addresses go through Adressevælger, and `exact` says whether the match is the
  same address written differently. Check non-exact matches before relying on them.

- Coded fields come as `{"code": "1", "text": "Fjernvarme/blokvarme"}`, and empty fields are left out.
- Coordinates are EPSG:25832 (ETRS89 / UTM 32N, metres).
- Every BBR query asks for the data as registered and valid now.
- `results` and `count` hold only buildings and units that stand now (status `6 Opført`). BBR keeps demolished buildings
  (status `10 Historisk`), and also holds planned, under-construction and shelved ones. These are listed apart in
  `not_existing`, each with a `state`. Objects registered by mistake (`11 Fejlregistreret`) are only counted.

### Building states

BBR's own values are never changed. Every row keeps its `status` code and official text. `state` and the grouping
are this server's reading:

| Status | `state` | Basis |
|---|---|---|
| 6 Opført | in `results` | BBR [D0130 Logisk Intern Datamodel](https://teknik.bbr.dk/file/666026/d0130-logisk-intern-datamodel-bbr-1_8.docx), section 3.2 |
| 2 Projekteret | planned | D0130 |
| 3 Under opførsel | under construction | D0130 |
| 10 Historisk | demolished or otherwise ceased | D0130: "ved nedrivning af en bygning sættes status til historisk" |
| 14 Henlagt | shelved, never built | D0130 |
| 11 Fejlregistreret | only counted | D0130: registered by mistake |
| 9 Afsluttet | closed without being built | inferred: in a November 2026 sample, no status-9 building had a year built |
| 13 Delvis afsluttet | partly completed, may be partly in use | inferred: most have a year built and heating. D0130 does not list this status for buildings, but BBR uses it. |
| 12 Midlertidig afsluttet | status unclear | not defined in D0130, and mixed in the data |

## Setup

1. Create an IT-system with an API key in [Datafordeler Administration](https://portal.datafordeler.dk). A new key
   takes about 15 minutes to become active.
2. Install [uv](https://docs.astral.sh/uv/).
3. Add the server to your MCP client, for example Claude Code:

```sh
claude mcp add datafordeler -e DATAFORDELER_API_KEY=<key> -- \
  uvx --from git+https://github.com/JulianUmbhau/datafordeler-mcp datafordeler-mcp
```

Or in a JSON client config:

```json
{
  "mcpServers": {
    "datafordeler": {
      "command": "uvx",
      "args": ["--from", "git+https://github.com/JulianUmbhau/datafordeler-mcp", "datafordeler-mcp"],
      "env": { "DATAFORDELER_API_KEY": "<key>" }
    }
  }
}
```

| Environment variable | |
|---|---|
| `DATAFORDELER_API_KEY` | required for the BBR tools |
| `ADRESSEVAELGER_TOKEN` | optional. Adressevælger has no user accounts yet and asks everyone to send `adressevaelger123`, the default. Set your own token once accounts arrive. |

`datafordeler-mcp --self-test` checks the installation offline: the tools are registered and the code lists translate.

## Development

```sh
uv sync
uv run ruff check
uv run ruff format --check
uv run pytest                      # offline, HTTP mocked
cp .example.env .env               # then fill in DATAFORDELER_API_KEY
set -a; . ./.env; set +a; uv run pytest -m live
uv run scripts/update_codes.py     # refresh src/datafordeler_mcp/codes.json from teknik.bbr.dk
```

## Data and licences

The code is MIT-licensed. The data keeps its own terms:

- **BBR:** Bygnings- og Boligregistret (BBR), [CC BY 4.0](https://datafordeler.dk/vejledning/brugervilkaar/bygnings-og-boligregistret-bbr/).
  The code lists come from [teknik.bbr.dk/kodelister](https://teknik.bbr.dk/kodelister).
- **DAR:** Danmarks Adresseregister, Klimadatastyrelsen, [CC BY 4.0](https://datafordeler.dk/vejledning/brugervilkaar/danmarks-adresseregister-dar/).

Credit these sources when you share results. Every tool result carries a `source` line for this.

## Related

[boligmcp](https://github.com/flyngaa/boligmcp) (MIT) is a much broader property-report MCP server over the same
services: ownership, valuation, sales, plans and more. This project covers only addresses, buildings and units.
