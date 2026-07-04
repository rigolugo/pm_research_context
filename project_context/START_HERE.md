# START HERE

*First file to read in any new chat for this project. It orients you to the pinned source-of-truth files, the current branch state, and the one rule that keeps this project consistent.*

---

## Rule 0 — pinned files are the only source of truth

**Old chats are NOT source of truth.** Do not rely on prior-chat memory, summaries, or recollection of "what we decided." Everything necessary must be in the pinned project files listed below. If something you need is not written in a pinned file, treat it as unknown and inspect/verify from source or data — do not reconstruct it from memory. This project has repeatedly been bitten by assumed shapes and assumed facts; the discipline is: **verify against an authoritative source before concluding.**

---

## Required read order

Read these, in this order, before doing anything:

1. `GUARDRAILS.md` — hard constraints (research only; no live/paper trading; no wallet-copying; no PnL-by-role or log_index without explicit authorization; etc.).
2. `PROJECT_STATE.md` — current objective, environment, and what is active/blocked right now.
3. `DECISION_LOG.md` — corrected history; settled decisions not to re-litigate.
4. `CLOSED_FINDINGS.md` — settled negative/complete results (Phase 1, Rank 1A, Rank 2).
5. `ARTIFACT_INDEX.md` — what exists and where (artifacts, scripts, tests, specs).
6. `DATA_CONTRACTS_named_binary_probe.md` — exact inspected schemas/API surfaces for the named-binary probe (contract JSON, resolution parquet, conflicts CSV, p0_preflight.json, `Store`, `named_binary` semantics, Rank 1A warm-up). Authoritative/diagnostic-only/absent tagging.
7. `PRICE_INPUT_CONTRACT_named_binary_probe.md` — the accepted S0 finding on the price input (why P1 is blocked).
8. `SPEC_named_binary_probe.md` — the governing spec-only document for the future offline probe.
9. `SPEC_price_source_s1_coverage.md` — ACCEPTED / SPEC ONLY / coverage-only. The S1 plan for testing whether Polymarket CLOB `/prices-history` per token can cover the P0 universe with a usable decision-time per-side price. Authorizes no implementation, no S1 run, no network/data fetch, no S2, no P1/P2/P3, no probe execution, no scoring, no backfill.
10. The **latest active handoff / review memo** — currently `HANDOFF_orchestrator_named_binary_probe_p1_REVIEW.md` (P1 paused/blocked) and `HANDOFF_orchestrator_named_binary_probe_p0.md` (P0 accepted).

Supporting reference (not overriding the above): `DUNE_DATA_NOTES.md` for Dune schema/export/precision lessons.

`CLAUDE_PROJECT_SETTINGS.md` — operational Claude capability settings only: network allowlist policy, enabled/recommended skills, disabled/not-recommended skills. This file does not override `GUARDRAILS.md`, `PROJECT_STATE.md`, `DECISION_LOG.md`, `CLOSED_FINDINGS.md`, `ARTIFACT_INDEX.md`, `DATA_CONTRACTS_*.md`, `PRICE_INPUT_CONTRACT_*.md`, or active specs. It does not authorize implementation, data fetching, P1/P2/P3 continuation, probe execution, wallet discovery, PnL, paper/live trading, `log_index`, or full indexer work.
 `ORCHESTRATOR_LOW_CONTEXT_MODE.md` — operating protocol for narrow review/decision tasks run without a full repo scan (minimal read policy + APPROVE/BLOCK/DEFER/ACCEPT FINDING/NEEDS VERIFICATION). It does not override the source-of-truth files and authorizes nothing.

---

## Current branch state

- **P0 preflight: ACCEPTED (`P0_CLEAR`).** Read-only universe verification; emitted eligible/excluded counts only (pooled: contract_eligible 39,957 / resolved 39,693 / ambiguous 253 / source_rows 39,946 / missing 11 / final_p0_eligible 39,693). Authorizes nothing further.
- **P1 (feature assembly): BLOCKED on missing per-side price input.** The accepted semantics layer's `OrientationContract.canonical_side_price(side_0_price, side_1_price)` needs **two per-side prices**; local `Store.load_prices()` returns only `[condition_id, ts, yes_price]`. Proven from source (S0): `yes_price` is a **YES/NO-only** construction, explicitly **unsafe** for UP_DOWN / OVER_UNDER / NAMED_OTHER, and **no local per-side/per-token price series exists**. The price *formula* is resolved; the price *input* is not derivable locally.
- **P2 / P3 / probe execution: UNAUTHORIZED.** `named_binary_probe_blocked = true` in all gate states. A `CLEAR_WITH_WARNINGS` outcome source does not authorize a probe.
- **Chat2 Dune wallet-cohort discovery: BLOCKED** (separate phase; outcome-source scoreability does not unblock it).

- **S1 price-source coverage spec: ACCEPTED / SPEC ONLY.** `SPEC_price_source_s1_coverage.md` is the accepted, coverage-only plan for testing whether Polymarket CLOB `/prices-history` per token can cover the P0 universe (39,693 conditions) with a usable decision-time per-side price. It is the first stage of the "Next possible step" below. **It authorizes nothing:** no implementation, no S1 run, no network/data fetch, no S2, no P1/P2/P3, no probe execution, no scoring, no backfill. It does **not** unblock P1 by itself — it only measures whether a candidate per-side price source is viable; a NEGATIVE result leaves P1 blocked with no `yes_price` fallback.

### Next possible step — only if explicitly authorized by the user
A **spec-only price-source build plan** for a per-side / token-identity-keyed decision-time price series (e.g. `[condition_id, ts, outcome_index/token_id, side_price]` from CLOB/print history per token), analogous to the Stage 0–4 non-YES/NO resolution-source build. The S1 coverage spec (accepted, above) is the first stage of this; a later S2 build spec would follow only if S1 coverage is favorable and separately authorized. **Spec only** — no implementation, no data run, no backfill, no scoring, no probe. P1 cannot resume until such a price source exists and is audit-checked in a later, separately authorized stage.

---

## What is NOT authorized (standing)

No P1/P2/P3 or probe execution. No scoring (Brier/log-loss/calibration/reliability/splits). No wallet/OrdersMatched/log_index/PnL work. No gate modification. Do not flip `named_binary_probe_blocked`. No live/paper trading, no wallet-copying, no full indexer. See `GUARDRAILS.md` for the complete list.

---

## Working discipline (short form)

- Verify before concluding; never conclude from one row or an all-one-role/all-one-direction output.
- Spec before implementation; get it reviewed before writing code for a new branch.
- Never silently reverse a prior decision — say so explicitly and show evidence.
- Close each task with a Claude-to-Orchestrator handoff memo.
- Files are delivered for the user to copy into `C:\b1\pm_research`; Claude's sandbox has no guaranteed network/pytest/pyarrow, so real-data runs happen on the user's Windows box.
