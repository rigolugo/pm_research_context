# HANDOFF (REVIEW) — Claude → The Orchestrator: Named-Binary Probe Stage P1

**Task chat:** Named-binary probe P1 (feature assembly).
**Status:** PAUSED at user request for Orchestrator review. Implementation NOT complete on real data — three real-run integration errors encountered, two fixed, one open. Pure-logic tests pass (25/25). No real-data P1 artifacts produced yet.
**Scope unchanged:** P1 only. No scoring, no probe execution, no wallet/OrdersMatched/log_index/PnL, gate untouched, `named_binary_probe_blocked` not flipped. **P2 remains unauthorized.**

---

## Why this memo exists

P1's *logic* verifies in-sandbox (25 pure tests on injected frames), but the sandbox has no pyarrow, no repo modules, and no real artifacts, so every assumption about the **exact shape of a real artifact or loader** could only be tested on the user's Windows box. Three such assumptions were wrong. This is the same failure class the project already knows well (assume-a-shape → fails on real data), so pausing to let you review the artifact contracts before I hardcode more assumptions is the right call.

---

## Files as they currently stand

| File | State |
|---|---|
| `scripts/named_binary_probe_p1_features.py` | Runs through P0 verification, version pins, gate check, universe build start; **halts at contract row access** (Error 3 below). Store wiring fixed. |
| `tests/test_named_binary_probe_p1_features.py` | 25 pure-logic tests, all passing in-sandbox (via direct runner; pytest not in sandbox). Uses injected list[dict] loaders with the ASSUMED shapes — so tests did NOT catch the real-shape mismatches. |

**Important caveat about the tests:** they encode the *assumed* contract shape (`row["subclass"]`, `row["eligible"]`, module-level store loaders). They pass because the fixtures match the assumptions, not the real artifacts. After the contract shape is confirmed, the fixtures should be updated to mirror the real JSON so they actually guard against this.

---

## Real-run errors, in order

### Error 1 — store loader import (FIXED)
```
ImportError: cannot import name 'load_prices' from 'pm_research.data.store'
```
- **Cause:** wiring assumed module-level `load_prices`/`load_trades`. Real module exposes a `Store` class.
- **Confirmed real surface:** `Store(root)` with methods `load_trades(wallets: list[str] | None = None) -> DataFrame`, `load_prices() -> DataFrame`, `load_resolutions()`, `load_markets()`, `has_prices(condition_id)`, `coverage_ok(...)`, plus `save_*`.
- **Fix applied:** `build_real_loaders` now does `Store(root).load_prices()` / `.load_trades()` and filters to eligible condition_ids in-memory (no per-condition push-down exists on the store).

### Error 2 — prices schema has no token columns (FIXED, design consequence)
- **Confirmed:** `load_prices()` returns `[condition_id, ts, yes_price]` only — no `token_id` / `outcome_index` on price rows.
- **Consequence:** the per-price orientation audit fields cannot be cross-checked at the price-row level; orientation must come entirely from the semantics layer keyed on condition (as designed). Wiring now injects null `token_id`/`outcome_index` columns if absent so downstream `.get()` is uniform. **Open handoff question 3 (price-row token identity) is now answered: not available; do not rely on it.**

### Error 3 — contract row has no `subclass` key (OPEN — HALTS HERE)
```
File ".../named_binary_probe_p1_features.py", line 299, in assemble_features
    sub = row["subclass"]
KeyError: 'subclass'
```
- **Cause:** P1 assumed `named_binary_classification_contract.json` is (or contains) a list of per-condition records each with literal keys `condition_id`, `subclass`, `eligible`. The real JSON's shape/keys differ.
- **NOT yet fixed** (paused per user).

---

## What The Orchestrator needs to resolve (Error 3)

The fix is a data-contract question, not a logic question. Please confirm the real shape of `artifacts/named_binary/named_binary_classification_contract.json`:

1. **Top-level shape:** is it a list of condition records, or a dict keyed by `condition_id`, or a dict with a `conditions`/`records`/`rows` field wrapping the list, or a `{condition_id: {...}}` map? (P1 currently only unwraps a top-level `conditions` key, else assumes a bare list.)
2. **Per-condition key names:** what are the actual field names for
   - the condition id (assumed `condition_id`),
   - the subclass / category (assumed `subclass`; may be `named_binary_subclass`, `class`, `category`, `nb_subclass`, or nested),
   - the eligibility flag (assumed `eligible`; may be `is_eligible`, `probe_eligible`, an enum like `status`, or absent — in which case eligibility is defined some other way).
3. **Subclass vocabulary:** are the subclass values exactly `UP_DOWN` / `OVER_UNDER` / `NAMED_OTHER` / `YES_NO`, or different spellings/casing?

### Strongly preferred alternative (avoids the contract-shape guesswork entirely)
ARTIFACT_INDEX line 33 confirms the **Stage 3 resolution-source parquet already carries `subclass` per row**:
`named_binary_resolution_source_rows.parquet = [condition_id, subclass, resolved_winning_token_id, resolved_outcome_index, resolved_label, resolved_at, source_table, status, nb_contract_version]`.

Since P1's eligible universe is exactly the RESOLVED_SINGLE_WINNER rows anyway, **subclass (and status, resolved_at, winner) can come from the resolution-source parquet rather than the contract JSON.** That would let the contract JSON be used only for version-equality assertion (which only needs `nb_contract_version`, already working), sidestepping its per-condition shape completely.

**Recommended direction (for your approval):** drive the eligible universe from `named_binary_resolution_source_rows.parquet` (subclass + status + resolved_at + winner all present there, string-safe), and use the contract JSON solely for the `nb_contract_version` pin. If you approve, I'll refactor `assemble_features` to read subclass/status/resolved_at from the resolution rows and drop the contract per-condition field access — then update the test fixtures to the real shapes.

Open question still outstanding from before (unrelated to the errors):
- **Oriented-winner source:** does `named_binary_resolution_source_rows.parquet` carry a precomputed `oriented_side_won`, or must P1 map the winner to the canonical side via the Stage 2 `resolution_source.py` slot→token mapping? Needed so the held-aside target (`y_target_target_only_not_feature`) is derivable. If `nb.canonical_side_price` returns a bare float (not a dict with `canonical_side_token_id`), the token-id fallback can't fire and the target is undeterminable — which would surface as all-zero y1/y0 and (correctly) trip the all-one-direction guard.
- **Warm-up constant:** P1 defaults to 3600s. If `scripts/forecast_vs_price.py` (Rank 1A) exposes a shared warm-up constant/policy, point me to it so P1 imports it and cannot drift from Rank 1A.

---

## Confirmed-good so far (real run reached these without error)
- P0 verification path (reads `p0_preflight.json`, requires `P0_CLEAR` + `P0_PREFLIGHT_ONLY`).
- Contract version pin and resolution-source version pin (import + read succeeded up to the row-field access).
- Gate read + `non_yesno_gate_state == CLEAR_WITH_WARNINGS` check.
- `Store(root)` construction and loader calls (no import error second time).

## Guardrail status
Intact. No gate modification, no `named_binary_probe_blocked` flip, no scoring code, no wallet/OrdersMatched/log_index paths, `probe_execution_authorized`/`scoring_authorized` hard-false. No real-data P1 artifacts were written (halt occurred before assembly completed), so nothing partial is masquerading as a result.

## Next actions (awaiting Orchestrator decision — NOT started)
1. Approve driving the eligible universe from the resolution-source parquet (recommended) vs. supplying the exact contract-JSON shape.
2. Confirm oriented-winner source (precomputed flag vs. Stage 2 slot→token mapping) and `nb.canonical_side_price` return type.
3. Confirm/point to the Rank 1A warm-up constant.
Once resolved, I'll finish Error 3, align the test fixtures to real shapes, and re-run. Still P1 only; P2 remains unauthorized.
