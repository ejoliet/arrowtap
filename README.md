# arrowtap

> Three things TAP 1.1 already allows and nobody ships: Arrow IPC responses, answers with error bars, and unit-checked ADQL. One small service plus a benchmark that proves each claim with numbers.

Status: **spec (RDD Type A)**. No code yet. Owner: ejoliet. Date: 2026-10-11.

---

## Purpose

**Problem**

1. VOTable decoding is the slowest step in most browser TAP clients. Parquet is allowed by TAP 1.1 but is not streamable. No TAP service streams Arrow IPC.
2. On large tables every "how many / what's the distribution" question is a full scan. No TAP service gives a fast approximate answer with stated bounds.
3. TAP_SCHEMA carries `unit` and `ucd` per column, but every engine drops them. `ra + pmra` runs without complaint, and computed columns come back with no unit.

**Solution**: one FastAPI mini-TAP over DuckDB that:

- answers `RESPONSEFORMAT=application/vnd.apache.arrow.stream` (IANA-registered media type) as a streamed Arrow IPC response with full VOTable metadata;
- answers `PRECISION=approx` from precomputed mergeable sketches, labelled with bounds, with exact on demand;
- checks units and UCDs at ADQL compile time and propagates units to output columns.

Each claim ships with a benchmark or a test catalogue, and a one-page convention note aimed at IVOA DAL.

**Non-standard parameters** (`PRECISION`, `UNITCHECK`) are service-specific parameters, which TAP 1.1 permits. No spec change is needed for any of the three.

**Who benefits**: browser clients that decode results themselves (DuckDB-WASM tools, Aladin Lite, JS notebooks), Python users on Polars/pyarrow, and archive operators (IRSA, MAST, ESA) evaluating a cheap upgrade. Firefly decodes VOTable server-side, so claim 1 matters less for Firefly; say so in results.

---

## Claims and how each is proven

| # | Claim | Proof artefact | Pass condition |
|---|---|---|---|
| 1 | Arrow IPC reaches "queryable" faster than VOTable BINARY2 (+gzip) and Parquet, and streams | `docs/results.md` from bench matrix | Arrow beats VOTable-BINARY2+gzip on time-to-queryable at 100k and 1M rows in both browser and Python lanes; first rows usable before the response ends |
| 1b | No metadata loss | Round-trip tests | VOTable → Arrow → VOTable keeps every FIELD attribute in scope |
| 4 | Instant previews with honest bounds | Approx test suite + bench page | Exact answer inside reported bounds in ≥ 95% of 100 random partition subsets; approx latency < 50 ms at 100M rows locally |
| 5 | Unit/UCD errors caught before execution, with no false alarms on standard astronomy formulas | `tests/units/catalogue.yaml` | All catalogue cases produce exactly the expected diagnostics; clean cases produce none |

Wording rule for every claim: say **"no decode step"**, never "zero-copy end to end". DuckDB-WASM copies Arrow into its own storage on insert.

---

## Architecture

```
            data/ (Parquet, partitioned by HEALPix order 2 from source_id)
              │                         │
              │                  sketches.parquet (per file × column: CPC, KLL, min/max, rows)
              ▼                         ▼
   ┌──────────────────── FastAPI mini-TAP ─────────────────────┐
   │ /tap/sync  ── ADQL ─► units.check ─► compile ─► approx.plan│
   │                               │                │          │
   │                               ▼                ▼          │
   │                         DuckDB (exact)   sketch store     │
   │                               │ RecordBatchReader          │
   │                               ▼                            │
   │        formats: arrow_ipc | votable_b2 (streamed) | parquet│
   │ /tap/capabilities /tap/tables /tap/availability           │
   │ /arrowtap/check  /arrowtap/complete                       │
   └───────────────────────────────────────────────────────────┘
              │ HTTP (CORS for bench page)
   ┌──────────┴──────────┐
   bench/web (static)     arrowtap bench py
   Arrow JS + DuckDB-WASM httpx + pyarrow / Polars / astropy
              │                      │
              └──── results/*.json ──┴──► arrowtap report ─► docs/results.md
```

| Component | Owns |
|---|---|
| `schema` | Table and column specs (name, type, unit, ucd, description) loaded from `tap_schema.yaml`; single source for VOSI, VOTable FIELDs and Arrow field metadata |
| `compile` | ADQL → DuckDB SQL. Uses `adqlduck` if importable, else a minimal built-in subset (see Open Questions) |
| `formats` | MIME negotiation and three writers sharing one batch iterator. Server-free; installable as `arrowtap[formats]` for tapdrop |
| `units` | VOUnit algebra and UCD rules over the ADQL AST; output column units |
| `approx` | Query-shape classifier, sketch store, bounded answers |
| `server` | TAP sync, VOSI, error mapping, CORS |
| `bench` | Python bench, static browser bench page, Node gate scripts, report generator |

---

## Recommended Stack

Versions are pinned in Phase 0 and recorded in `docs/decisions.md`. Pick releases at least 14 days old.

| Layer | Chosen | Why | Rejected |
|---|---|---|---|
| HTTP | FastAPI + uvicorn | `StreamingResponse`, async, matches house stack | Starlette bare (same thing, less docs), Flask (no async streaming) |
| Engine | DuckDB (Python) | `fetch_record_batch` gives an Arrow `RecordBatchReader`; datasketches community extension | Polars SQL (no sketch BLOBs), DataFusion (heavier for an MVP) |
| Arrow | pyarrow | IPC stream writer with buffer compression and per-batch custom metadata | nanoarrow (fewer conveniences) |
| VOTable reference | astropy.io.votable | Ground truth for parsing in tests | pyvo only (wraps astropy anyway) |
| Units | astropy.units (`format="vounit"`) | Mature VOUnit parser and algebra | pint (no VOUnit grammar) |
| ADQL parse | adqlduck AST; fallback sqlglot (`tsql` dialect for `TOP`) | Reuse; fallback keeps arrowtap standalone | Hand-written parser |
| Sketches | DuckDB `datasketches` community extension (Query.Farm) | Serializable sketch BLOBs, union aggregates, bounds | DuckDB built-in `approx_count_distinct` (no state, still scans) |
| Browser | apache-arrow (JS), @duckdb/duckdb-wasm, from jsDelivr, pinned | The clients the claim is about | — |
| Python clients | pyarrow, Polars, astropy | Three realistic consumers | — |
| Packaging | uv, `uvx --from git+https://github.com/ejoliet/arrowtap arrowtap` | House rule | pip-only |

---

## Repository Layout

```
arrowtap/
├── src/arrowtap/
│   ├── cli.py                 # typer: data, serve, check, bench, report
│   ├── config.py              # pydantic-settings, env prefix ARROWTAP_
│   ├── schema.py              # TableSpec/ColumnSpec; loads tap_schema.yaml
│   ├── tap_schema.yaml        # Gaia-like table: types, units, UCDs, descriptions
│   ├── compile.py             # adqlduck adapter + minimal fallback
│   ├── data/                  # synth.py, fetch.py, partition.py, sketches.py
│   ├── formats/               # negotiate.py, metadata.py, arrow_ipc.py, votable_b2.py, parquet.py
│   ├── units/                 # checker.py, rules.py, ucd.py, diagnostics.py
│   ├── approx/                # classify.py, store.py, answer.py
│   ├── server/                # app.py, tap.py, vosi.py, errors.py
│   ├── bench/py_bench.py
│   └── report.py
├── bench/
│   ├── web/                   # index.html, bench.js, votable.js (static, no build)
│   └── node/gate0.mjs         # Gate-0 checks runnable by the agent (Node)
├── scripts/gate0.py           # Gate-0 checks: pyarrow batch metadata, datasketches functions
├── tests/                     # see Testing
│   └── units/catalogue.yaml
├── docs/
│   ├── convention.md          # Phase 5 deliverable (the DAL note)
│   ├── decisions.md           # pinned versions, Gate-0 outcomes
│   ├── results.md             # generated
│   └── handoff/               # adqlduck.md, tapdrop.md
├── .claude/skills/            # local agent skills
├── AGENTS.md                  # CLAUDE.md is a symlink to this
├── Dockerfile
├── docker-compose.yml
├── Makefile
└── pyproject.toml
```

---

## Data: table `gaia` (`tap_schema.yaml`)

Synthetic data mimics Gaia DR3 columns, units, and UCDs. Real data (`data fetch`) maps onto the same spec. Units are VOUnit.

| Column | Type | Unit | UCD | Nulls |
|---|---|---|---|---|
| `source_id` | long | | `meta.id;meta.main` | no |
| `hpx2` | int | | `pos.healpix` | no (derived) |
| `designation` | char * | | `meta.id` | no |
| `ra`, `dec` | double | deg | `pos.eq.ra;meta.main`, `pos.eq.dec;meta.main` | no |
| `l`, `b` | double | deg | `pos.galactic.lon`, `pos.galactic.lat` | no |
| `pos_err_arcsec` | float | arcsec | `stat.error;pos` | no (synthetic only, for unit tests) |
| `parallax` | double | mas | `pos.parallax.trig` | ~20% |
| `pmra`, `pmdec` | double | mas/yr | `pos.pm;pos.eq.ra`, `pos.pm;pos.eq.dec` | ~20% |
| `phot_g_mean_mag` | float | mag | `phot.mag;em.opt` | ~1% |
| `phot_bp_mean_mag`, `phot_rp_mean_mag` | float | mag | `phot.mag;em.opt.B`, `phot.mag;em.opt.R` | ~10% |
| `bp_rp` | float | mag | `phot.color` | ~10% |
| `radial_velocity` | float | km/s | `spect.dopplerVeloc.opt` | ~95% |
| `ruwe` | float | | `stat.value` | ~20% |
| `has_xp_continuous` | boolean | | `meta.code` | no |

Null fractions approximate DR3 so null-handling costs show up in the benchmark.

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Python | 3.11+ | |
| uv | latest | |
| Node | 20+ | Gate-0 scripts only |
| Docker | 24+ | Optional, for `docker compose up` |
| Chrome or Firefox | current | Browser lane, run by Emmanuel |

No credentials. Public data only. Tests never touch the network.

---

## Quick Start

```bash
# Run from GitHub, no clone
uvx --from git+https://github.com/ejoliet/arrowtap arrowtap data synth --rows 1000000
uvx --from git+https://github.com/ejoliet/arrowtap arrowtap data sketch
uvx --from git+https://github.com/ejoliet/arrowtap arrowtap serve       # :8000

# Try it
curl -s 'http://localhost:8000/tap/sync' \
  --data-urlencode 'REQUEST=doQuery' --data-urlencode 'LANG=ADQL' \
  --data-urlencode 'QUERY=SELECT TOP 5 source_id, ra, dec, parallax FROM gaia' \
  --data-urlencode 'RESPONSEFORMAT=application/vnd.apache.arrow.stream' \
  -o out.arrows
python -c "import pyarrow.ipc as i; print(i.open_stream('out.arrows').read_all().schema)"

uvx --from git+https://github.com/ejoliet/arrowtap arrowtap check "SELECT ra + pmra FROM gaia"

# Whole stack
docker compose up          # server :8000, bench page :8080
```

---

## Configuration Reference

| Variable | Type | Default | Description |
|---|---|---|---|
| `ARROWTAP_DATA_DIR` | path | `./data` | Partitioned Parquet + `sketches.parquet` |
| `ARROWTAP_HOST` | str | `127.0.0.1` | Bind address (`0.0.0.0` in Docker) |
| `ARROWTAP_PORT` | int | `8000` | |
| `ARROWTAP_BATCH_ROWS` | int | `65536` | Rows per streamed batch, all formats |
| `ARROWTAP_MAXREC_DEFAULT` | int | `10000000` | Default `MAXREC` |
| `ARROWTAP_ARROW_COMPRESSION` | `none`\|`lz4`\|`zstd` | `none` | Default IPC buffer compression; overridable per request |
| `ARROWTAP_UNITCHECK` | `off`\|`warn`\|`error` | `error` | Default unit checking mode |
| `ARROWTAP_CORS_ORIGINS` | csv | `http://localhost:8080` | Bench page origin |
| `ARROWTAP_ADQL_BACKEND` | `auto`\|`adqlduck`\|`builtin` | `auto` | Compiler choice |
| `LOG_LEVEL` | str | `INFO` | |

---

## Interface Contract

### TAP sync

`GET|POST /tap/sync` with `REQUEST=doQuery`, `LANG=ADQL`, `QUERY`, optional `MAXREC`, `RESPONSEFORMAT`, plus service-specific `PRECISION`, `UNITCHECK`, `ARROW_COMPRESSION`.

| `RESPONSEFORMAT` value (case-insensitive) | Aliases | Content-Type | Streamed |
|---|---|---|---|
| `application/x-votable+xml;serialization=BINARY2` | `votable`, `votable/b2` | `application/x-votable+xml` | yes |
| `application/vnd.apache.parquet` | `parquet` | same | no (footer at end) |
| `application/vnd.apache.arrow.stream` | `arrow`, `arrows` | same | yes |

Default is VOTable BINARY2. Unknown value → TAP error. HTTP `Accept-Encoding: gzip` honoured for VOTable and Arrow (bench uses it explicitly).

> ⚠️ Verify the Parquet media type string against the IANA registry in Phase 0. If unregistered, keep the alias only and record that in `docs/decisions.md`.

### Errors

- **Before the first byte**: standard TAP error document (VOTable, `QUERY_STATUS=ERROR`), HTTP 400, whatever `RESPONSEFORMAT` was asked. This keeps every existing client working.
- **After streaming started**: format-native terminal signal (Arrow: terminal batch; VOTable: trailing `INFO name="QUERY_STATUS" value="ERROR"`). HTTP status is already 200.

### Arrow metadata convention

Schema-level metadata (all UTF-8 strings):

| Key | Example |
|---|---|
| `tap:query` | the ADQL as received |
| `tap:lang` | `ADQL` |
| `tap:service` | base URL |
| `tap:table` | primary table name when single-table |
| `tap:maxrec` | `1000000` |
| `tap:precision` | `exact` or `approx` |
| `tap:warnings` | JSON array of diagnostics (units, approx fallback) |

Field-level metadata: `votable:ucd`, `votable:unit`, `votable:description`, `votable:utype`, `votable:xtype`, `votable:datatype`, `votable:arraysize`, `votable:ID`. Units use VOUnit strings. Computed columns carry the propagated unit and `votable:ucd` only when the rules can derive one.

Stream termination: the last message is a **zero-row record batch** with custom metadata `tap:query_status` (`OK`|`OVERFLOW`|`ERROR`), `tap:rows`, and `tap:message` on error. Clients that cannot read per-batch metadata fall back to `rows == tap:maxrec` ⇒ possible overflow (same heuristic VOTable clients use).

Full type mapping and edge cases: `.claude/skills/votable-arrow-mapping/SKILL.md`.

### Approximate answers (`PRECISION=approx`)

Answerable shapes (anything else runs exact and adds a `tap:warnings` entry `A001 approx not applicable: <reason>`):

| Query shape | Method | Bounds reported |
|---|---|---|
| `COUNT(*)` | Parquet footer row counts | exact |
| `MIN(c)`, `MAX(c)` | footer statistics | exact |
| `COUNT(DISTINCT c)` | CPC sketch union | `arrowtap:lower`, `arrowtap:upper` at 2σ |
| `arrowtap_quantile(c, q)` | KLL union | `arrowtap:rank_error` |
| `arrowtap_histogram(c, lo, hi, n)` | KLL CDF at bin edges | per-bin count bounds from rank error |

`WHERE` may only use the partition column `hpx2` (`=`, `IN`, `BETWEEN`); `GROUP BY hpx2` is allowed. `arrowtap_quantile` and `arrowtap_histogram` are declared as UDFs in VOSI capabilities and also run exactly (DuckDB `quantile_cont`, `histogram`) under `PRECISION=exact`.

### Unit checking (`UNITCHECK`)

`error` blocks on `U*` codes, `warn` turns them into warnings, `off` skips. `W*` codes never block. Diagnostics appear in `tap:warnings` (Arrow) or `INFO` elements (VOTable). Codes and rules: `.claude/skills/vounit-checker/SKILL.md`.

`POST /arrowtap/check` body `{"query": "..."}` → `{"diagnostics": [...], "columns": [{"name","unit","ucd"}]}`.
`GET /arrowtap/complete?ucd=pos.eq.ra&table=gaia` → matching columns, for autocomplete.

### CLI

```
arrowtap data synth --rows N [--seed 42]     # Gaia-like synthetic, partitioned
arrowtap data fetch --url URL [--rows N]     # real Gaia DR3 subset (Emmanuel supplies URL)
arrowtap data sketch                         # build sketches.parquet
arrowtap serve
arrowtap check "ADQL"
arrowtap bench py [--sizes 1e4,1e5,1e6] [--runs 5]
arrowtap report                              # results/*.json → docs/results.md
```

---

## Benchmark design

Full protocol: `.claude/skills/bench-protocol/SKILL.md`. Summary:

| Axis | Values |
|---|---|
| Sizes | 10k, 100k, 1M rows (≈ 16 columns); 100M synthetic for approx only |
| Formats | VOTable BINARY2, VOTable BINARY2 + gzip, Parquet (zstd), Arrow, Arrow + LZ4, Arrow + gzip (HTTP) |
| Browser clients | Arrow JS table, DuckDB-WASM table |
| Python clients | pyarrow, Polars, astropy Table |
| Metrics | bytes on wire, time-to-first-row, time-to-queryable, main-thread long-task ms (browser), peak RSS (Python) |
| Runs | 1 warm-up + 5, report median and p90 |

The VOTable JS baseline is a minimal BINARY2 decoder written for this repo and tuned for speed. That makes VOTable look as good as it can; say so in results.

---

## Error Handling

| Class | Raised when | Maps to |
|---|---|---|
| `AdqlSyntaxError` | Parse fails | TAP error, 400 |
| `UnitCheckError` | `U*` code under `UNITCHECK=error` | TAP error, 400, message lists codes |
| `UnsupportedFormatError` | Unknown `RESPONSEFORMAT` | TAP error, 400 |
| `MaxrecOverflow` | Row count reaches `MAXREC` with more available | Terminal `OVERFLOW`, not an error |
| `StreamAbort` | Exception after first byte | Terminal `ERROR` batch or trailing INFO |
| `SketchStoreMissing` | `PRECISION=approx` with no sketches | Exact run + warning `A002` |

No retries server-side. Log every request as one JSON line: query hash, format, rows, bytes, duration, precision, diagnostic codes.

---

## Testing

| Suite | Covers |
|---|---|
| `tests/formats/test_roundtrip.py` | VOTable → Arrow → VOTable metadata and values; nulls, NaN, strings, booleans |
| `tests/formats/test_equivalence.py` | Same query in all formats gives identical values (via DuckDB `EXCEPT ALL` both ways) |
| `tests/formats/test_votable_stream.py` | Streamed BINARY2 parses in astropy at every batch boundary; base64 chunking at 3-byte multiples |
| `tests/formats/test_termination.py` | `OK`/`OVERFLOW`/`ERROR` terminal batch; pre-stream errors are VOTable |
| `tests/units/test_catalogue.py` | Every case in `catalogue.yaml` yields exactly its expected codes |
| `tests/units/test_propagation.py` | Output units for computed columns |
| `tests/approx/test_bounds.py` | 100 random `hpx2` subsets: exact inside bounds ≥ 95% |
| `tests/approx/test_classify.py` | Shapes accepted and rejected with the right `A*` code |
| `tests/server/test_tap.py` | Sync, VOSI documents valid against schema, CORS headers |

Fixtures use `arrowtap data synth --rows 20000 --seed 1` into a temp dir. No network.

---

## Non-Goals (v1)

- Async TAP / UWS.
- ADQL geometry beyond what the chosen compiler backend already supports. The built-in fallback supports none.
- Array, complex, and bit columns; Arrow dictionary and nested types.
- Sampling tier for approx (filtered queries on non-partition columns).
- Iceberg tables and puffin files. Sketches live in a plain Parquet sidecar.
- A DuckDB extension for units. The checker runs at ADQL compile time.
- Authentication, multi-user, production deployment.
- Firefly integration.

---

## Open Questions

| # | Question | Default if unanswered |
|---|---|---|
| 1 | Is `adqlduck` importable from GitHub today, with an AST API? | `ARROWTAP_ADQL_BACKEND=auto` falls back to built-in (sqlglot, `SELECT`/`TOP`/`WHERE`/`GROUP BY`/`ORDER BY`, no geometry) |
| 2 | Real Gaia DR3 source URL for `data fetch` | Bench runs on synthetic; real-data run added when Emmanuel supplies the URL |
| 3 | Does the pinned Arrow JS read LZ4/ZSTD-compressed IPC bodies? (Gate 0) | If not, browser lane drops Arrow+LZ4 and uses Arrow + HTTP gzip; LZ4 stays in Python lane |
| 4 | Do Arrow JS and DuckDB-WASM expose per-batch custom metadata? (Gate 0) | If not, browser clients use the `tap:maxrec` heuristic; convention note says so |
| 5 | Does DuckDB-WASM ingest an IPC stream incrementally? (Gate 0) | If not, time-to-first-row in the DuckDB lane is measured via Arrow JS batches |
| 6 | Exact sketch function names in the pinned datasketches extension (Gate 0) | Record in `docs/decisions.md` |

---

## Agent Build Instructions

> Implement end-to-end from this README, `AGENTS.md`, and the skills in `.claude/skills/`. Do not run browsers, push to GitHub, or open PRs. Where a check needs a browser, write the page and print the exact command for Emmanuel.

### Build order

| Phase | Deliverable | Done when |
|---|---|---|
| 0 Gate | `bench/node/gate0.mjs`, `scripts/gate0.py`, `docs/decisions.md` | Open Questions 3–6 answered with evidence; versions pinned |
| 1 Data | `data synth`, `partition`, `tap_schema.yaml` | 1M rows in ~48 `hpx2` files; schema loads |
| 2 Formats | three writers + metadata mapping | Round-trip, equivalence, stream, termination suites pass |
| 3 Server | `/tap/sync`, VOSI, errors, CORS | `test_tap.py` passes; curl Quick Start works |
| 4 Bench | `bench py`, `bench/web`, `report` | `results/py-*.json` produced; page prints the command for Emmanuel |
| 5 Units | checker, propagation, `/arrowtap/check`, `complete`, CLI `check` | Catalogue and propagation suites pass |
| 6 Approx | `data sketch`, classifier, answers, bench page approx/exact toggle | Bounds suite passes; 100M synthetic approx < 50 ms |
| 7 Docs | `docs/convention.md`, `docs/results.md`, handoff notes updated | Each claim in the table above has its artefact linked |

### Acceptance Criteria

- [ ] `make test` and `make lint` (ruff, mypy) pass; coverage ≥ 80% on `formats`, `units`, `approx`.
- [ ] Equivalence test passes for every format at 20k rows.
- [ ] Metadata round-trip loses nothing in scope.
- [ ] `uvx --from git+https://github.com/ejoliet/arrowtap arrowtap --help` works from a clean machine.
- [ ] `docker compose up` serves API on :8000 and bench page on :8080.
- [ ] `docs/results.md` has the full matrix for the Python lane; browser lane rows filled after Emmanuel's run.
- [ ] Every Open Question resolved in `docs/decisions.md` or moved to Non-Goals.

---

## Commands for Emmanuel (checks the agent does not run)

```bash
# Browser lane
docker compose up -d
open http://localhost:8080/?sizes=1e4,1e5,1e6&runs=5     # click Run; it downloads results/web-<ts>.json
mv ~/Downloads/web-*.json results/ && uvx --from . arrowtap report

# Gate-0 confirmation in a real browser
open http://localhost:8080/gate0.html

# Throttled run (Chrome DevTools → Network → "Fast 4G"), then repeat the Run
```

---

## Next Steps

1. Agent: Phase 0 gate; report outcomes before Phase 1.
2. Emmanuel: answer Open Questions 1 and 2.
3. Agent: Phases 1–4 (claim 1 complete and reviewable on its own).
4. Agent: Phase 5 (units), Phase 6 (approx), Phase 7 docs.
5. Apply `docs/handoff/adqlduck.md` and `docs/handoff/tapdrop.md` once their gates pass.
