---
name: approx-sketches
description: How arrowtap builds per-file sketches and answers PRECISION=approx queries with bounds — partitioning, sidecar schema, query-shape classifier, bounds math, and tests. Use when editing src/arrowtap/approx/, data/sketches.py, data/partition.py.
---

# Approximate answers from sketches

## Partitioning

- Column `hpx2` = HEALPix order-2 nested index, derived from Gaia `source_id`: `hpx12 = source_id // 34359738368`, then `hpx2 = hpx12 >> 20` (order 12 → 2 drops 10 levels × 2 bits).
- One Parquet file per `hpx2` value (48 files). Row-group size 128k. zstd.
- Synthetic data must generate `source_id` consistent with `ra, dec` so the derivation holds (use `healpy` or `astropy_healpix` in `data synth`, test the round trip).

## Sidecar: `data/sketches.parquet`

| Column | Type |
|---|---|
| `file` | utf8 (relative path) |
| `hpx2` | int32 |
| `column` | utf8 |
| `kind` | utf8: `cpc`, `kll`, `stats` |
| `blob` | binary (serialized sketch; null for `stats`) |
| `rows`, `nulls` | int64 |
| `min`, `max` | float64 (null for non-numeric) |
| `lg_k` / `k` | int32 sketch parameter |

Build with DuckDB + `datasketches` community extension, one `GROUP BY` per file and column. CPC for every column used in `COUNT(DISTINCT)` demos (`source_id`, a low-cardinality `has_xp`, a binned magnitude). KLL for every float column.

Gate 0 records exact function names (create, union, estimate, bounds, quantile, CDF, rank error) for the pinned extension version. Use those names only through `approx/store.py` so a rename touches one file.

## Classifier (`approx/classify.py`)

Input: AST. Output: `ApproxPlan | NotApplicable(code, reason)`.

Accept only:
- one table, no joins, no subqueries;
- select list of `COUNT(*)`, `MIN/MAX(col)`, `COUNT(DISTINCT col)`, `arrowtap_quantile(col, q)`, `arrowtap_histogram(col, lo, hi, n)`, optionally `hpx2`;
- `WHERE` built only from `hpx2 = k`, `hpx2 IN (...)`, `hpx2 BETWEEN a AND b`, combined with AND/OR;
- `GROUP BY hpx2` or none.

Reject codes: `A001 not applicable: <reason>` (run exact), `A002 sketch store missing` (run exact), `A003 column has no sketch of the needed kind` (run exact).

## Answers and bounds

| Aggregate | Merge | Value | Bounds |
|---|---|---|---|
| `COUNT(*)` | sum `rows` | exact | none (`tap:precision` stays `approx` for the response, field metadata `arrowtap:exact=true`) |
| `MIN/MAX` | min/max of stats | exact | `arrowtap:exact=true` |
| `COUNT(DISTINCT)` | CPC union | estimate | `arrowtap:lower`, `arrowtap:upper` at 2 std dev |
| `arrowtap_quantile` | KLL union | quantile | `arrowtap:rank_error` (normalized) |
| `arrowtap_histogram` | KLL union, CDF at edges × total rows | bin counts | per-bin ± `2 * rank_error * total_rows` |

Histogram returns one row per bin: `bin_lo`, `bin_hi`, `count`, `count_lo`, `count_hi`.

## Tests

- `test_bounds.py`: 100 random subsets of `hpx2` values (seeded). For each aggregate, exact DuckDB answer must sit inside bounds in ≥ 95 cases.
- `test_classify.py`: one accept and one reject case per rule.
- `test_store.py`: serialize → union → estimate equals single-pass sketch over the same rows (within sketch determinism).

## Demo

Bench page toggle "approx then exact": sends `PRECISION=approx`, renders histogram with error bars and latency; then fires the exact query and overlays it. 100M synthetic rows (`data synth --rows 100000000`) is where the latency gap is visible.
