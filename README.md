# Public Records Gym — sample

A work sample from a private RL environment. Two train tasks, their answer keys, the
grader, and the tool contract. The frozen snapshot, the held-out test split and the task
templates are not here; they are the product.

**What it is.** A tool-use environment. An agent answers questions about Minnesota land,
buildings and businesses by querying seven joined public-record layers (parcels, recorded
sales, building permits, business licences, code and food inspections) through an
ArcGIS-style API. Every answer key is computed from the frozen snapshot by SQL and graded
without a model.

## At a glance

| | |
|---|---|
| Tasks | 25 (15 train, 10 held-out test), 6 templates |
| Baseline, held-out test, with tools | claude-sonnet **4/10 (40%)**, mean 42 tool calls per episode |
| Without tools | claude-sonnet passes **0 of 51** candidates |
| Answer keys | computed by SQL, then re-derived through the agent's own tools: 51/51 reproduced |
| Tool calls per solve | 4–16 minimum on the reference trajectory; chains need 8+ |
| Grader | deterministic: `exact`, `set_f1`, `map_f1`, `numeric_tol`; pass = reward 1.0 |
| Harness | 7 tools over MCP stdio, or plain Python |
| Snapshot | 1.97M rows, ~43 MB zstd Parquet, frozen 2026-10-15 |

## The lesson that set the difficulty

Round 1: seven spatial templates. Sonnet passed 10/10 with tools. Retired.

Round 2: traps written into each layer's notes. Sonnet read the notes first and passed
9/10. Retired.

Round 3: traps that are true in the rows and written nowhere. Sonnet finds them only by
looking. 4/10. A trap the docs disclose is not a trap.

| template | the trap |
|---|---|
| `ratio_parcels_assessed` | the sales layer carries its own `assessed` column; on 333 rows it disagrees with the parcels layer the question asks for. Carver and Scott counties share 2,582 parcel ids, so a join on id alone pulls the other county's twin |
| `permits_30_days` | 34 Saint Paul permits are completed *before* they were issued. "Within 30 days" is 0–30; `<= 30` counts the negatives |
| `food_code_overlap` | parcels in both a food-inspection slice and a code-inspection slice; some food rows have no parcel id |
| `fees_two_cities` | Minneapolis repeats the fee on every parcel row of a permit; >2,000 permits to page; Saint Paul publishes no fee at all |
| `sales_ratio_median` | multi-parcel deeds carry the whole deal's price on each parcel; the flag is true-or-null |
| `chain` | 8+ dependent hops from a messy register address to distinct code inspections |

## The two tasks here

`tasks/sample.jsonl` holds the prompts; `tasks/sample_keys.jsonl` the keys, with the
intermediate the naive path lands on.

**prg-cbeb9c48f9** (tier 4). Total permit fees on building permits issued August 2024,
Minneapolis and Saint Paul separately. Key: `{"Minneapolis": "1551568", "Saint Paul":
"NOT_OBSERVED"}`. Summing rows gives $3,064,994; the right sum is over 2,904 distinct
permits, which means paging past the 2,000-row cap. Saint Paul publishes no fee, and "0"
is wrong.

**prg-4573caffa7** (tier 3). Saint Paul Mechanical Permits issued in 2021 completed within
30 days of issue. Key: `1361`. The naive `<= 30` gives 1,364 because three were completed
before they were issued.

## Tools

See `TOOLS.md`. `query_layer` returns at most 2,000 rows per call (ArcGIS's default
`maxRecordCount`) with a `more` flag; there is no GROUP BY or SUM, so aggregates go through
`calc`. Parcel ids are unique only within a county.

## Grader

`gym/grader.py`. No model in the loop. `map_f1` scores each part of a multi-part answer so a
model that abstains everywhere earns partial credit on the observed parts and never a
pass; that is why every abstention task mixes observed and unobserved parts.

## Licence

Sample files: free for research. The full environment is licensed for commercial use on
request.

Aidan Jude · aidan@brickandmortar.dev · https://randomaidan.substack.com
