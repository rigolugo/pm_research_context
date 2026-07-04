# ARTIFACT INDEX

*What exists and where. Paths are in the local repo `C:\b1\pm_research` unless noted.*

---

## Artifacts (`artifacts/`)

Recommended subfolder organization (currently flat — reorganize when convenient):

### `artifacts/rank1a/`
- `forecast_vs_price_rank1a.json` + `.md` report + `_splits.csv` + `_conditions.csv` — Rank 1A probe (NEGATIVE).
- `rank1a_price_bucket_diagnostic.json` + `.md` + `.csv` — NO_ACTIONABLE_BUCKET.

### `artifacts/audits/`
- `probe_fast_gate.*` — data gate result CLEAR_WITH_WARNINGS_FOR_RANK1A.
- (audit_market_structure outputs, if retained.)

### `artifacts/rank2_ordersmatched/`
- `ordersmatched_economic_role_expanded.json` + `.md` + `.csv` — Rank 2 final distribution (117 trades, 96 wallets, 71T/46M).
- `ordersmatched_maker_pairing_validation.json` + `.md` + `.csv` — MAKER_PAIRING_VALIDATED.
- `ordersmatched_economic_role.{json,md,csv}` — the pre-expansion probe.
- `ordersmatched_expanded_dune_query.sql` (varchar-cast, 150-tx) and `ordersmatched_expanded_dune_tx_list.txt`.
- `orderfilled_operator_filtered_semantic.*` — event-role recovery + operator filter.
- `orderfilled_fee_role_diagnostic.*` — fee diagnostic (SUPERSEDED, preserved).

### `artifacts/named_binary/`
- `named_binary_audit_gate.json` + `.md` — default/local-only named-binary semantics audit gate: legacy pooled-all gate BLOCKED_BY_RESOLUTION_MAPPING; schema_chosen=label; YES/NO resolves 8,521/8,521 where local resolution rows exist; non-YES/NO unresolved locally. When run with `--resolution-source`, the same gate artifact also includes `stage4_nonyesno_branch` fields described below.
- `named_binary_resolution_mapping_coverage.csv` — per-subclass coverage table. In the local-only audit it reports eligible / with-resolution / winner-maps / rate; after Stage 3/4 it extends this with source_rows, resolved_single_winner, conflicts-by-type, exact_winner_rate, and coverage_rate.
- `named_binary_classification_contract.json` + `.md` — per-condition subclass + eligibility; carries `nb_contract_version` (`nb-contract-2026-06-28.1`). **Chat2 consumes this and asserts version equality.**
- `named_binary_semantics_report.md` — narrative summary.
- `named_binary_label_pair_census.csv` — frequency-ranked label-pair census (11,598 pairs; head covered by seed lexicon).
- `named_binary_resolution_source_rows.parquet` — Stage 3 normalized winners (condition_id, subclass, resolved_winning_token_id/outcome_index/label, resolved_at, source_table, status, nb_contract_version); string-safe token fields; RESOLVED_SINGLE_WINNER rows only.
- `named_binary_resolution_conflicts.csv` — Stage 3 excluded conflicts with status + payout vector.
- `named_binary_resolutions_source_audit.json` + `.md` — Stage 3 build audit (per-subclass resolved/conflict counts).

### `artifacts/named_binary_probe/`
- `p0_preflight.json` — Stage P0 preflight result (machine-readable): `p0_state` (P0_CLEAR), `authorized_scope` (P0_PREFLIGHT_ONLY), `probe_execution_authorized` (false), `named_binary_probe_blocked_observed` (true), contract/resolution-source versions, `gate_snapshot`, `counts_pooled`, `counts_by_subclass`, `excluded_counts`, `reconciliation`, notes. Accepted real-data figures: contract_eligible 39,957 / resolved_single_winner 39,693 / ambiguous_multiple_winners 253 / source_rows 39,946 / missing_source_rows 11 / final_p0_eligible 39,693.
- `p0_preflight.md` — Stage P0 narrative report (same figures + gate snapshot + reconciliation).
- `p0_excluded_counts.csv` — per-subclass tally (source_rows / resolved / ambiguous / missing / non_scoreable / final_p0_eligible).
  Written by `scripts/named_binary_probe_p0_preflight.py`. Read-only preflight; emits counts only; authorizes no P1/P2/P3 and no probe execution.
- `price_input_s0_inspection.txt` — read-only S0 price-input inspection evidence (produced by `scripts/inspect_price_input_s0.py`). Records price provenance (`backfill.py` builds `yes_price`), the YES/NO-only / named-binary-unsafe meaning of `yes_price`, the absence of any local per-side/per-token price artifact, and a non-YES/NO price sample. Evidence only — no code, no gate, no probe change.

### `artifacts/named_binary_probe/price_source_s1/`
S1 Pass 1 **coverage-only** artifacts (user-run, ACCEPTED; verdict `S1_SOURCE_NOT_VIABLE`). Coverage diagnostics only — **no price series is persisted**; the by-condition CSV carries statuses/windows/token-ids/gaps/point-counts but never a reusable price series.
- `price_source_s1_coverage.json` — machine-readable Pass-1 result: `s1_verdict = S1_SOURCE_NOT_VIABLE`, `sample_size_actual 300`, `decision_window_validity` (valid_window 248 / invalid_window 52, per-subclass valid/invalid counts), `request_window_summary` (wide windows on valid conditions), `fetched_token_count 496`, `level_a_status_counts`, `level_b_class_counts`, `per_subclass_coverage` (UP_DOWN 19/50 = 0.38, OVER_UNDER 51/98 ≈ 0.5204, NAMED_OTHER 65/100 = 0.65; threshold 0.95; none clear), `level_c_validation`, `universe_reconciliation`, `pass2_available false`, `named_binary_probe_blocked_observed true`.
- `price_source_s1_coverage.md` — narrative summary (verdict, per-subclass coverage, request-window + decision-window-validity summaries, honest framing).
- `price_source_s1_coverage_by_condition.csv` — per-condition coverage ledger: condition_id, subclass, side tokens, Level-A per side, reachability, Level-B class, nearest-gap per side, and window diagnostics (first_trade_ts, decision_lower_ts, resolved_at_ts, request_start_ts, request_end_ts, request_window_seconds, observed_point_count per side). **No price values.**
- `price_source_s1_excluded.csv` — excluded conditions with reasons: `TOKEN_PAIR_UNRESOLVED` (with malformed-trade-row counts) and `NO_VALID_DECISION_WINDOW_AFTER_WARMUP` (with first_trade_ts / decision_lower_ts / resolved_at_ts / window_seconds).
- `price_source_s1_endpoint_shape.md` — documented-vs-observed endpoint shape: `GET /prices-history` parsed cleanly (`history` list, point keys `p`/`t`, no deviation); Level-A scope note (status reflects the queried decision window, not a broader reachability probe); whole-sample request-window summary so a single observed GET is not mistaken for the sample.
  Written by `scripts/price_source_s1_coverage.py` (coverage-only; network hard-gated behind explicit CLI flags; Pass 2 hard-stopped; no writes to `prices/`; `named_binary_probe_blocked` never flipped).


---

## Scripts (`scripts/`)

### Rank 1A
- `forecast_vs_price.py` — pooled YES/NO forecast-vs-price probe (Rung 0 + Rung 1).
- `rank1a_price_bucket_diagnostic.py` — price-bucket segmentation diagnostic.

### Audit / gate
- `audit_market_structure.py` — streaming market-structure audit.
- `probe_fast_gate.py` — overlap/concentration data gate.

### Rank 2 (OrdersMatched / OrderFilled)
- `orderfilled_sample_join.py` — **validated decoder** (topic2=maker, topic3=taker; canonical-int helpers live here / in the OrdersMatched module).
- `ordersmatched_economic_role.py` — economic-role probe (taker via takerordermaker; maker via pairing). Contains `canonical_int`, `DataExportPrecisionLoss`, varchar Dune query template.
- `ordersmatched_economic_role_expanded.py` — stratified 150-tx expansion + per-wallet/contract/time/fill breakdowns.
- `ordersmatched_maker_pairing_validation.py` — known-maker-case validation harness.

### Named-binary semantics
- `audit_named_binary_semantics.py` — orchestration entry point; streams sharded trades (batch reader + progress ticks), runs mapping/classification/resolution/orientation/gate, writes named-binary artifacts. Layer ends at the gate (no probe). **Stage 4: optional `--resolution-source <rows.parquet>` flag** overlays the Stage 3 RESOLVED_SINGLE_WINNER winners onto non-YES/NO subclasses (YES/NO stays on the local path; conflicts excluded + counted). The legacy pooled-ALL `gate_state` stays honest (BLOCKED_BY_RESOLUTION_MAPPING — YES_NO local coverage is sparse, ~0.13). A separate **non-YES/NO branch gate** (`stage4_nonyesno_branch`) applies the orchestrator two-tier threshold: non-YES/NO pooled exact-winner rate ≥ 0.99 AND each subclass ≥ 0.95 → `non_yesno_gate_state = CLEAR_WITH_WARNINGS`; any subclass < 0.95 marks that subclass not scoreable. On real data: pooled 0.99339, UP_DOWN 0.99995 / NAMED_OTHER 0.98657 / OVER_UNDER 0.96535 (all ≥ 0.95) → CLEAR_WITH_WARNINGS. CLEAR_WITH_WARNINGS means the outcome source/audit is usable and never authorizes a probe (`named_binary_probe_blocked` stays True).
- `named_binary_label_census.py` — frequency-ranked label-pair census feeding the lexicon.
- `diag_resolution_mapping.py`, `diag_resolution_coverage.py`, `diag_named_binary_resolvable.py` — read-only diagnostics that isolated the YES/NO-only resolutions finding.
- `inspect_named_binary_resolution_dune_schema.py` — Stage 0 read-only Dune schema inspection (emits information_schema SQL, precision-checks samples).
- `export_contract_for_dune.py` — exports the classification contract to a Dune-upload CSV; validates condition_id format (fails loud on non-0x-lowercase).
- `build_named_binary_resolution_source.py` — Stage 3 bulk builder: consumes Stage 1 Dune CSVs + contract + streamed sides, runs Stage 2 logic, writes normalized winners + conflicts + coverage + audit. No gate integration, no probe.

### Named-binary probe (Stage P0 only)
- `named_binary_probe_p0_preflight.py` — Stage P0 preflight (ACCEPTED, P0_CLEAR). Read-only except for its three `artifacts/named_binary_probe/` outputs. Loads contract + gate + resolution-source rows (+ optional conflicts CSV); asserts `nb_contract_version` equality (`nb-contract-2026-06-28.1`, contract == resolution-source), `stage4_nonyesno_branch.non_yesno_gate_state == CLEAR_WITH_WARNINGS`, per-subclass scoreability read from the gate, source non-empty, and token/id precision safety (fail-loud on scientific notation). Counts ONLY: `source_rows = resolved_single_winner + ambiguous_multiple_winners` (resolved from the source parquet; ambiguous from conflicts CSV, else gate `per_subclass_breakdown` flagged `gate_breakdown_only`), `missing_source_rows = contract_eligible − source_rows`, `final_p0_eligible = resolved` for scoreable subclasses; YES_NO excluded; pooled sums subclasses. Reconciles like-for-like against the gate breakdown (hard-fail → `STOP_COUNT_RECONCILIATION_FAILED`). Typed halts: `STOP_MISSING_OUTCOME_SOURCE` / `STOP_STALE_CONTRACT` / `STOP_PRECISION_LOSS` / `STOP_DATA_GATE_NOT_CLEAR` / `STOP_COUNT_RECONCILIATION_FAILED`. No trades/prices, no decision timestamps, no canonical_side_price, no scoring, no Dune/network, no log_index, no wallet logic. `probe_execution_authorized` hard-wired false; `named_binary_probe_blocked` observed, never flipped; gate not modified. CLI: `--root` (accepted for command-shape parity, unused) `--artifacts-dir`.

### Named-binary probe (S1 price-source coverage — coverage-only, user-run)
- `price_source_s1_coverage.py` — S1 Pass 1 CLOB `/prices-history` coverage test (ACCEPTED run; verdict `S1_SOURCE_NOT_VIABLE`). **Coverage-only:** enumerates side-token pairs from `trades` distinct `(condition_id, token_id, outcome_index)` (exactly two stable string-safe tokens, never `resolved_winning_token_id`); builds a deterministic subclass/time-stratified Pass-1 sample bounded by `--sample-size` (default 300, never the full universe); for **valid-window** sampled conditions only (skips + reports `NO_VALID_DECISION_WINDOW_AFTER_WARMUP` when `resolved_at` is missing or `<= first_trade_ts + warmup`), fetches both side tokens over the decision window `[floor(first_trade_ts + 3600), ceil(resolved_at)]`; classifies Level A (per-token reachability of the queried window) and Level B (both-sides decision-window density, binding at 0.95 per subclass, strict `ts < resolved_at`); runs a Level-C validation sample before trusting all-one aggregates; emits coverage-only artifacts. **Network is hard-gated** behind two explicit CLI flags (`--i-authorize-s1-pass1-network-run` + `--confirm-external-host clob.polymarket.com`); default prints `STOP_NOT_AUTHORIZED` and writes nothing. **Pass 2 is hard-stopped** (`run_pass2` raises). Emits a stderr `[S1]` progress heartbeat (`--quiet` / `--progress-every`). No `yes_price`/`1 - yes_price`/`1 - price`/`1 - p` synthesis; no writes to `prices/`; no scoring/backfill; `named_binary_probe_blocked` never flipped. `parse_ts` accepts the real resolution-source form `"YYYY-MM-DD HH:MM:SS.mmm UTC"` plus ISO / pandas Timestamp / datetime / epoch; `load_resolution_rows` preserves `resolved_at` (does not stringify-mangle it) and fails loud (`STOP_RESOLUTION_SCHEMA`) if the column is absent.

### Named-binary probe (read-only inspection scripts — no code/gate/probe changes)
- `inspect_named_binary_probe_data_contracts.py` — read-only inspector that captured the exact real schemas/API surfaces feeding `DATA_CONTRACTS_named_binary_probe.md` (contract JSON, resolution parquet, conflicts CSV, p0_preflight.json, `Store`, `named_binary` semantics, Rank 1A warm-up). Writes only its `--out` report.
- `inspect_oriented_price_formula.py` — read-only inspector that prints verbatim source of price/orientation callables in `pm_research/semantics/` and scans the repo for oriented/canonical price patterns. Writes only its `--out` report.
- `inspect_price_input_s0.py` — read-only S0 price-input inspector: locates price provenance (`backfill.py`), dumps `Store.load_prices/save_prices` source, walks local data/artifacts for any per-side/per-token price series, and samples non-YES/NO `yes_price` (values + per-condition multiplicity only). Explicitly refuses to infer `side_1 = 1 - yes_price`. Writes only `artifacts/named_binary_probe/price_input_s0_inspection.txt`. Produced the accepted S0 finding; no code/gate/probe changes.

### Tests (`tests/`)
- `test_forecast_vs_price.py`, `test_orderfilled_sample_join.py`, `test_ordersmatched_economic_role.py`, plus the existing store/audit/backfill suites.
- `test_named_binary_semantics.py` — named-binary suite (37 tests: classification, mapping stability, orientation, no-label-only-inversion, resolution mapping incl. Issue-A regression, leakage, gate states).
- `test_named_binary_resolution_source.py` — Stage 2 suite (24 tests: payout parsing, winner derivation, condition_id normalization, slot→token mapping, precision rejection, resolution-independence).
- `test_build_named_binary_resolution_source.py` — Stage 3 builder suite (10 tests: CSV dtype=str read, winner wiring, conflict isolation, precision fail-loud, coverage math, stable schema).
- `test_named_binary_stage4_gate.py` — Stage 4 suite (16 tests: resolution-source loader filtering/fail-loud, overlay token/index agreement, gate-policy invariant, non-YES/NO branch policy, missing-vs-ambiguous count split, probe stays blocked).
- `test_named_binary_probe_p0_preflight.py` — Stage P0 preflight suite (22 tests: each typed STOP state; corrected `source_rows = resolved + ambiguous` and `missing = eligible − source_rows` definitions incl. the resolved-only-parquet + ambiguous-in-conflicts regression; conflicts-CSV-absent gate fallback; YES_NO / ambiguous / missing / non-scoreable exclusions; like-for-like reconciliation + hard-fail; probe_execution stays False; `named_binary_probe_blocked` observed not flipped; no trades/prices loaders called (monkeypatched to fail); precision rejection; artifact emission).
- `test_price_source_s1_coverage.py` — S1 Pass 1 coverage suite (pure-logic / mocked; no network). Covers P0/contract validation, token-pair enumeration (exactly two stable string-safe tokens; NaN/missing/malformed trade-row skipping + counting; precision-loss fail-loud; never `resolved_winning_token_id`), Level-A status mapping and endpoint-shape capture (documented-shape parse, one-sample-GET labeling, deviation surfacing), Level-B decision-window classification (point-at-warmup counts, point-at/after-`resolved_at` excluded as leakage, one-side not usable), the request-window fix + request-window summary diagnostics, the invalid-decision-window guard (invalid conditions not fetched, excluded/reported, all-invalid sample → `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE` not NOT_VIABLE, mixed samples reconcile), `parse_ts` accepting the real `"…mmm UTC"` form + pandas Timestamp/datetime/epoch, `load_resolution_rows` `resolved_at` preservation + `STOP_RESOLUTION_SCHEMA` on missing column, network hard-gate (`STOP_NOT_AUTHORIZED`, no artifacts without both flags), Pass 2 unavailable, and the progress heartbeat (stderr-only, quiet toggle).

---

## Core package (`pm_research/`)

- `data/store.py` — flat parquet store; `load_trades` applies hygiene (drop null condition_id, key-preferring dedup); `load_prices`, `load_resolutions`, `load_markets`.
- `data/schemas.py` — TRADES_COLS (11; no log_index), PRICES_COLS [condition_id, ts, yes_price], RESOLUTIONS_COLS [condition_id, winning_outcome, resolved_at], MARKETS_COLS.
- `semantics/` — named-binary semantics layer: `lexicon.py` (version-pinned subclasses/rules, `NB_CONTRACT_VERSION`), `mapping_audit.py` (identity stability checks), `resolution_schema.py` (winning_outcome schema; Issue-A fix), `resolution_source.py` (Stage 2: payout-vector winner derivation + slot→token mapping + 8-status conflict model), `named_binary.py` (classification, canonical_side_price/oriented_price, audit gate), `__init__.py` (public surface).
- `splits.py` — `rolling_train_test_splits` (shared by audit/gate/probes).
- `calibration.py` — `IsotonicCalibrator` (.fit/.apply, static .brier).

---

## Specs & handoffs (repo root or specs folder)
- `SPEC_named_binary_resolution_source.md` — spec for sourcing non-YES/NO realized outcomes from Dune; IMPLEMENTED + ACCEPTED across Stages 0–4.
- `PLAN_named_binary_resolution_source_implementation.md` — staged implementation plan (Stages 0–4).
- `HANDOFF_orchestrator_named_binary_taskAB.md` — orchestrator handoff: Issue-A fix + spec, with real-data before/after.
- `FINDINGS_named_binary_resolution_blocker.md` — dated finding: YES/NO-only resolutions block the named-binary probe (superseded by the Dune source).
- **Stage 0** — `SCHEMA_INSPECTION_named_binary_resolution_source.md` (Dune schema confirmed: ctf_evt_conditionresolution payoutnumerators array(uint256) + ctf_evt_payoutredemption); script `inspect_named_binary_resolution_dune_schema.py`.
- **Stage 1** — `DUNE_SQL_named_binary_resolution_stage1.sql` (5 queries A–E, varchar-cast, no trailing ';') + `STAGE1_named_binary_resolution_query_plan.md` + `FINDINGS_stage1_named_binary_resolution_coverage.md` (coverage ~1.0; recommend proceed).
- **Stage 2** — `HANDOFF_orchestrator_stage2_resolution_source.md` (resolution_source.py + 24 tests).
- **Stage 3** — `HANDOFF_orchestrator_stage3_resolution_builder.md` (39,693 resolved / 253 ambiguous; per-condition CSV + pagination notes).
- **Stage 4** — `HANDOFF_orchestrator_stage4_gate_integration.md` (additive `--resolution-source`; legacy gate BLOCKED; non-YES/NO branch CLEAR_WITH_WARNINGS; corrected missing-vs-ambiguous breakdown; probe still blocked).
- `REVIEW_nautilus_scavenge_named_binary_resolution.md` — file-based review of the NautilusTrader bundle (engineering shapes only; LGPL — no code copied; not a canonical source).
- `SPEC_named_binary_probe.md` — ACCEPTED spec-only document for a future offline historical non-YES/NO named-binary forecast-vs-price probe. Authorizes no implementation, no data run, and no probe execution.
- `HANDOFF_orchestrator_named_binary_probe_p0.md` — Stage P0 preflight handoff (ACCEPTED, P0_CLEAR): scope, the count-definition patch (`source_rows = resolved + ambiguous`), stop conditions, JSON schema, real-data figures, and local run/pytest commands. P0 authorizes no P1/P2/P3 and no probe execution.
- `HANDOFF_orchestrator_named_binary_probe_p1_REVIEW.md` — Stage P1 review memo (PAUSED): the three real-run shape mismatches (store loaders, prices schema, contract keys) that motivated the data-contract layer. Superseded by DATA_CONTRACTS + PRICE_INPUT_CONTRACT; P1 stays blocked.
- `DATA_CONTRACTS_named_binary_probe.md` — ACCEPTED reference contract: exact inspected schemas/API surfaces for every artifact + code surface P0/P1 consume, with authoritative/diagnostic-only/absent tagging. Pins the P1 universe rule (contract∩parquet join), target derivation (token identity), warm-up (`3600 s`), and the canonical-price **formula** (`OrientationContract.canonical_side_price`, identity-select). Flags the price **input** gap.
- `PRICE_INPUT_CONTRACT_named_binary_probe.md` — ACCEPTED S0 finding: proves from source that local `yes_price` is YES/NO-only and unsafe for named-binary canonical-side pricing, that `canonical_side_price` needs two per-side prices the local schema lacks, and that no per-side/token price artifact exists locally. **P1 remains BLOCKED on price input.** Next possible step (if authorized): spec-only price-source build plan.
- `SPEC_price_source_s1_coverage.md` — **ACCEPTED / SPEC ONLY / coverage-only.** The S1 plan for testing whether Polymarket CLOB `/prices-history` per token (documented `GET /prices-history` with `market`/`startTs`/`endTs`/`interval`/`fidelity`, and `POST /batch-prices-history`) can cover the P0 universe (39,693 conditions) with a usable decision-time per-side price for both sides. Distinguishes documented-shape vs candidate-behavior vs accepted-source; locks the token-pair enumeration basis (`trades` distinct `(condition_id, token_id, outcome_index)`, validated for exactly two stable string-safe side tokens, never `resolved_winning_token_id`), the ≥95%-both-sides-per-subclass / all-three-clear / Level-B-binding coverage threshold, and the mandatory-Pass-1 → separate-go/no-go-Pass-2 gating. First stage of the per-side price-source build plan; a later S2 build spec would follow only if coverage is favorable and separately authorized. **Authorizes no implementation, no S1 run, no network/data fetch, no backfill, no scoring, no S2, no P1/P2/P3, no probe execution.** User-run only if later explicitly authorized; Claude does not fetch Polymarket data; no network-allowlist change implied. Does **not** unblock P1 by itself; `named_binary_probe_blocked` stays `true`. Forbids `yes_price` / `1 - yes_price` / `1 - p` side-synthesis.
- `HANDOFF_orchestrator_s1_pass1_RESULT.md` — **S1 Pass 1 sampled coverage RESULT (ACCEPTED): `S1_SOURCE_NOT_VIABLE`.** Records the accepted user-run: sample 300/300, 248 valid-window / 52 invalid-window, 496 fetches, clean endpoint shape, and Level-B both-sides coverage below 0.95 in every subclass (UP_DOWN 0.38, OVER_UNDER ≈ 0.5204, NAMED_OTHER 0.65). Pass 1 sampled coverage only (not Pass 2 full-universe). Consequence: CLOB `/prices-history` is not a viable per-side price source for the sampled universe; **P1 stays blocked, no `yes_price` fallback**; no Pass 2 / S2 / P1/P2/P3 / probe / scoring / backfill / gate change authorized. Also lists the sequence of data-shape fixes that produced a trustworthy healthy-window measurement (Timestamp `traded_at`, NaN skipping, request-window fix, invalid-window guard, `parse_ts` millisecond-UTC).

## Stage 4 audit gate fields (in `named_binary_audit_gate.json` when `--resolution-source` is supplied)
- `gate_state` / `base_gate_state` — legacy pooled-all (stays BLOCKED_BY_RESOLUTION_MAPPING; YES/NO sparsity).
- `gate_policy_note` — describes both the legacy gate and the non-YES/NO branch without contradiction.
- `stage4_nonyesno_branch` — `non_yesno_gate_state` (CLEAR_WITH_WARNINGS), `non_yesno_scoreable`, `non_yesno_pooled_map_rate` (0.99339), `pooled_threshold` 0.99, `subclass_threshold` 0.95, `per_subclass_map_rate`, `per_subclass_scoreable`, and `per_subclass_breakdown` (eligible / source_rows / resolved_single_winner / missing_source_rows / AMBIGUOUS_MULTIPLE_WINNERS / total_not_scoreable / exact_winner_rate / scoreable_rate).
- `non_yesno_scoreable_from_source`, `source_winner_count` (39,693), `source_conflict_counts`, `per_subclass_source_coverage`.
- `named_binary_probe_blocked` — always true; CLEAR_WITH_WARNINGS does NOT authorize a probe.

---

## Pinned context (this folder, `project_context/`)

Pin all of the following in the Claude Project Files panel (read `START_HERE.md` first):
- `START_HERE.md` — onboarding + required read order + current branch state + "old chats are not source of truth" rule.
- `PROJECT_STATE.md` — current objective / active / blocked.
- `GUARDRAILS.md` — hard constraints.
- `DECISION_LOG.md` — corrected history / settled decisions.
- `CLOSED_FINDINGS.md` — settled negative/complete results.
- `ARTIFACT_INDEX.md` (this file) — what exists and where.
- `DATA_CONTRACTS_named_binary_probe.md` — exact inspected schemas/API surfaces for the probe.
- `PRICE_INPUT_CONTRACT_named_binary_probe.md` — accepted S0 price-input finding (why P1 is blocked).
- `CLAUDE_PROJECT_SETTINGS.md` — operational Claude capability settings (network allowlist, skills); does not override the above and authorizes nothing.
- Active specs/handoffs as applicable — `SPEC_named_binary_probe.md` (spec only; post-S0 blocker addendum), `SPEC_price_source_s1_coverage.md` (accepted / spec only / coverage-only), `HANDOFF_orchestrator_named_binary_probe_p0.md` (P0 accepted), `HANDOFF_orchestrator_named_binary_probe_p1_REVIEW.md` (P1 paused/blocked).
 - `ORCHESTRATOR_LOW_CONTEXT_MODE.md` — reusable low-context review/decision protocol (minimal read policy + typed decision verbs). Documentation only; overrides nothing and authorizes nothing.
- Supporting reference (not overriding): `DUNE_DATA_NOTES.md`.

Keep these version-controlled in the repo AND pinned in the Claude Project Files panel; re-upload when changed so the two stay in sync.
