# HANDOFF — Claude → Copilot: S1 Pass 1 invalid decision-window handling

**Task:** Patch S1 Pass 1 so invalid/missing decision windows are excluded and reported
instead of being coerced into 1-second queries and false `DECISION_PRICE_NEITHER` negatives.
Implementation + tests only. No network, no data run.

## Was the all-1-second result an invalid-window issue or a code bug? → BOTH, and I correct a prior wrong call.

**The all-1-second / all-NEITHER / `S1_SOURCE_NOT_VIABLE` result came from INVALID decision
windows in the sample — specifically, `resolved_at` is missing (blank) for every sampled
condition — combined with a code defect that silently coerced those into 1-second fetches and
counted them as real coverage negatives.**

I verified this against the uploaded by-condition CSV (all 300 rows): `resolved_at_ts` is blank
for 100% of sampled conditions, `request_window_seconds == 1` for all 300, and every row is
`DECISION_PRICE_NEITHER`. The observed GET (`startTs=1778387595`, `endTs=1778387596`)
corresponds to `decision_lower = first_trade_ts + 3600` with `resolved_at` absent, so
`_decision_request_window` hit its `resolved_at is None -> start+1` clamp.

**Correction to my previous handoff (per "never silently reverse a prior decision"):** in the
prior diagnostics patch I concluded the earlier 1-second artifact was merely *stale* (pre-fix).
That was wrong. The new diagnostics I added (`request_window_summary`, per-condition window
columns) are exactly what exposed the truth: the current code, on the current data, produces
1-second windows because `resolved_at` is missing for the sample. The request-window *fix* was
real, but it did not address the *invalid/missing-window* case — which this patch now does.

## Root cause (two layers)

1. **Data:** `resolved_at` is blank for the sampled conditions reaching the fetch loop. (Why the
   resolution-source `resolved_at` is empty is an upstream data question for the loader/parquet;
   it is out of scope for this coverage-only patch, but flagged below.)
2. **Code defect (fixed here):** the fetch loop treated a missing/invalid window as a normal
   condition — queried a degenerate 1-second window and classified the empty result as
   `DECISION_PRICE_NEITHER`, which then rolled into a false `S1_SOURCE_NOT_VIABLE`. A coverage
   negative must mean "we queried a real decision window and found nothing", never "there was no
   decision window to query."

## Fix

- New `valid_decision_window(first_trade_ts, resolved_at_ts)`:
  `valid <=> both anchors present AND resolved_at > first_trade_ts + warmup`.
- Fetch loop now **partitions the sample first**:
  - **Invalid-window condition** → NOT fetched, NOT classified `DECISION_PRICE_NEITHER`.
    Recorded in the by-condition ledger as `NO_VALID_DECISION_WINDOW_AFTER_WARMUP` with
    `first_trade_ts`, `decision_lower_ts`, `resolved_at_ts`, and window seconds; added to the
    excluded ledger with the same diagnostics. Level-A columns show `NOT_QUERIED`.
  - **Valid-window condition** → fetched and classified exactly as before (validity guarantees
    the request window can never collapse to the 1-second clamp).
- **All-invalid sample** → typed verdict `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE`
  (never `S1_SOURCE_NOT_VIABLE`). Endpoint coverage was not measured because the sample has no
  valid warmup/resolution window.
- **Mixed sample** → coverage measured over valid-window conditions only; invalid ones excluded
  and reported; per-subclass valid/invalid counts included.
- New `decision_window_validity` block in the JSON/MD: `sampled`, `valid_window`,
  `invalid_window`, and per-subclass breakdowns. Reconciles: by-condition rows == full sample
  (valid measured + invalid reported); `request_window_summary.n` == valid fetched count.
- Progress heartbeat preserved; final tally now reads
  `N/M valid-window conditions fetched (K skipped, invalid window)`.

Leakage discipline unchanged (strict `< resolved_at` in Level B). No price series persisted.
No `yes_price`/`1 - p`/side synthesis. Pass 2 hard-stopped. No gate change;
`named_binary_probe_blocked` never assigned. By-condition ledger schema unchanged (18 columns,
no price columns).

## End-to-end reproduction (mocked, no network)

Reproducing the uploaded scenario — 300 conditions across all three subclasses, all with blank
`resolved_at`:
- verdict: `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE` (was `S1_SOURCE_NOT_VIABLE`);
- GET calls made: **0** (nothing fetched);
- `DECISION_PRICE_NEITHER`: **0** (was 300);
- `decision_window_validity`: 0 valid / 300 invalid;
- excluded ledger: 300 × `NO_VALID_DECISION_WINDOW_AFTER_WARMUP` with window diagnostics.

## Tests

`tests/test_price_source_s1_coverage.py`: **132 passed** (125 prior + 7 new; 2 prior tests
updated to the corrected semantics — the obsolete "1-second window is fetched" premise and the
final heartbeat wording).

New tests:
- `valid_decision_window` truth table (strict `>`, missing anchors);
- invalid-window condition is **not** fetched (zero client calls);
- invalid-window is **excluded/reported**, never `DECISION_PRICE_NEITHER`, with window
  diagnostics in both the by-condition and excluded ledgers;
- **all-invalid sample → `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE`**, not NOT_VIABLE;
- mixed valid/invalid sample fetches only valid conditions (exact GET count) and reconciles
  counts (by-condition rows == full sample; summary n == valid count; excluded == invalid);
- mixed sample measures coverage on valid conditions only (per-subclass measured == valid
  count) and reports per-subclass valid-window counts;
- JSON reports the `decision_window_validity` counts.

## Explicit non-performance statement

No network call, no Polymarket endpoint call, no Pass 2, no S2, no P1/P2/P3, no probe, no
scoring, no backfill, no wallet/OrdersMatched/log_index/PnL, no gate modification,
`named_binary_probe_blocked` not flipped.

## Files delivered

1. `scripts/price_source_s1_coverage.py`
2. `tests/test_price_source_s1_coverage.py`
3. this handoff.

## Next concrete action + upstream flag

Copy the two files into `C:\b1\pm_research`, re-run the test suite, then **re-run S1 Pass 1 and
discard the prior NOT_VIABLE** (it was a false negative from invalid windows). The fresh run will
almost certainly return `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE` **until the upstream
`resolved_at` gap is understood** — which is the real blocker:

> **Upstream data flag (needs verification, not fixed here):** the resolution-source rows reaching
> the loader have blank `resolved_at` for the sampled conditions, even though the universe is
> built from `status == RESOLVED_SINGLE_WINNER` rows that should carry it. Before S1 can measure
> anything, confirm whether `named_binary_resolution_source_rows.parquet` actually populates
> `resolved_at` (correct column name? datetime vs string? populated for these conditions?). This
> is a `NEEDS VERIFICATION` on the data contract, separate from this coverage-only code patch.

The network run itself remains unauthorized; a real fetch needs separate explicit authorization.
