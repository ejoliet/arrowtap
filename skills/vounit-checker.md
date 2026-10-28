---
name: vounit-checker
description: Rules, diagnostic codes, and test catalogue for the ADQL unit/UCD checker (dimensional analysis at compile time, unit propagation, UCD autocomplete). Use when editing src/arrowtap/units/, tests/units/, or adding a rule.
---

# VOUnit / UCD checker

## Model

- Walk the ADQL AST bottom-up. Each expression node gets `(unit | None, ucd | None)`.
- Column units come from `tap_schema.yaml`, parsed with `astropy.units.Unit(s, format="vounit")`. Empty or unparseable unit → `None` (unknown). Unknown is silent; count it for the coverage report.
- Numeric literals are **dimensionless-adaptive**: in `+ - < > = BETWEEN` they adopt the other operand's unit. In `* /` they are dimensionless.
- The checker reports. It never rewrites the query.

## Rules

| Code | Level | Rule | Suggestion in message |
|---|---|---|---|
| U001 | error | `+ - comparisons BETWEEN` on units with different physical type | — |
| U002 | error | Same physical type, different scale (deg vs arcsec, mas/yr vs arcsec/yr) | conversion factor, e.g. `* 3600` |
| U003 | error | `SIN COS TAN` argument has angle unit other than rad | wrap in `RADIANS()` |
| U004 | error | `EXP`, `POWER` exponent not dimensionless | — |
| U005 | error | Geometry radius/size (`CIRCLE`, `BOX`) is a column with non-deg angle unit | conversion factor |
| W010 | warn | `mag + mag` (addition of magnitudes) | colours are differences; sums are usually a mistake |
| W011 | warn | `AVG` or `SUM` over `mag` | flux-average differs from magnitude-average |
| W020 | warn | UCD rule: arithmetic between `pos.eq.ra` and `pos.eq.dec` | — |
| W021 | warn | UCD rule: comparing `pos.eq.ra` to `pos.eq.dec` | — |

Propagation:

- `* /` multiply units. `RADIANS(deg)` → rad, `DEGREES(rad)` → deg. `ASIN ACOS ATAN ATAN2` → rad.
- `LOG LOG10 SQRT` of a dimensional value: **no diagnostic**; `LOG*` → dimensionless, `SQRT` → unit**0.5. (Astronomy uses `LOG10(1000/parallax)` routinely; flagging it breaks invariant 5.)
- `mag - mag` → mag, UCD `phot.color` if both operands are `phot.mag`.
- Aggregates keep the unit, except `COUNT` → dimensionless.
- Output unit string: `unit.to_string("vounit")`.

## Catalogue (tests/units/catalogue.yaml)

Each entry: `id`, `adql`, `expect` (list of codes, empty = must be clean), optional `out_unit` per column.

| id | ADQL | expect |
|---|---|---|
| ra_plus_pm | `SELECT ra + pmra FROM gaia` | U001 |
| dist_vs_parallax | `... WHERE DISTANCE(POINT('ICRS',ra,dec), POINT('ICRS',10,20)) < parallax` | U001 |
| sin_deg | `SELECT SIN(dec) FROM gaia` | U003 |
| pm_vs_rv | `... WHERE pmra > radial_velocity` | U001 |
| deg_vs_arcsec | `... WHERE ABS(dec - b) < pos_err_arcsec` (schema col in arcsec) | U002 |
| exp_dim | `SELECT EXP(parallax) FROM gaia` | U004 |
| mag_sum | `SELECT phot_g_mean_mag + bp_rp FROM gaia` | W010 |
| avg_mag | `SELECT AVG(phot_g_mean_mag) FROM gaia` | W011 |
| ra_plus_dec | `SELECT ra + dec FROM gaia` | W020 |
| circle_arcsec_col | `... CIRCLE('ICRS', ra, dec, pos_err_arcsec)` | U005 |
| abs_mag | `SELECT phot_g_mean_mag + 5*LOG10(parallax/1000) + 5 FROM gaia` (parallax mas) | clean; this sums mag with dimensionless terms via adaptive literals and a LOG result, so it must not trigger W010 |
| color | `SELECT phot_bp_mean_mag - phot_rp_mean_mag FROM gaia` | clean, out_unit mag |
| pm_cosdec | `SELECT pmra * COS(RADIANS(dec)) FROM gaia` | clean, out_unit mas/yr |
| inv_parallax | `SELECT 1000/parallax AS d FROM gaia` | clean, out_unit mas**-1 |
| literal_cmp | `... WHERE phot_g_mean_mag < 15 AND dec BETWEEN -5 AND 5` | clean |

W010 fires only when **both** operands carry `mag` from columns or mag-typed expressions, not when one side is dimensionless.

Add a clean case for every new rule (AGENTS invariant 5).

## UCD autocomplete

`/arrowtap/complete?ucd=X&table=T`: return columns whose UCD contains `X` as a word (`;`-separated atoms, primary first). Rank primary-atom matches above secondary. Include `meta.main` ties first.

## Coverage report

`arrowtap check --coverage` prints per table: columns with parseable unit / total, columns with UCD / total. Run it on any real TAP_SCHEMA before claiming results.
