---
name: bench-protocol
description: Fair benchmark protocol for arrowtap — metric definitions, baselines, results JSON schema, and how to add a client or format. Use when editing bench/, src/arrowtap/bench/, report.py, or writing results text.
---

# Benchmark protocol

## Metric definitions (measure exactly these)

| Metric | Start | Stop |
|---|---|---|
| `bytes` | — | Bytes received on the wire (after HTTP content-encoding) |
| `ttfr_ms` | Request sent | First row usable in client memory (a typed column value readable) |
| `ttq_ms` | Request sent | Full result queryable: `SELECT count(*), avg(<double col>)` returns in the target client |
| `longtask_ms` | Request sent | Sum of `PerformanceObserver('longtask')` durations (browser only) |
| `peak_rss_mb` | — | Peak RSS delta during the run (Python only, `resource`/`psutil`) |

`ttq_ms` is the headline. It includes the query in the client so formats cannot hide decode cost behind lazy views.

## Fairness rules

1. Baseline is VOTable BINARY2 **with gzip**. Also report BINARY2 without gzip. Never headline against TABLEDATA.
2. Same machine, same server process, same query, same columns, same row order.
3. One warm-up run, then N=5. Report median and p90. Discard nothing else.
4. Server-side serialize time is logged separately and not subtracted.
5. The JS VOTable decoder is ours and tuned; results text must say it is a best-case VOTable baseline.
6. Localhost hides transfer cost: also report `bytes` and a modelled transfer time at 100 Mbit/s. Emmanuel runs one throttled browser pass for real numbers.
7. Parquet cannot have a `ttfr` earlier than its full download; record it as such, not as missing.
8. Arrow+LZ4 in the browser lane only if Gate 0 proved Arrow JS decodes it.

## Client × format matrix

| Client | VOTable | Parquet | Arrow |
|---|---|---|---|
| Arrow JS | JS BINARY2 decoder → Arrow table | parquet-wasm (only if Gate 0 picks it) else skip | `RecordBatchReader` on fetch stream |
| DuckDB-WASM | decoder → `insertArrowTable` | `registerFileBuffer` + `CREATE TABLE AS read_parquet` | `insertArrowFromIPCStream` |
| pyarrow | astropy parse → `Table.to_pandas` → pyarrow | `pq.read_table` | `ipc.open_stream` |
| Polars | via pyarrow table | `pl.read_parquet` | `pl.read_ipc_stream` |
| astropy | `votable.parse` | `Table.read(format="parquet")` | via pyarrow → `Table` |

## Results JSON (one file per run batch)

```json
{"lane": "py|web", "host": "...", "versions": {"duckdb": "...", "pyarrow": "..."},
 "runs": [{"client": "polars", "format": "arrow+lz4", "rows": 1000000,
           "bytes": 0, "ttfr_ms": [..5], "ttq_ms": [..5], "longtask_ms": null, "peak_rss_mb": 0}]}
```

`report.py` renders: one table per lane (rows × format, cells = median ttq_ms), one bytes table, one ratio table vs baseline, and a chart (PNG via matplotlib) per lane.

## Adding a format or client

1. Add it to the matrix above with a one-line rationale.
2. Add it to the equivalence test first.
3. Then to the bench. Never the other way round.
