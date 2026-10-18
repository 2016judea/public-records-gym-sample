# Tool contract

Seven tools, served over MCP stdio (`FastMCP`) or called as plain Python. Every call is
appended to an episode trace (`$GYM_TRACE`, jsonl). All reads are against a frozen,
read-only DuckDB snapshot, so the same input always returns the same output.

| tool | signature | notes |
|---|---|---|
| `list_layers` | `()` | the data layers, their fields and row counts |
| `layer_notes` | `(layer)` | the publisher's read-before-quoting caveats: coverage, what a row is, known traps |
| `query_layer` | `(layer, where="true", fields=None, order_by=None, limit=2000, offset=0, count_only=False, distinct=False)` | ArcGIS-style query. `where` is a SQL boolean over the layer's fields, e.g. `"city = 'Saint Paul' and issued >= DATE '2024-01-01'"`. At most 2,000 rows per call; page with `offset` while `more` is true. `count_only` returns the total; `distinct` returns distinct combinations of `fields`. No GROUP BY or SUM: use `calc` |
| `query_near` | `(layer, lat, lon, radius_m, where="true", fields=None, limit=2000, offset=0, count_only=False, distinct=False)` | rows within `radius_m` metres (haversine) of a point, nearest first, with `distance_m` |
| `geocode` | `(address, city=None)` | resolve a street address to parcel(s) using the parcel file's own situs strings: exact match, then prefix. Unit numbers and spelled-out words will not match; normalise first |
| `parcel_lookup` | `(parcel_id, county=None)` | every parcel carrying this id, with its recorded sale. Ids are unique only within a county |
| `calc` | `(expression)` | arithmetic over numbers and lists: `+ - * / **`, `sum len min max median mean round sorted set`; elementwise `div mul add sub` over two equal-length lists; `intersect union minus distinct` over lists of ids |

Layers in the full environment: `parcels` (7 metro counties), `sales` (assessor's last
sale of record, one row per parcel), `permits` (Minneapolis + Saint Paul), `licences`
(Minneapolis + Saint Paul registers), `code_inspections` and `food_inspections`
(Minneapolis), plus a wage layer. Private-individual contact fields are dropped before the
snapshot is built; a test sweeps every VARCHAR and JSON column.
