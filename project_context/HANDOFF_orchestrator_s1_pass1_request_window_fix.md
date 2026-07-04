# HANDOFF — Claude → Orchestrator: S1 Pass 1 request-window fix

**Task:** Review and patch the S1 Pass-1 request-window logic. The user-run completed but the
observed GET used a ~1-second window (`startTs=1778383995`, `endTs=1778383996`), producing
`SERIES_EMPTY: 600`, `DECISION_PRICE_NEITHER: 300`, and a **false** `S1_SOURCE_NOT_VIABLE`.
The reviewer correctly refused that verdict. Implementation + tests only. No network, no data run.

## Analysis (verified against the spec)

The verdict is **not trustworthy** because the request window was too narrow to observe the
decision window at all. Per `SPEC_price_source_s1_coverage.md`:

- **Level A (§4.1)** — token-endpoint reachability / whether a series exists.
- **Level B (§4.2, §7.2, binding)** — a point at/after `first_trade_ts + 3600` and **strictly
  before** `resolved_at` (leakage-safe).

A 1-second request window `[first_trade_ts, first_trade_ts+1]` cannot contain any point at
`first_trade_ts + 3600`, so every series came back empty and every condition classified
`DECISION_PRICE_NEITHER` — a measurement artifact, not a real coverage negative.

Note on code archaeology: the working-tree fetch loop had `start_ts = first_trade_ts`,
`end_ts = resolved_at`, with a degenerate `else: start_ts + 1` only when `resolved_at` was
missing. The 1-second artifact indicates the run was produced by a build where the narrow/
degenerate path dominated. Either way the request-window construction was not robustly spanning
the decision window, so it is now centralized and fixed.

## Fix

New helper `_decision_request_window(first_trade_ts, resolved_at_ts, warmup_seconds)`:

```
lower = first_trade_ts + warmup_seconds
upper = resolved_at
request startTs = floor(lower)
request endTs   = ceil(upper)      # inclusive of the upper bound so a point just below
                                   # resolved_at is actually returned
```

- The endpoint request now spans the **full decision window**. The leakage-safe cut
  (**strictly `< resolved_at`**) is unchanged and still applied in `classify_decision_window`
  / `has_decision_window_point` — never by narrowing the request. **No price at/after
  `resolved_at` is counted.**
- Degenerate inputs are handled explicitly: no trade anchor → request from 0 (Level B yields
  `NO_TRADE_ANCHOR` regardless); no `resolved_at` → `end = start + 1` (cannot bound the window,
  so it stays a non-usable diagnostic rather than a false hit); `lower >= upper` → clamp
  `end = start + 1` so the request is always well-formed (never a backwards range).
- The fetch loop now calls the helper instead of building the window inline.

### Level A labeling (reviewer's sub-question)

Because the code queries only the decision window and performs **no separate broad
reachability probe**, `SERIES_EMPTY` means "no points in the decision window", not "this token
has no history anywhere". Rather than overclaim, both `price_source_s1_endpoint_shape.md` and
`price_source_s1_coverage.md` now carry an explicit **Level A scope note**: the Level A status
reflects the endpoint response for the queried decision window
`[floor(first_trade_ts+warmup), ceil(resolved_at)]`, and a future S2 build may add a broader
reachability probe if that distinction matters. This is a labeling clarification only — no false
widening of what the code actually measures.

## Preserved (unchanged)

Progress heartbeat, NaN/malformed token-pair handling, no `yes_price`/`1 - p` side synthesis,
batch disabled unless `--allow-batch`, Pass 2 hard-stopped, sample bounded, no price series
persisted, by-condition ledger price-free, `named_binary_probe_blocked` never assigned. Verified
by executable-code audit and the full suite.

## Tests

`tests/test_price_source_s1_coverage.py`: **117 passed** (109 prior + 8 new).

New tests:
- `_decision_request_window` floor/ceil correctness; degenerate no-`resolved_at`; backwards
  clamp is well-formed.
- **GET receives the decision window, not 1 second** — asserts every recorded GET has
  `startTs == floor(first_trade_ts + warmup)` and `endTs == ceil(resolved_at)`, and
  `end - start > 1`.
- **A point after decision_ts but before resolved_at counts** — in-window point → per-subclass
  `DECISION_PRICE_BOTH_SIDES` clears (verdict `S1_SOURCE_PARTIAL` on the UP_DOWN-only fixture,
  since pooled `VIABLE` requires all three subclasses to clear, spec §7.2).
- **A point at/after resolved_at does not count** — leakage points only → `NEITHER` everywhere →
  `S1_SOURCE_NOT_VIABLE`.
- **`endpoint_shape.md` records the wider request params** — asserts the observed `startTs`/
  `endTs` in the shape ledger are the wide decision-window bounds, and the Level A scope note is
  present.

## Explicit non-performance statement

No network call, no Polymarket endpoint call, no Pass 2, no S2, no P1/P2/P3, no probe, no
scoring, no backfill, no wallet/OrdersMatched/log_index/PnL, no audit-gate modification,
`named_binary_probe_blocked` not flipped.

## Files delivered

1. `scripts/price_source_s1_coverage.py`
2. `tests/test_price_source_s1_coverage.py`
3. this handoff.

## Next concrete action

Copy the two files into `C:\b1\pm_research`, re-run the test suite, then **re-run S1 Pass 1**.
The prior `S1_SOURCE_NOT_VIABLE` should be **discarded** — it was produced under the narrow
window and is not a valid negative. Re-check the new `endpoint_shape.md` to confirm the observed
GET now shows the wide `startTs`/`endTs` before trusting the fresh verdict. The network run
itself remains unauthorized here; a real fetch still needs separate explicit authorization.
