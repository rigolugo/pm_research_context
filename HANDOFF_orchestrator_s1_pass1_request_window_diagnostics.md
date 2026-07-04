# HANDOFF — Claude → Copilot: S1 Pass 1 request-window DIAGNOSTICS

**Task:** Add request-window diagnostics to the S1 artifacts so the orchestrator can verify
whether the queried windows are correct; change coverage semantics only if the request-window
bug still exists. Implementation + tests only. No network, no data run.

## Was this diagnostics-only, or was a bug found? → DIAGNOSTICS-ONLY.

**The request-window bug is already fixed in the current code; the uploaded artifacts are
stale (produced by a pre-fix build).** Verified directly:

- `_decision_request_window(first_trade_ts, resolved_at_ts)` is present and the fetch loop
  calls it: `start = floor(first_trade_ts + warmup)`, `end = ceil(resolved_at)`. It cannot
  return a 1-second window when both anchors are real and ordered (checked by unit test and by
  reading the function).
- The uploaded `coverage.json`/`endpoint_shape.md` show `startTs=1778387595`,
  `endTs=1778387596` (a 1-second window) with `SERIES_EMPTY: 600` and
  `malformed_trade_rows_total: 2413`. That 1-second window is the **old**
  `[first_trade_ts, first_trade_ts+1]`-style behavior — i.e. these artifacts predate the fix
  the user has since copied in. A fresh run on the current code produces wide windows
  (e.g. ~428,400 s on the test fixture), `count_le_1 = 0`, `count_gt_3600 = n`.

So the reviewer's refusal to accept the run was **correct**, but the cause is a stale artifact,
not a live bug. The real gap was **observability**: the artifacts couldn't *prove* the window
was correct because only the first GET's window was recorded and the by-condition CSV had no
window columns. This patch closes that gap. No coverage semantics were changed.

## Diagnostics added

### price_source_s1_coverage_by_condition.csv (new columns, timestamps/counts only)
`first_trade_ts`, `decision_lower_ts` (= first_trade_ts + warmup_seconds), `resolved_at_ts`,
`request_start_ts`, `request_end_ts`, `request_window_seconds` (= end - start),
`observed_point_count_side_0`, `observed_point_count_side_1`.
Still no price values — the `_FORBIDDEN_LEDGER_COLUMNS` guard passes; `observed_point_count_*`
is a count of returned points, not a price.

### price_source_s1_coverage.json + .md — `request_window_summary`
`n`, `min_request_window_seconds`, `median_request_window_seconds`, `max_request_window_seconds`,
`count_request_window_seconds_le_1`, `count_request_window_seconds_le_60`,
`count_request_window_seconds_gt_3600`, `sample_count_by_subclass`. Computed once per run via
`request_window_summary(...)`. `count_le_1` is the explicit tripwire for the 1-second bug;
`count_gt_3600` counts windows wide enough to actually contain a decision-window point.

### price_source_s1_endpoint_shape.md
- The observed single-token GET is now explicitly labeled **"ONE SAMPLE ONLY"**.
- A **"Request-window summary (whole sample)"** section is included so a single 1-second
  example can never be read as the full sample. Points readers to the JSON summary and the
  per-condition CSV columns.

## Verification (mocked, no network)

A mocked authorized run over a fixture with `first_trade_ts = 2025-03-01`,
`resolved_at = 2025-03-06`:
- per-condition `request_start_ts = 1740790800` (= floor(ft+3600)),
  `request_end_ts = 1741219200` (= ceil(resolved_at)), `request_window_seconds = 428400`;
- the set of `(request_start_ts, request_end_ts)` in the CSV **equals** the set of
  `(startTs, endTs)` actually passed to the client's GET calls;
- `request_window_summary`: `count_le_1 = 0`, `count_gt_3600 = 6`, `min = 428400`.

## Tests

`tests/test_price_source_s1_coverage.py`: **125 passed** (117 prior + 8 new).

New:
- by-condition CSV includes all request-window columns (and still no price columns);
- CSV window values are internally consistent (`decision_lower = ft+warmup`,
  `request_start = floor(lower)`, `request_end = ceil(resolved_at)`, `window = end-start > 1`);
- CSV windows match the actual client GET calls;
- JSON contains `request_window_summary` with all required keys and correct wide-window counts;
- MD contains the summary;
- endpoint_shape.md marks the GET as one sample and includes the whole-sample summary;
- **a single 1-second first GET does not imply all windows are 1 second** — one deliberately
  degenerate condition (blanked `resolved_at`) yields exactly one ≤1s window while the summary
  and per-condition ledger still show the rest wide (a mix, not all-one);
- empty-sample summary is well-formed (no crash).

## Guardrails (re-audited on the final file)

No `yes_price` / `1 - yes_price` / `1 - price` / `1 - p`; no `save_*`; no
`backfill_prices_from_clob`; `named_binary_probe_blocked` never assigned; leakage-safe strict
`< resolved_at` unchanged in Level B; Pass 2 hard-stopped; no price series persisted; by-condition
ledger carries no price columns.

## Explicit non-performance statement

No network call, no Polymarket endpoint call, no Pass 2, no S2, no P1/P2/P3 resume, no probe,
no scoring, no backfill, no wallet/OrdersMatched/log_index/PnL, no gate modification.

## Files delivered

1. `scripts/price_source_s1_coverage.py`
2. `tests/test_price_source_s1_coverage.py`
3. this handoff.

## Next concrete action

Copy the two files into `C:\b1\pm_research`, re-run the test suite, then **re-run S1 Pass 1 on
the current code and discard the uploaded stale artifacts.** On the fresh run, verify in
`price_source_s1_coverage.json → request_window_summary` that `count_request_window_seconds_le_1`
is ~0 and `median_request_window_seconds` is large (days, not seconds), and spot-check a few
rows of the by-condition CSV. Only then is the coverage verdict trustworthy. Separately note:
the fresh run's `malformed_trade_rows_total` (2413 in the stale artifact) is worth a glance —
it is diagnostic only and does not block, but a high value is worth understanding before S2.
The network run itself remains unauthorized; a real fetch needs separate explicit authorization.
