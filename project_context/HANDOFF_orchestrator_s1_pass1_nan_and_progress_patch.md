# HANDOFF — Claude → Orchestrator: S1 Pass 1 token-pair NaN fix + progress heartbeat

**Task:** Patch the S1 Pass-1 token-pair enumeration edge case that stopped the run with
`STOP_PRECISION_LOSS: float token/index not string-safe: nan`, and add a visible progress
heartbeat to `scripts/price_source_s1_coverage.py` for long local runs. Implementation +
tests only. No network, no data run.

## Part 1 — NaN / missing token-pair enumeration fix

### Root cause (verified, not assumed)

`Store.load_trades()`'s `traded_at`/`token_id`/`outcome_index` pass through
`.astype(str)` in `StoreLoader.load_trades_index`. I confirmed empirically (sandbox pandas
3.0.2) that **`.astype(str)` does NOT stringify `NaN`/`None` under pandas 3.x's native
string dtype** — it leaves a raw Python `float('nan')` in place. That raw NaN then reached
`canonical_int`, whose `isinstance(value, float)` branch raises
`DataExportPrecisionLoss(f"float token/index not string-safe: {value!r}")`, and
`repr(float('nan'))` is the unquoted text `nan` — reproducing the exact reported message
character-for-character. This is a real pandas-3.x behavior, not a hypothetical.

### Fix

`enumerate_token_pair` now skips and counts missing/malformed rows **before** any value
reaches `canonical_int`:

- `_is_missing_field(value)` — `True` for `None`, an empty/whitespace string, a case-insensitive
  `"nan"`/`"none"` string, or a real float NaN (`value != value`). **Not** true for any other
  non-null float (e.g. `5.2e76`) — those still reach `canonical_int` and still
  `STOP_PRECISION_LOSS`.
- A trade row with a missing `token_id` or `outcome_index` is skipped and counted in a new
  `malformed_rows` counter; it does not affect enumeration correctness.
- `enumerate_token_pair` now returns `(side_0_token, side_1_token, status, malformed_rows)` —
  a 4-tuple. `resolve_token_pairs` stores the count on a new
  `ConditionRecord.malformed_trade_rows` field.
- A condition whose only rows are malformed still resolves to `TOKEN_PAIR_UNRESOLVED`
  (never a stop) once all its rows are skipped and the remaining valid set can't form a pair.
- A condition with enough valid rows resolves normally despite unrelated malformed rows.
- **Unaffected:** `resolved_winning_token_id` is still never a token-pair source (guard-by-
  construction, test-checked); no `yes_price` / `1 - price` / `1 - yes_price` / `1 - p`
  synthesis anywhere; scientific-notation and float-mangled **non-null** ids still hard-stop.

### Diagnostic reporting

`malformed_trade_rows_total` (summed once, right after `resolve_token_pairs`, and reused
everywhere downstream so the heartbeat line, the JSON result, and the excluded-ledger reasons
never drift from each other) is now:
- printed in the `[S1] token pairs resolved: ...` heartbeat line,
- included in `price_source_s1_coverage.json` (`malformed_trade_rows_total`),
- included in `price_source_s1_coverage.md`,
- annotated per-condition in `price_source_s1_excluded.csv` when a `TOKEN_PAIR_UNRESOLVED`
  condition had malformed rows.

No artifact schema change beyond this diagnostic count — `BY_CONDITION_HEADER` is unchanged,
the `_FORBIDDEN_LEDGER_COLUMNS` assertion (no price columns) still passes.

## Part 2 — progress / heartbeat

Added `_progress(quiet, msg)` — a single-line `[S1] ...` write to **stderr**, `flush=True`,
no-op when `quiet`. Never writes to stdout (keeps the final JSON summary machine-readable) and
never prints a price series or a per-condition token list — only counts, phase names, and
elapsed time.

Phase markers wired into `run_pass1_coverage`, in order:
```
[S1] loading P0/contract/resolution/trades...
[S1] universe built: X conditions
[S1] token pairs resolved: A/B, unresolved C, malformed trade rows D
[S1] sample selected: N conditions
[S1] fetching coverage: 0/N conditions, elapsed 00:00:00        (before the first fetch)
[S1] fetching coverage: 25/N conditions, elapsed 00:01:12        (every --progress-every)
...
[S1] fetching coverage: N/N conditions, elapsed HH:MM:SS         (final tally)
[S1] writing artifacts...
[S1] done: verdict=...
```
A `[S1] done: verdict=...` line also fires on every **early stop** (P0 not clear, stale
contract, unreliable enumeration, sample-bound violation, endpoint-shape deviation,
precision loss) via `_stop_result(verdict, quiet)`, so a run that aborts still gives visible
closure instead of silence.

**CLI:** progress is **ON by default** (`RunConfig.quiet = False`). New flags:
`--quiet` (suppress) and `--progress-every N` (default 25). `main()`'s existing
`STOP_NOT_AUTHORIZED` message and the final JSON summary are unchanged (still stdout).

**Manual verification** (mocked run, stdout redirected to confirm markers are stderr-only):
```
[S1] loading P0/contract/resolution/trades...
[S1] universe built: 60 conditions
[S1] token pairs resolved: 60/60, unresolved 0, malformed trade rows 0
[S1] sample selected: 55 conditions
[S1] fetching coverage: 0/55 conditions, elapsed 00:00:00
[S1] fetching coverage: 25/55 conditions, elapsed 00:00:00
[S1] fetching coverage: 50/55 conditions, elapsed 00:00:00
[S1] fetching coverage: 55/55 conditions, elapsed 00:00:00
[S1] writing artifacts...
[S1] done: verdict=S1_SOURCE_PARTIAL
```
stdout was empty in that run — confirmed markers are stderr-only.

## Tests

`tests/test_price_source_s1_coverage.py`: **109 passed** (92 prior + 17 new).

New tests (NaN/malformed, 10): NaN token_id ignored + counted; NaN outcome_index ignored +
counted; resolves despite unrelated malformed rows (NaN float, `None`, empty, whitespace,
text "nan"/"none" all mixed in); only-malformed condition → `TOKEN_PAIR_UNRESOLVED` (asserted
**no exception**, not `STOP_PRECISION_LOSS`); a real non-null float-mangled id amid otherwise-
valid rows still `STOP_PRECISION_LOSS`; same for a scientific-notation **string** amid missing
rows; `resolved_winning_token_id` still never a source (guard re-asserted); `resolve_token_pairs`
aggregates the count onto `ConditionRecord`; an end-to-end `StoreLoader` reproduction using a
**real `np.nan`** in a mocked pandas dataframe (the literal originally-reported scenario) —
confirmed via direct sandbox repro to no longer raise; orchestration-level test confirming
`malformed_trade_rows_total` reaches the JSON result.

New tests (progress, 7): markers appear on stderr (not stdout) when not quiet, matching every
required phase name; suppressed entirely when `quiet=True`; heartbeat fires before the first
fetch and every `progress_every` conditions plus a final tally; a stop path still emits a done
marker; CLI default is progress-on (`quiet is False`, `progress_every == 25`); `--quiet` /
`--progress-every` flags parse correctly; `_fmt_elapsed` formats HH:MM:SS.

Existing orchestration tests' shared `_cfg()` helper now defaults to `quiet=True` so the suite
stays non-noisy; only the dedicated progress tests opt into `quiet=False`.

Guardrail re-audit on the final file (executable code only, comments/strings excluded): no
`yes_price` / `1 - yes_price` / `1 - price` / `1 - p`, no `save_prices`, no
`backfill_prices_from_clob`, `named_binary_probe_blocked` never assigned, `run_pass2` never
called from `run_pass1_coverage`.

## Explicit non-performance statement

No network call, no Polymarket endpoint call, no Pass 2, no S2, no P1/P2/P3, no probe
execution, no scoring, no backfill, no wallet/OrdersMatched/log_index/PnL work, no audit-gate
modification, `named_binary_probe_blocked` not flipped.

## Files delivered

1. `scripts/price_source_s1_coverage.py`
2. `tests/test_price_source_s1_coverage.py`
3. this handoff.

## Future coding convention (not applied retroactively)

Any future user-run script with a long loop, a network call, or a large local data scan
should include a visible progress/heartbeat by default (stderr, `flush=True`, compact phase
markers, counts/elapsed only — never bulk data) plus a `--quiet` option to suppress it. This
patch does not retrofit that pattern onto any other existing script; it applies only to
`price_source_s1_coverage.py` as requested.

## Next concrete action

Copy the two files into `C:\b1\pm_research`, re-run the test suite locally, then re-attempt
the S1 Pass-1 run. The network run itself remains unauthorized here — separate explicit
authorization is required before any real fetch.
