---
name: votable-arrow-mapping
description: Rules for mapping VOTable FIELDs and TAP status to Arrow IPC schema/field/batch metadata, and for the streamed BINARY2 writer. Use when editing src/arrowtap/formats/, adding a column type, or touching stream termination or error handling.
---

# VOTable ↔ Arrow mapping

## Type mapping (in scope)

| VOTable datatype / arraysize | Arrow type | Notes |
|---|---|---|
| `boolean` | `bool` | |
| `unsignedByte` | `uint8` | |
| `short` | `int16` | |
| `int` | `int32` | |
| `long` | `int64` | |
| `float` | `float32` | NaN stays NaN, never null |
| `double` | `float64` | NaN stays NaN, never null |
| `char` / `unicodeChar`, `arraysize="*"` | `utf8` | |
| `char`, `arraysize="n"` | `utf8` + `votable:arraysize=n` | Pad/truncate only in the VOTable writer |

Arrow → VOTable for types VOTable lacks: `uint16`→`int`, `uint32`→`long`, `uint64`→`long` (raise on overflow), `large_utf8`→`char *`. Anything else (lists, structs, dictionaries, decimals, timestamps) raises `UnsupportedTypeError` in v1.

Always write `votable:datatype` so the reverse mapping is exact (e.g. `char` vs `unicodeChar`).

## Nulls

- Arrow validity bitmap ↔ BINARY2 per-row null mask. Same semantics, no sentinels.
- Do not emit `<VALUES null=...>` in BINARY2 output.
- Reading TABLEDATA/BINARY inputs with `null` sentinels: convert sentinel to null on read.

## Metadata keys

Schema: `tap:query`, `tap:lang`, `tap:service`, `tap:table`, `tap:maxrec`, `tap:precision`, `tap:warnings` (JSON array).
Field: `votable:ucd`, `votable:unit`, `votable:description`, `votable:utype`, `votable:xtype`, `votable:datatype`, `votable:arraysize`, `votable:ID`.
Absent attribute → absent key. Never write empty strings.

## Termination

- Last IPC message: zero-row batch, same schema, `custom_metadata` = `tap:query_status` (`OK`|`OVERFLOW`|`ERROR`), `tap:rows`, `tap:message` (ERROR only).
- OVERFLOW: emit when `MAXREC` rows sent and the reader still had rows. Read one row ahead; do not count all rows.
- Exception after the first byte: catch, write the ERROR terminal batch, close. Never let the connection reset silently.
- Exception before the first byte: VOTable error document, HTTP 400.
- VOTable stream: trailing `<INFO name="QUERY_STATUS" value="OK|OVERFLOW|ERROR"/>` after `</TABLE>`, inside `<RESOURCE>`.

## Streamed BINARY2 writer

- Header (RESOURCE, INFO, TABLE, FIELDs, `<DATA><BINARY2><STREAM encoding="base64">`) first, then rows.
- Encode each batch to big-endian row bytes with numpy (structured dtype + null mask bytes per row).
- Base64 boundary rule: carry the remainder so each emitted chunk encodes a multiple of 3 bytes; flush the carry at the end.
- Strings: `char *` → 4-byte big-endian length + bytes; `unicodeChar` uses UCS-2 big-endian per spec.
- Test: parse the output with `astropy.io.votable` after 1, 2 and N batches (truncate and close tags to test partial streams).

## Pitfalls

- pyarrow per-batch custom metadata needs `RecordBatchStreamWriter.write_batch(batch, custom_metadata=...)`; confirm in Gate 0 for the pinned version.
- Compression is per buffer in IPC (`IpcWriteOptions(compression=...)`); it is not HTTP `Content-Encoding`. Keep both options distinct in code and in results.
- DuckDB `fetch_record_batch` may return batches smaller than requested; never assume fixed batch size.
