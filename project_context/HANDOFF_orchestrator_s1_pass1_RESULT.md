# HANDOFF — Claude → Orchestrator: S1 Pass 1 coverage RESULT (ACCEPTED)

**Status:** S1 Pass 1 CLOB `/prices-history` coverage test **completed (user-run) and accepted
as a sampled coverage finding.** Documentation-only task: records the result in PROJECT_STATE,
ARTIFACT_INDEX, and DECISION_LOG, and captures the finding here. No code, no rerun, no network,
no gate change, `named_binary_probe_blocked` not flipped.

## Result — `S1_SOURCE_NOT_VIABLE`

Pass 1 **sampled** coverage (not Pass 2 full-universe):

- **Sample:** 300 / 300 conditions.
- **Valid-window conditions measured:** 248. **Invalid-window excluded + reported:** 52
  (`resolved_at` missing or `<= first_trade_ts + warmup`; excluded as
  `NO_VALID_DECISION_WINDOW_AFTER_WARMUP`, never counted as coverage negatives).
- **Side-token fetches:** 496 (= 248 valid conditions × 2 sides).
- **Endpoint shape:** parsed cleanly — `GET /prices-history`, `history` list, point keys
  `p` / `t`, no deviation.
- **Level-B both-sides coverage (binding criterion; threshold 0.95 per subclass):**
  | subclass | both-sides / measured | rate | clears 0.95 |
  |---|---|---|---|
  | UP_DOWN | 19 / 50 | 0.38 | no |
  | OVER_UNDER | 51 / 98 | ≈ 0.5204 | no |
  | NAMED_OTHER | 65 / 100 | 0.65 | no |
- **Verdict:** `S1_SOURCE_NOT_VIABLE` — **no subclass cleared the 0.95 both-sides bar.**

Figures reconcile: 248 + 52 = 300; 248 × 2 = 496; measured 50 + 98 + 100 = 248.

## What it means

CLOB `/prices-history` does **not** provide a usable decision-window per-side price for **both**
sides across the sampled P0 universe. The decision window is `[first_trade_ts + 3600,
resolved_at)` (leakage-safe, strict `< resolved_at`); a condition only counts as covered if
**both** side tokens have a point inside that window. Most sampled conditions do not.

**Consequence:** P1 remains **BLOCKED** with **no `yes_price` fallback** (the local `yes_price`
is YES/NO-only and unsafe for named binaries — see `PRICE_INPUT_CONTRACT_named_binary_probe.md`).
The per-side price source that P1 needs is not available from `/prices-history` on this evidence.

## Scope of the finding (important)

- This is **Pass 1 sampled coverage only.** It measures a bounded, stratified 300-condition
  sample — **not** the full 39,693-condition universe. The negative was accepted as the finding;
  a **Pass 2 full-universe** run (to test whether the sampled negative generalizes) is a
  **separate, explicitly-authorized** step and is **not** authorized here.
- S2 (a price-source *build* spec) was gated on **favorable** S1 coverage. Coverage was
  unfavorable, so **S2 does not follow** on this evidence.

## Why this verdict is trustworthy (artifact-vs-finding)

The verdict was accepted only after the measurement was made healthy. Three earlier
`S1_SOURCE_NOT_VIABLE` outputs were **measurement artifacts**, each caught and fixed before
acceptance:
1. a request-window bug collapsing the endpoint query to a **1-second window** (all
   `SERIES_EMPTY`);
2. an **invalid-decision-window** case where `resolved_at`-missing conditions were coerced into
   1-second queries and mislabeled `DECISION_PRICE_NEITHER` — fixed by excluding + reporting them
   and returning `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE` for an all-invalid sample
   instead of a false NOT_VIABLE;
3. a `parse_ts` gap that rejected the **real** `resolved_at` string form
   `"YYYY-MM-DD HH:MM:SS.mmm UTC"`, blanking every window.

`request_window_summary` (whole-sample window distribution) and the `decision_window_validity`
split were added so a degenerate window can never masquerade as a coverage negative. The accepted
run shows **wide valid windows** and **248 genuinely measured conditions** — a real coverage
negative, consistent with the project's "never conclude from an all-one output without a
validation pass" discipline. (An earlier attempt to record NOT_VIABLE from the stale 1-second
artifact was correctly refused until these healthy-window figures existed.)

## Artifacts (under `artifacts/named_binary_probe/price_source_s1/`)

Coverage-only; **no price series persisted.**
- `price_source_s1_coverage.json` — machine-readable result (verdict, `decision_window_validity`
  248/52, `request_window_summary`, `fetched_token_count` 496, `per_subclass_coverage`,
  `level_a_status_counts`, `level_b_class_counts`, `level_c_validation`, reconciliation,
  `pass2_available false`).
- `price_source_s1_coverage.md` — narrative summary.
- `price_source_s1_coverage_by_condition.csv` — per-condition coverage + window diagnostics
  (no price values).
- `price_source_s1_excluded.csv` — `TOKEN_PAIR_UNRESOLVED` + `NO_VALID_DECISION_WINDOW_AFTER_WARMUP`.
- `price_source_s1_endpoint_shape.md` — documented-vs-observed shape; Level-A scope note;
  whole-sample request-window summary.

## Guardrails / authorization

No Pass 2, no S2, no P1/P2/P3, no probe execution, no scoring, no backfill, no
wallet/OrdersMatched/log_index/PnL, no gate change. `named_binary_probe_blocked` stays `true`.
No network run and no rerun performed by this task.

## Next concrete action (only if separately authorized)

If the Orchestrator wants to know whether the sampled negative generalizes, the **only** candidate
next step is a **Pass 2 full-universe coverage run** of the same coverage-only tool (network
hard-gated, user-run) — explicitly authorized, spec-referenced, and still non-probe. Absent that,
P1 stays blocked and the CLOB `/prices-history` per-side price line is closed negative on Pass 1
evidence.
