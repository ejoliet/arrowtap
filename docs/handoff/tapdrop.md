# Handoff: arrowtap → tapdrop

Owner: ejoliet · Written: 2026-10-11 · Apply after: arrowtap Phase 3 (formats + server) for items 1–3; after the adqlduck handoff for item 4; item 5 deferred.

## Why

tapdrop is the service people will actually run. arrowtap proves the conventions; tapdrop ships them to users with real catalogs (Parquet/HATS, FITS, CSV/ECSV, VOTable).

## How the code travels

| Option | Pros | Cons |
|---|---|---|
| Depend on `arrowtap.formats` by git tag | One implementation, one test suite | arrowtap must keep `formats` importable without FastAPI/DuckDB |
| Vendor `formats/` into tapdrop | No cross-repo dependency | Two copies drift; convention fixes land twice |

**Recommendation: depend by git tag**, as an extra: `arrowtap[formats] @ git+https://github.com/ejoliet/arrowtap@vX.Y.Z`. Requires an arrowtap invariant: `arrowtap.formats` imports only pyarrow, numpy, astropy. Add a test in arrowtap that imports it in a clean venv without the server deps.

## What moves

| # | Item | tapdrop change | Acceptance |
|---|---|---|---|
| 1 | Arrow IPC response | Accept `RESPONSEFORMAT` values and aliases from arrowtap README; stream via `arrowtap.formats.arrow_ipc` from tapdrop's DuckDB `RecordBatchReader` | curl + pyarrow read matches VOTable values on every tapdrop input format |
| 2 | Streamed BINARY2 | Replace any buffered VOTable writer with `arrowtap.formats.votable_b2` | pyvo and TOPCAT read results unchanged; memory flat at 10M rows |
| 3 | Error and termination semantics | Pre-stream errors as VOTable 400; terminal status batch / trailing INFO; `tap:maxrec` in schema | Overflow and mid-stream error tests from arrowtap pass against tapdrop |
| 4 | VOSI capabilities | List each output format as `<outputFormat><mime>…</mime><alias>…</alias></outputFormat>`; declare UDFs from adqlduck registry | TOPCAT shows the formats; capabilities validate |
| 5 | Units check | Comes from adqlduck `compile()`; expose `UNITCHECK` param, default `warn` in tapdrop | Catalogue cases behave as in arrowtap |
| 6 | Async results | UWS result written as `.arrows` when requested; Content-Type from format | `GET …/results/result` returns a readable stream file |

Default `RESPONSEFORMAT` stays VOTable. Existing clients must see no change.

## Deferred (v1.x, decide after arrowtap results)

- **`PRECISION=approx`** only for HATS inputs: HATS `Norder/Npix` partitions already give the partition column, so `tapdrop --sketch` can build the sidecar at ingest. Not for loose CSV/FITS folders, where there is no natural partitioning.
- **Arrow IPC file format** (`application/vnd.apache.arrow.file`) for async results, which are stored and could benefit from random access. Only if a client asks for it.

## Input metadata to preserve

tapdrop ingests units/UCDs from FITS `TUNITn`, ECSV headers, and VOTable FIELDs. Map them into the same `TapSchema` the formats and checker read. Unit strings from FITS often are not valid VOUnit (`deg.`, `mag(AB)`): normalise with `astropy.units.Unit(s, format="fits")` then `to_string("vounit")`; on failure keep the raw string in `description` and set `unit` to null, log once per column.

## Commands for Emmanuel (after integration)

```bash
uvx --from git+https://github.com/ejoliet/tapdrop tapdrop serve ./catalogs
curl -s http://localhost:8000/tap/sync --data-urlencode REQUEST=doQuery --data-urlencode LANG=ADQL \
  --data-urlencode 'QUERY=SELECT TOP 10 * FROM <table>' \
  --data-urlencode RESPONSEFORMAT=arrow -o t.arrows
python -c "import pyarrow.ipc as i; t=i.open_stream('t.arrows').read_all(); print(t.schema)"
# TOPCAT: File → Load → TAP → localhost:8000/tap → check Output formats list
```
