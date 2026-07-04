# HANDOFF — Claude → Orchestrator: S1 parse_ts millisecond-UTC format fix

**Task:** Patch `parse_ts` to accept the real `resolved_at` string format. Timestamp-parsing
only. No network, no S1 rerun, no gate change.

## Root cause (confirmed by the user's parquet inspection)

The user inspected `named_binary_resolution_source_rows.parquet`:
`resolved_at` dtype `str`, null rate `0.0`, values like `"2025-03-06 07:25:37.000 UTC"`.

So the column is **populated** — the prior all-invalid-window sample was **not** a data gap.
The blocker was `parse_ts`: after stripping the ` UTC` suffix, the residue
`"2025-03-06 07:25:37.000"` carries **millisecond fractional seconds**, which matched none of
the whole-second/ISO/date-only formats → `ValueError` → `None` → blank → invalid window for
every row.

(Note: the plain `"…37 UTC"` whole-second form already parsed; the real data's `.000`
fraction is what was rejected. This refines my earlier diagnosis, which over-attributed the
failure to a tz-aware datetime coercion — the actual column is a fractional-second string.)

## Fix (parse_ts only)

Added two fractional-second formats to the strptime list, alongside the existing ones:
```
"%Y-%m-%d %H:%M:%S"
"%Y-%m-%d %H:%M:%S.%f"     # NEW - "2025-03-06 07:25:37.000 UTC" (UTC already stripped)
"%Y-%m-%dT%H:%M:%S"
"%Y-%m-%dT%H:%M:%S.%f"     # NEW - ISO fractional
"%Y-%m-%d"
```
`%f` matches 1–6 fractional digits, covering `.000` millisecond precision. ` UTC` is stripped
before parsing and the result is stamped `tzinfo=timezone.utc` (UTC treated as tz-aware UTC).

Preserved unchanged: ISO strings, pandas `Timestamp` / Python `datetime` (the earlier
datetime-object branch), epoch numerics, date-only. Token-id / large-integer identity handling
is untouched (token ids never pass through `parse_ts`; `canonical_int` still fails loud on
scientific notation). **No** remapping of `evt_block_time` / `block_time` / `endDate` (verified
absent from the code). Non-timestamp identity validation unchanged. A non-UTC tz suffix
(e.g. `" PST"`) is still rejected, not silently accepted.

## Verification

Real head values now parse:
- `"2025-03-06 07:25:37.000 UTC"` → `1741245937.0`
- `"2025-03-06 04:49:48.000 UTC"` → `1741236588.0`
- fractional correctness: `".500"` adds exactly `0.5` s.

## Tests

`tests/test_price_source_s1_coverage.py`: **143 passed** (137 prior + 6 new).

New:
- accepts `"2025-03-06 07:25:37.000 UTC"` (millisecond);
- accepts `"2025-03-06 07:25:37 UTC"` (whole-second);
- fractional seconds honored (`.500` → +0.5 s);
- prior formats preserved (ISO, ISO-fractional, date-only, epoch numeric/string, py datetime);
- still fails loud on garbage and on a non-UTC tz suffix;
- **end-to-end**: a mocked S1 condition with `first_trade_ts` earlier and `resolved_at` in the
  real millisecond-UTC form now has a VALID decision window (was invalid before) and the verdict
  is no longer `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE`.

## Guardrails

No `yes_price` / `1 - yes_price` / `1 - price` / `1 - p`; no `save_*`; no
`backfill_prices_from_clob`; no `evt_block_time`/`block_time`/`endDate` remap;
`named_binary_probe_blocked` never assigned; leakage-safe strict `< resolved_at` unchanged;
Pass 2 hard-stopped; no price series persisted. No network, no S1 rerun, no Pass 2/S2/P1/P2/P3,
no probe, no scoring, no backfill, no wallet/PnL, no gate change.

## Files delivered

1. `scripts/price_source_s1_coverage.py`
2. `tests/test_price_source_s1_coverage.py`
3. this handoff.

## Next concrete action

Copy the two files into `C:\b1\pm_research`, re-run the test suite, then re-run S1 Pass 1
(separately authorized). With `resolved_at` now parsing, the sample should show valid decision
windows and the run should fetch coverage instead of returning the inconclusive verdict. This
patch also supersedes the loader-coercion concern from the prior handoff for the string-typed
column that actually ships; the `STOP_RESOLUTION_SCHEMA` guard added there remains as a
belt-and-suspenders check for a missing column. The network run itself remains unauthorized.
