# HANDOFF — Claude → Orchestrator: S1-ALT Pass 1 (Option A local trade-print) coverage RESULT (ACCEPTED)

**Status:** S1-ALT Pass 1 Option A (local trade-print) coverage test **completed (user-run) and
accepted as a sampled coverage finding.** Documentation-only task: records the result in
PROJECT_STATE, ARTIFACT_INDEX, and DECISION_LOG, and captures the finding here. No code, no
rerun, no network, no gate change, `named_binary_probe_blocked` not flipped.

## Result — `S1ALT_SOURCE_NOT_VIABLE`

Pass 1 **sampled** coverage (not Pass 2 full-universe), reusing the **exact accepted S1 Pass-1
300-condition sample**:

- **Sample reconstructed:** 300 / 300 conditions (union of the accepted S1 by-condition +
  excluded ledgers).
- **Measured-eligible:** 248. **S1-invalid-window, pre-excluded + reported:** 52 (the same
  conditions the accepted S1 run classified `NO_VALID_DECISION_WINDOW_AFTER_WARMUP` — carried
  forward and never re-measured against local trades, so they cannot become coverage negatives).
- **measured_by_subclass:** UP_DOWN 50 / OVER_UNDER 98 / NAMED_OTHER 100 — **identical** to the
  accepted S1 Pass-1 breakdown, confirming the same-sample-reuse requirement held exactly.
- **Level-B class counts:** BOTH_SIDES 124 / ONE_SIDE 65 / NEITHER 59 (sums to 248 measured).
- **Level-B both-sides coverage (binding criterion; threshold 0.95 per subclass):**
  | subclass | both-sides / measured | rate | clears 0.95 |
  |---|---|---|---|
  | UP_DOWN | 13 / 50 | 0.26 | no |
  | OVER_UNDER | 40 / 98 | ≈ 0.4081632653 | no |
  | NAMED_OTHER | 71 / 100 | 0.71 | no |
- **Verdict:** `S1ALT_SOURCE_NOT_VIABLE` — **no subclass cleared the 0.95 both-sides bar.**
- **Diagnostics:** `token_pair_unresolved 0`; `malformed_trade_rows_total 19`;
  `invalid_price_rows_total 0`; `all_one_status_detected false` (a genuinely mixed
  BOTH_SIDES/ONE_SIDE/NEITHER result, not a suspicious uniform output); `level_c_validation`
  **passed**.

Figures reconcile: 248 + 52 = 300; measured 50 + 98 + 100 = 248; Level-B 124 + 65 + 59 = 248;
per-subclass both-sides 13 + 40 + 71 = 124 (= BOTH_SIDES total exactly).

## What it means

Local trade prints (Option A — `Store.load_trades()` reconstruction, no network) do **not**
provide a usable decision-window per-side price for **both** sides across the sampled P0
universe, on this evidence. The decision window is `[first_trade_ts + 3600, resolved_at)`
(leakage-safe, strict `< resolved_at`); a condition only counts as covered if **both** side
tokens have an own-token trade print inside that window. Most sampled conditions do not — the
one-side-only failure mode flagged as most plausible in the approved spec turned out to dominate.

**Consequence:** P1 remains **BLOCKED** with **no `yes_price` fallback** and **no `1 - price`
synthesis**. Two independent candidate per-side price sources — CLOB `/prices-history` (S1) and
local trade prints (S1-ALT / Option A) — are now both closed negative on sampled evidence. The
per-side price source P1 needs is not available from either.

## Scope of the finding (important)

- This is **Pass 1 sampled coverage only.** It measures the same bounded, stratified
  300-condition sample as S1 — **not** the full 39,693-condition universe. The negative was
  accepted as the finding; a **Pass 2 full-universe** run (to test whether the sampled negative
  generalizes) is a **separate, explicitly-authorized** step and is **not** authorized here.
- S2 (a price-source *build* spec) was gated on **favorable** S1 (or S1-ALT) coverage. Coverage
  was unfavorable, so **S2 does not follow** on this evidence.
- **Option B** (Polymarket Data API `/trades`, gap-fill only) remains untried and may be
  considered only through a separate **SPEC ONLY** feasibility review. **Option C** (on-chain
  event reconstruction) is not a next step in this branch and remains out of scope under the
  current guardrails. Nothing about this result authorizes or implies a broader source search.

## Why this verdict is trustworthy (artifact-vs-finding)

The verdict was accepted only after the measurement itself was made healthy — the same
discipline applied to the original S1 result. Four issues were caught and fixed across this
implementation's review cycle, each **before** this result was accepted:

1. **Setup/schema-stop patch.** A missing or malformed required pinned input (the classification
   contract JSON or the resolution-source parquet) could otherwise have flowed through as blank
   `resolved_at` for every condition → all invalid windows →
   `S1ALT_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE` — a **setup failure masquerading as a
   data finding**. Fixed with typed `STOP_CONTRACT_SCHEMA` / `STOP_RESOLUTION_SOURCE_SCHEMA`
   stops that fire before any coverage classification, distinguishable from the pre-existing
   `STOP_STALE_CONTRACT` (version mismatch only).
2. **Sample-reconstruction overlap fix.** The accepted S1 by-condition ledger carries the full
   300-row sample (248 measured + 52 invalid-window), and the same 52 invalid-window ids
   **also** appear in the S1 excluded ledger — an accepted, expected overlap, not a defect. An
   earlier reconstruction wrongly rejected this overlap as `STOP_SAMPLE_IRREPRODUCIBLE`. Fixed by
   building the sample as the union of both ledgers, with the valid/invalid split read from each
   condition's status rather than raw file-disjointness.
3. **Excluded-reason normalization fix.** The accepted S1 excluded ledger's `reason` field may
   bundle the exclusion class with a parenthesized diagnostic suffix in the same string (e.g.
   `"NO_VALID_DECISION_WINDOW_AFTER_WARMUP (first_trade_ts=..., window_seconds=-3444)"`) — an
   exact-match comparison against the bare class token rejected this real, documented shape.
   Fixed by normalizing to the leading class token before comparison, while preserving the raw
   diagnostic string for audit.
4. **`outcome_index` precision-handling fix.** `outcome_index` is a bounded `{0,1}` integer slot
   field (unlike `token_id`, a precision-critical 78-digit identifier); a real trades table
   produced `outcome_index=0.0` (a pandas `int→float64` column upcast triggered by an unrelated
   null elsewhere in the column), which a shared strict-identifier rule wrongly flagged as
   `STOP_PRECISION_LOSS`. Fixed with a separate, narrowly-scoped normalization rule for
   `outcome_index` that safely accepts exact-integral forms (`0`, `1`, `0.0`, `1.0`, `"0"`,
   `"1"`, `"0.0"`, `"1.0"`) without ever treating a domain violation as precision loss, and without
   loosening `token_id`'s strict rule at all.

Each fix was caught by a real user-run halt, diagnosed against the authoritative accepted S1
artifacts and data contracts (not assumed), patched, regression-tested, and re-verified before
the next run — the same "never conclude from an artifact" discipline that produced the trustworthy
S1 result. The accepted run reconciles exactly at every level (sample, measured/excluded,
per-subclass, Level-B totals) and shows a genuinely **mixed** Level-B distribution — not an
all-one-status output — so `level_c_validation` passing here is a real corroboration, not a
formality.

## Artifacts (under `artifacts/named_binary_probe/price_source_s1_alt/`)

Coverage-only; **no price series persisted.**
- `price_source_s1_alt_coverage.json` — machine-readable result: `status`/`verdict =
  S1ALT_SOURCE_NOT_VIABLE`, `sample_reconstruction` (300/248/52, measured_by_subclass), `universe_reconciliation`
  (measured 248, token_pair_unresolved 0, no_trade_anchor, no_valid_decision_window,
  s1_invalid_window 52, reconciled true), `level_b_class_counts` (124/65/59),
  `per_subclass_coverage` (rates + clears_threshold per subclass), `threshold 0.95`,
  `all_one_status_detected false`, `level_c_validation` (passed), `malformed_trade_rows_total 19`,
  `invalid_price_rows_total 0`, `named_binary_probe_blocked_observed true`,
  `no_price_series_persisted true`, `wrote_artifacts true`.
- `price_source_s1_alt_coverage.md` — narrative summary.
- `price_source_s1_alt_coverage_by_condition.csv` — per-condition ledger: condition_id, subclass,
  side tokens, status, Level-B class, window diagnostics (first_trade_ts, decision_lower_ts,
  resolved_at_ts), per-side observation timestamp + gap + print count. **No price values.**
- `price_source_s1_alt_excluded.csv` — excluded conditions with reasons: `TOKEN_PAIR_UNRESOLVED`,
  `NO_TRADE_ANCHOR`, `NO_VALID_DECISION_WINDOW_AFTER_WARMUP` (including the 52 S1-authoritative
  invalid-window conditions, each carrying its preserved raw S1 diagnostic detail string).
- `price_source_s1_alt_source_shape.md` — observed local `Store.load_trades()` column shape (no
  price values recorded).
  Written by `scripts/price_source_s1_alt_pass1_coverage.py` (coverage-only; local-only, two
  independent safety flags gating read vs write; Pass 2 out of scope; no writes to `prices/`;
  `named_binary_probe_blocked` never flipped).

## Guardrails / authorization

No Pass 2, no Option B, no Option C, no S2, no P1/P2/P3, no probe execution, no scoring, no
backfill, no wallet/OrdersMatched/log_index/PnL, no gate change. `named_binary_probe_blocked`
stays `true`. No `yes_price`, `1 - price`, `1 - p`, or `1 - yes_price` synthesis anywhere. No data
run and no rerun performed by this task — this task only records the already-accepted result.

## Next concrete action (separate document; no run authorized here)

This handoff records the accepted S1-ALT Option A negative finding only. The separately prepared
`SPEC_price_source_option_b_data_api_feasibility.md` may be reviewed/pinned as a **SPEC ONLY**
continuation. Accepting or pinning that spec does **not** authorize implementation, API/network
execution, Pass 2, S2, P1/P2/P3, probe execution, scoring, backfill, wallet/OrdersMatched/`log_index`/PnL,
or any gate change.

Absent a separate explicit authorization for a future Option B phase, **P1 stays blocked** and
both already-tested per-side price-source candidates — CLOB `/prices-history` and local trade
prints — are closed negative on Pass 1 sampled evidence.
