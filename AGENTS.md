<!-- AGENTS.md — agent guidance for arrowtap (CLAUDE.md is a symlink to this file)
     Last reviewed: 2026-10-11  Owner: ejoliet
     AIDEV-NOTE: keep ≤ 200 lines; detailed rules live in .claude/skills/ -->

# arrowtap

Mini-TAP service plus benchmark proving three conventions TAP 1.1 already allows: Arrow IPC responses, `PRECISION=approx`, and unit-checked ADQL. Spec: `README.md`.

## Project Invariants (DO NOT VIOLATE)

1. **Every format returns the same values for the same query.** Equivalence test gates every change in `formats/`.
   Blast: the benchmark compares different results and the whole pitch is void.
2. **Metadata round-trip is lossless for in-scope types.** VOTable → Arrow → VOTable keeps unit, ucd, description, utype, xtype, datatype, arraysize, ID.
   Blast: DAL reviewers dismiss the convention on first test.
3. **Approximate answers are always labelled.** No response with sketch-derived values may omit `tap:precision=approx` and bounds.
   Blast: a scientist cites an estimate as exact.
4. **The unit checker never converts silently.** It reports; it does not rewrite queries.
   Blast: results change meaning without the user seeing why.
5. **No false alarms on standard formulas.** Absolute magnitude, colours, `COS(RADIANS(dec))*pmra` must pass clean. Add a clean case to the catalogue for every new rule.
   Blast: users switch `UNITCHECK=off` and the feature is dead.
6. **Benchmarks follow `.claude/skills/bench-protocol`.** Baseline is VOTable BINARY2 + gzip, not TABLEDATA.
   Blast: results are dismissed as a strawman.
7. **Pre-stream errors are standard VOTable error documents,** regardless of `RESPONSEFORMAT`.
   Blast: existing TAP clients (pyvo, TOPCAT) mis-handle errors.
8. **Tests never touch the network.** Use `data synth` fixtures.
   Blast: flaky CI, slow loop.
9. **`arrowtap.formats` imports only pyarrow, numpy, astropy.** No FastAPI, DuckDB, or server code. A clean-venv import test enforces it.
   Blast: tapdrop's dependency on `arrowtap[formats]` breaks (see `docs/handoff/tapdrop.md`).

> If an instruction below conflicts with an invariant, the invariant wins.

## Recently Burned

<!-- AIDEV-BURN: last 3–5 mistakes, dated. Prune quarterly. -->

- None yet. Add entries as `YYYY-MM-DD: what went wrong → corrective rule`.

## Workflow Expectations

- Finish Phase 0 (Gate) and report outcomes before writing Phase 1 code.
- Do not run browsers, push, or open PRs. Write the page or script, then print the exact command for Emmanuel.
- Do not install heavy optional deps to verify something Emmanuel can check; give him the command.
- Package for `uvx --from git+https://github.com/ejoliet/arrowtap arrowtap`.
- Pin CDN script versions exactly (no ranges, no `@latest`), each release ≥ 14 days old; record in `docs/decisions.md`.
- Update `docs/decisions.md` for every decision that answers an Open Question.
- When a change affects the convention (keys, codes, MIME strings), update `README.md`, `docs/convention.md`, and the matching skill in the same change.
- Run `make test lint` before declaring a phase done.

## Conventions

- `AIDEV-NOTE`, `AIDEV-TODO`, `AIDEV-QUESTION` comments are anchors; preserve them.
- Metadata keys: `tap:*` for protocol-level, `votable:*` for VOTable FIELD attributes, `arrowtap:*` for service-specific (bounds). Never invent unprefixed keys.
- Diagnostic codes: `U*` blocking unit errors, `W*` warnings, `A*` approx notices. Codes are stable once released.
- One batch iterator feeds all writers. Do not add a format-specific query path.

## Stack

- Lang: Python 3.11 · Pkg: uv · Test: pytest · Lint: ruff, mypy
- Server: FastAPI + DuckDB · Arrow: pyarrow · Units: astropy.units (vounit)
- Browser: Arrow JS + DuckDB-WASM, static page, no build step

## Out of Scope for the Agent

- Async TAP/UWS, Iceberg/puffin, DuckDB extensions, geometry in the built-in compiler.
- Editing `adqlduck` or `tapdrop` repos. Changes for them go in `docs/handoff/*.md`.

## Skills (read before working in the area)

| Area | Skill |
|---|---|
| `formats/`, metadata, termination | `.claude/skills/votable-arrow-mapping` |
| `bench/`, `report.py`, results | `.claude/skills/bench-protocol` |
| `units/` | `.claude/skills/vounit-checker` |
| `approx/`, `data/sketches.py` | `.claude/skills/approx-sketches` |
