# @pipeworx/fjc-case-outcomes

Federal caseload and case-outcome statistics — fleet #3271. Complements the
`court-listener` pack: court-listener finds and reads individual dockets and
opinions; this pack answers aggregate statistical questions court-listener
has no data for ("how many civil rights cases were filed in SDNY in FY2023,
and how did they terminate") plus a direct docket+district outcome lookup.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1847+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Status (2026-10-19): promoted, data loading, tool-examples pending

Migration 264 is applied and `fjc-idb-refresh.yml` has run for real (run
39288114956): **FY2023-24 is loaded** — civil 690,312 rows, criminal
435,234 rows, appellate 80,722 rows, plus their summary tables. FY2020-22
is loading separately. The pack was promoted out of `_incubator` in this
session, but **`tool-examples.json`, the live smoke payloads, and the
post-deploy re-probe are not yet done** — they need a real call through
the pack against the live backing tables, which needs the backing
database's URL and service-role credential as environment variables (see
the ingest script's own header for the exact names), and this session had
neither (confirmed absent from `.env`, every `.dev.vars`, and the process
environment). Running `node scripts/add-pack.mjs fjc-case-outcomes`
stopped cleanly at step 3/11 ("backing-store credentials not injected").
Whoever has the credential: `node scripts/add-pack.mjs fjc-case-outcomes --from 3`
resumes exactly there and writes `tool-examples.json` from the real calls.

## Sizing and v1 scope (revised 2026-10-19 on PM review)

The FJC publishes **one SAS7BDAT file per fiscal year per case type**, no
query API, no multi-year bulk export. Per-year file naming was confirmed
live for FY2021-FY2023 (`cv23.sas7bdat`, `cr23_0.sas7bdat`,
`ap23.sas7bdat` — see `scripts/ingest-fjc-idb.mjs`'s header for the
filename-probing logic, since the suffix is inconsistent and undocumented).

**v1 scope: civil + criminal only, FY2020-FY2024 (5 fiscal years).**
Appellate is included opportunistically where cheap (see sizing below) but
is not the committed scope; bankruptcy is out entirely (see below).

**Real measured sizing, not an estimate.** FY2023 civil (293,677 rows) and
criminal (218,853 rows) were parsed from the live SAS7BDAT files and
actually loaded into a local Postgres 17 — raw tables, summary tables,
indexes, `ANALYZE` — then measured with `pg_total_relation_size`:

| Table | Table size | Indexes | Total |
|---|---|---|---|
| `fjc_idb_civil` | 44 MB | 21 MB | 65 MB |
| `fjc_idb_criminal` | 27 MB | 14 MB | 41 MB |
| `fjc_idb_civil_summary` | 1.4 MB | 1.6 MB | 2.9 MB |
| `fjc_idb_criminal_summary` | 0.7 MB | 0.9 MB | 1.6 MB |
| **One fiscal year, total** | | | **110 MB** |

Extrapolated linearly to v1 (5 fiscal years, civil+criminal): **~550 MB** —
comfortably under the ~1.5 GB ceiling that would force reconsidering the
store. Caveat: this extrapolates from ONE year (FY2023); case volume does
vary year to year, so actual 5-year size could differ by some margin, but
not by the multiples that would change the conclusion. `EXPLAIN ANALYZE`
on both the lookup query (`district`+`docket`, indexed) and a filtered
summary-table read confirmed index-scan-only, sub-millisecond, zero
sequential scans — see the migration's header for the exact plans.

**Bankruptcy IDB data is NOT in scope for this build.** The FJC has
published bankruptcy case data since 2008, but no `bk##`/`bq##`/`bankr##`
filename (plain or `_0`-suffixed) returned HTTP 200 during this session's
probing — it likely uses a different naming scheme or portal entirely.
Needs separate research before it can be sized; not attempted further here
to keep this build's scope honest rather than guess at a URL.

## Store

**Supabase** (structured relational data, aggregation-heavy queries), not
R2/FTS5 — this is tabular caseload statistics with GROUP BY-shaped
questions (counts by district/year/nature-of-suit/disposition), not text
search. Confirmed viable by the real-load sizing above, not assumed.

Three raw tables (`fjc_idb_civil`, `fjc_idb_criminal`, `fjc_idb_appellate`)
carry only the fields the tools here actually use — no `raw jsonb`
catch-all column (an earlier draft had one; dropped on review, since every
byte paid for in a real Postgres table even when unused is exactly the
cost this revision is supposed to be honest about).

**Aggregates are PRECOMPUTED SUMMARY TABLES, not views.** A plain `GROUP
BY` view over the raw tables looked cheap in the first draft of this
migration, but PostgREST runs under an ~8s `statement_timeout`, and the
`uspto-maintenance-fees` pack (27.5M rows) timed out on every call under
exactly that shape until `count=exact` and full scans were removed.
`fjc_idb_*_summary` is built ONCE PER FISCAL-YEAR LOAD by
`scripts/ingest-fjc-idb.mjs` (it has no direct SQL connection — only the
same PostgREST credential every other pack's ingest uses — so the GROUP BY
runs in Node over the already-parsed rows, not in SQL) and reinserted on
every full-fiscal-year refresh. The aggregate tool reads ONLY these summary
tables, never the raw per-case tables. See the migration's own header
comment for the full design rationale.

## Refresh

**Full fiscal-year replace, not a row upsert — for both the raw table and
its summary table.** The IDB's own quarterly update model only revises
records within a fiscal year still open for reporting (FJC Research
Guide); once a year closes, its file stops changing. A scheduled refresh
therefore only needs to re-pull the current (and, defensively, prior)
fiscal year's files, delete every previously-loaded row for that `(table,
fiscal_year)` — raw AND summary — and reinsert —
`scripts/ingest-fjc-idb.mjs --case-type all --years <current>,<current-1>`.
A full v1 backfill across FY2020-FY2024 is a separate, one-time run of the
same script with a wider `--years` list. `ANALYZE` is not run from the
script (no direct SQL access); Postgres autovacuum picks it up after a
full-year replace touches the whole year's rows — the one-time initial
backfill should have a human run a manual `ANALYZE` via `psql` as its last
step rather than wait on it.

`fjc-idb-refresh.yml` is live at `.github/workflows/fjc-idb-refresh.yml`
(applied by the config-applier, since `.github/workflows/**` is
push-blocked) and has already run once successfully for the data load
(run 39288114956); its smoke step is what will pass once tool-examples.json
exists (see "Status" above).

## Remaining before this fully ships

1. Someone with the backing database's URL and service-role credential runs
   `node scripts/add-pack.mjs fjc-case-outcomes --from 3` — this smoke-tests
   both tools for real against the loaded data, writes
   `tool-examples.json` FROM those calls, and carries through the rest of
   the add-pack checklist (npm-release-status confirmation, deliverable
   entry).
2. Re-probe both tools through the live gateway after the next deploy.
3. `fleet_close` on #3271 with the smoke_payloads / live_probes / etc.
   artifacts the task's acceptance checklist requires.

## Tools

- `fjc_case_outcome_lookup(case_type, district_or_circuit, docket, fiscal_year?, defendant_number?)`
  — one case's/defendant's outcome by docket number + district (or circuit
  for appellate).
- `fjc_case_outcomes_summary(case_type, fiscal_year?, district_or_circuit?, nature_of_suit_code?, offense_code?, disposition_code?, group_by?, limit?)`
  — aggregate counts, total first, broken down by whichever dimensions are
  requested (nature of suit / offense code / disposition / year / district
  or circuit).

## Data sources

- Federal Judicial Center, Integrated Database — civil case, criminal
  defendant, bankruptcy and appellate case filings and terminations in the
  federal courts, reported quarterly by the Administrative Office of the
  U.S. Courts. US federal government data, public domain. Continuous back
  to 1970 (civil/criminal), 1971 (appellate). No query API — bulk files
  only; this pack mirrors a scoped year range into Supabase.

## Traps

**Per-year filename suffix is inconsistent and undocumented.**
`cv23.sas7bdat` (civil) needs no suffix; `cr23_0.sas7bdat` (criminal) does;
`ap23.sas7bdat` (appellate) needs none. The ingest script probes both forms
per year/case-type rather than hardcoding one pattern.

**`pandas.read_sas` returns `NaT` for missing dates, which is NOT caught by
a plain `isinstance(v, float) and math.isnan(v)` check** and fails
`json.dumps` with `TypeError: Object of type NaTType is not JSON
serializable` — hit live on the civil file's `TRANSDAT` column (all-`NaT`
for the vast majority of cases, which were never transferred to another
district). Fixed by checking `pd.isna(v)` first, which covers `None`, float
`NaN`, and `NaT` in one call.

**Criminal defendant names are already redacted by the FJC itself**
(verified empirically: the `NAME` column is 100% null across all 218,853
FY2023 criminal rows), per 1995/2003 Judicial Conference policy — there is
nothing to carry, so the criminal table has no name field at all. Civil
party names (`PLT`/`DEF`) ARE populated and published; stored as-is.

**Fiscal-year boundaries changed in 1992.** Statistical years SY70-SY91 run
July-June; FY1993 onward runs the federal government's October-September
fiscal year; the 1992 file covers a 15-month transition span. A caller
asking for "year 1991" and "year 1993" is asking about non-comparable
12-month windows unless this is accounted for — not yet surfaced in the
tool responses; a documented gap for the promotion step to address.

**Codes are raw FJC codes (nature of suit, disposition, offense, district,
circuit), not yet labelled with human-readable names.** See the migration
header for why a crosswalk was not guessed at, and the planned follow-up
(a small reference table sourced from the FJC's own published codebooks,
e.g. `https://www.fjc.gov/sites/default/files/idb/codebooks/`).

**`(fiscal_year, district, office, docket)` is NOT unique — found loading
the real FY2023 civil file, not assumed.** 293,677 rows resolve to only
290,139 distinct triples (~1.2% collide). The one pair inspected has the
same disposition and nature-of-suit but a different `filedate` (2017 vs.
2023) — likely a reopened or renumbered proceeding; the IDB has no
case-level version column to disambiguate. Criminal hit the same shape
(collision even including `defendant_number`). The raw tables therefore
have plain indexes, not unique constraints, on this tuple — the ingest
script loads every row the FJC publishes rather than deduplicating, and
`fjc_case_outcome_lookup` returns every matching row for a docket.

**`fjc_case_outcomes_summary` requires `fiscal_year` and/or
`district_or_circuit`.** The summary tables are pre-aggregated but an
unfiltered, multi-year query could still carry more distinct groups than
PostgREST's row cap — a real FY2023-only measurement already has 26,067
distinct civil groups. The tool fetches the full matching set at the
summary table's fine grain and re-aggregates client-side to whatever
coarser `group_by` the caller asked for (PostgREST cannot re-`GROUP BY` a
table via a plain `select=`); if the fetch hits the safety cap the response
carries `total_may_be_incomplete: true` rather than a silently wrong total.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "fjc-case-outcomes": {
      "url": "https://gateway.pipeworx.io/fjc-case-outcomes/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/fjc-case-outcomes/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1847+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/fjc_case_outcome_lookup \
  -H 'Content-Type: application/json' \
  -d '{"case_type":"civil","district_or_circuit":"90","docket":"1701877","fiscal_year":2023}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/fjc_case_outcome_lookup`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "fjc-case-outcomes": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-fjc-case-outcomes"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-fjc-case-outcomes
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Fjc Case Outcomes data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
