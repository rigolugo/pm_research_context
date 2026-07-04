# HANDOFF — Claude → Orchestrator: Named-Binary Probe Stage P0 Preflight

**Task:** Implement Stage P0 preflight only, per `SPEC_named_binary_probe.md` §11 (Stage P0) and `PROJECT_STATE.md` "Next possible step". Read-only except for the three P0 artifacts it writes.

> **Patch (count-definition fix).** An earlier P0 build defined `source_rows` as
> RESOLVED_SINGLE_WINNER rows only, which halted real data with
> `STOP_COUNT_RECONCILIATION_FAILED` against the Stage 4 gate. Corrected to match
> Stage 4: **`source_rows = resolved_single_winner + ambiguous_multiple_winners`**
> (rows *found* in the outcome source) and **`missing_source_rows = contract_eligible − source_rows`**.
> The resolution-source parquet holds RESOLVED_SINGLE_WINNER rows only
> (ARTIFACT_INDEX), so ambiguous is counted from
> `named_binary_resolution_conflicts.csv` when present, else from the gate
> `per_subclass_breakdown` (flagged `ambiguous_source = gate_breakdown_only`,
> and those gate-derived fields are then not treated as an independent
> reconciliation check). `final_p0_eligible = resolved_single_winner` for
> scoreable subclasses. Verified to reproduce the expected real-data pooled
> figures exactly: eligible 39957, resolved 39693, ambiguous 253, source 39946,
> missing 11, final 39693 → `P0_CLEAR`, reconciliation exact.

**Scope honored (and NOT exceeded):**
- Stage P0 ONLY. No P1/P2/P3/P4. No scoring (no Brier / log-loss / calibration / reliability / edge / ROI / PnL / fees / sizing / execution).
- No trades, no prices, no decision timestamps, no `canonical_side_price`, no outcome→feature joins, no train/test splits.
- No Dune/network, no `log_index`, no wallet logic, no OrdersMatched features.
- The named-binary audit gate is **not modified**. `named_binary_probe_blocked` is **observed and echoed, never flipped**. `probe_execution_authorized` is hard-wired `false`.

## Deliverables

1. `scripts/named_binary_probe_p0_preflight.py` — the read-only preflight.
2. `tests/test_named_binary_probe_p0_preflight.py` — 20 tests covering every stop state + exclusion rule + the read-only guarantees.
3. Artifacts written at runtime under `artifacts/named_binary_probe/`:
   - `p0_preflight.json` (machine-readable, full schema below)
   - `p0_preflight.md` (narrative)
   - `p0_excluded_counts.csv` (per-subclass tally)
4. This handoff memo.

## What P0 verifies (in order; first failure halts with a typed state)

1. **Loads** contract JSON, audit-gate JSON, resolution-source parquet rows (and optional conflicts CSV). Missing/empty source → `STOP_MISSING_OUTCOME_SOURCE`.
2. **Precision discipline first.** Token-id / outcome-index fields are forced to `string` and run through `canonical_int`; scientific-notation or float-mangled values (`5.20896e+76`, `0.0`, any float) → `STOP_PRECISION_LOSS`. Never reconstructs (DUNE_DATA_NOTES §5).
3. **Version equality.** Contract `nb_contract_version` must equal `nb-contract-2026-06-28.1` AND equal the version stamped on the resolution-source rows. Any mismatch / missing / mixed → `STOP_STALE_CONTRACT`.
4. **Gate state.** `stage4_nonyesno_branch.non_yesno_gate_state` must be `CLEAR_WITH_WARNINGS`; per-subclass scoreability is **read from the gate, never assumed**. Missing branch / not-clear → `STOP_DATA_GATE_NOT_CLEAR`.
5. **Counts only** (no scoring): per subclass (`UP_DOWN`, `OVER_UNDER`, `NAMED_OTHER`) and pooled — `contract_eligible`, `source_rows`, `resolved_single_winner`, `missing_source_rows`, `ambiguous_multiple_winners`, `non_scoreable`, `final_p0_eligible`. Definitions: `source_rows = resolved + ambiguous`; `missing_source_rows = contract_eligible − source_rows`; `final_p0_eligible = resolved` for scoreable subclasses. `YES_NO` is excluded; ambiguous / missing / non-scoreable are excluded **and counted separately** (missing is derived, never relabeled ambiguous — DECISION_LOG Stage 4 lesson). Pooled sums subclass counts after YES_NO exclusion.
6. **Reconciliation** against the gate's `per_subclass_breakdown`, like-for-like on `contract_eligible`, `source_rows`, `resolved_single_winner`, `ambiguous_multiple_winners`, `missing_source_rows`, `final_p0_eligible`. A hard disagreement on a core field → `STOP_COUNT_RECONCILIATION_FAILED`. When ambiguous came from the gate (`gate_breakdown_only`), the gate-derived fields are reported but not counted as an independent check. If the gate lacks row-level breakdown, reconciliation is reported as not-exact (never silently accepted).

`p0_state` ∈ {`P0_CLEAR`, `STOP_MISSING_OUTCOME_SOURCE`, `STOP_STALE_CONTRACT`, `STOP_PRECISION_LOSS`, `STOP_DATA_GATE_NOT_CLEAR`, `STOP_COUNT_RECONCILIATION_FAILED`}.

## `p0_preflight.json` schema

`stage`, `p0_state`, `authorized_scope`, `probe_execution_authorized`, `named_binary_probe_blocked_observed`, `nb_contract_version_expected`, `nb_contract_version_contract`, `nb_contract_version_resolution_source`, `gate_snapshot`, `counts_pooled`, `counts_by_subclass`, `excluded_counts`, `reconciliation`, `notes`, `created_at_utc`.

## Verification done in Claude's sandbox

The sandbox lacks `pyarrow`/`pytest`, so the parquet read and `pytest` runner could not execute here. Logic was verified by a pickle-backed harness exercising the same `run_preflight` path: **37/37 checks pass**, covering every stop state, the corrected `source_rows = resolved + ambiguous` / `missing = eligible − source_rows` definitions (with the regression case: resolved-only in parquet + ambiguous in conflicts still yields the right source_rows), the conflicts-CSV-absent gate fallback, the YES_NO / ambiguous / missing / non-scoreable exclusions, the reconciliation hard-fail, artifact emission, and `canonical_int` precision rejection. A separate real-data-scale check reproduced the expected pooled figures exactly (eligible 39957 / resolved 39693 / ambiguous 253 / source 39946 / missing 11 / final 39693 → `P0_CLEAR`). The committed `tests/…` file uses real parquet + `pytest` and should be run in the `pmresearch` env.

## Run locally (Windows / Miniconda, env `pmresearch`)

```powershell
$env:PYTHONPATH="C:\b1\pm_research"
cd C:\b1\pm_research
python scripts\named_binary_probe_p0_preflight.py --root C:\b1\data --artifacts-dir artifacts
```

(`--root` is accepted for command-shape compatibility but **unused** — P0 reads no trades/prices.)

Run only the new tests:

```powershell
python -m pytest tests\test_named_binary_probe_p0_preflight.py -q
```

## Open items / guardrail reminders

- P0 clearing **does not** authorize the probe. Running P1+ requires a **separate, explicit, in-chat execution authorization** (SPEC §9.7 `STOP_NOT_AUTHORIZED`); P0 deliberately implements no such flag.
- Chat2 Dune wallet discovery remains BLOCKED and is untouched here.
- If P0 emits any `STOP_*`, do not proceed — fix the upstream artifact and re-run. A round-number or all-one-direction count is a red flag, not a result.
