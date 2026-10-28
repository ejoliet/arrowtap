# Handoff: arrowtap → adqlduck

Owner: ejoliet · Written: 2026-10-11 · Apply after: arrowtap Phase 5 (units) and Phase 6 (approx) pass on the built-in parser.

## Why

Three things arrowtap prototypes are compiler concerns, not service concerns. Every adqlduck consumer (sidetap, tapdrop, obstap-local, arrowtap) gets them for free once they live in the compiler.

## What moves

| # | Piece | From (arrowtap) | To (adqlduck) | Consumers gain |
|---|---|---|---|---|
| 1 | Output column metadata | `units/checker.py` propagation | `compile()` result | Correct FIELD/Arrow metadata for computed columns everywhere |
| 2 | Unit/UCD checker | `units/` | AST pass `check()` | `UNITCHECK` in every service |
| 3 | Query-shape descriptor | `approx/classify.py` (generic half) | `shape()` | Any service can route aggregates to precomputed answers |
| 4 | UDF registry | hard-coded `arrowtap_*` UDFs | registry + TAPRegExt `<language><feature>` output | Services declare custom functions once |
| 5 | Conformance fixtures | `tests/units/catalogue.yaml` | `tests/conformance/units.yaml` | Shared regression suite |

## What stays in arrowtap

Writers (`formats/`), sketch store and bounds math (`approx/store.py`, `approx/answer.py`), server, bench.

## Proposed interface

```python
@dataclass(frozen=True)
class ColumnMeta:
    name: str
    datatype: str            # VOTable datatype
    arraysize: str | None
    unit: str | None         # VOUnit string, propagated
    ucd: str | None          # derived only when rules allow
    description: str | None

@dataclass(frozen=True)
class Diagnostic:
    code: str                # U001…, W010…; stable
    level: Literal["error", "warning"]
    message: str
    span: tuple[int, int]    # char offsets in the ADQL

@dataclass(frozen=True)
class CompileResult:
    sql: str                 # DuckDB SQL
    columns: list[ColumnMeta]
    diagnostics: list[Diagnostic]
    shape: "QueryShape"

def compile(adql: str, schema: TapSchema, *, unitcheck: Literal["off","warn","error"] = "warn",
            udfs: UdfRegistry | None = None) -> CompileResult: ...
```

`QueryShape` describes, without deciding anything about sketches: tables, joins present, subqueries present, aggregates `(func, column, args)`, predicates as `(column, op, literals)` with AND/OR tree, group-by columns. arrowtap's classifier becomes a pure function over `QueryShape`.

`UnitCheckError` is raised only when `unitcheck="error"` and an `error` diagnostic exists; otherwise diagnostics are returned.

## Rules to carry over

Exactly the rule table and propagation rules in arrowtap `.claude/skills/vounit-checker/SKILL.md`, including the no-false-alarm list. Copy that skill into adqlduck's `.claude/skills/` with the paths updated; it is the spec.

## Dependencies added to adqlduck

`astropy` (units only). Keep it an optional extra `adqlduck[units]`; without it, `unit`/`ucd` pass through from schema and `check()` returns no diagnostics plus one info note.

## Acceptance

- [ ] arrowtap with `ARROWTAP_ADQL_BACKEND=adqlduck` passes its full test suite unchanged.
- [ ] Units conformance YAML passes in adqlduck CI.
- [ ] Geometry functions get units: `DISTANCE` → deg, `AREA` → deg**2; `CIRCLE` radius checked (U005).
- [ ] `compile()` overhead with `unitcheck="warn"` < 2 ms for catalogue queries.
- [ ] VOSI `tables` generator emits `unit`/`ucd` from the same `TapSchema` used by the checker.

## Order of work

1. `ColumnMeta` + propagation (no diagnostics). Smallest change, unblocks correct metadata in all services.
2. `check()` with the rule table.
3. `QueryShape`.
4. UDF registry.
