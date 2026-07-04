# SPEC — Named-Binary Offline Historical Forecast-vs-Price Probe

**Status:** SPEC ONLY. Authorizes and builds nothing. No code, no audit changes, no data runs, no probe execution.
**Task chat:** Chat1C.
**Consumes (does not redefine):** named-binary classification contract (`nb-contract-2026-06-28.1`) and the Stage 3/4 non-YES/NO realized-outcome source.
**Reconciles with:** GUARDRAILS.md, PROJECT_STATE.md, DECISION_LOG.md, CLOSED_FINDINGS.md, ARTIFACT_INDEX.md, DUNE_DATA_NOTES.md.

> This spec describes a probe that is **NOT AUTHORIZED to run**. `named_binary_probe_blocked = true` is a **standing non-authorization marker** and remains in force; a future authorized run does **not** require flipping that artifact field to false. Two independent conditions are separated throughout this spec: (a) a **data-quality gate** — Stage 4 `non_yesno_gate_state = CLEAR_WITH_WARNINGS` plus per-subclass scoreability — which the CLEAR_WITH_WARNINGS branch already satisfies and which only makes the outcome source *usable as a scoreable source*; and (b) **user execution authorization** — a future, explicit, in-chat authorization from the user, represented in any later implementation by a deliberate CLI flag or authorization manifest. Both must hold to run. Accepting this spec is **not** authorization; absent (b), the probe stops with `STOP_NOT_AUTHORIZED`.

> **POST-S0 PRICE-INPUT BLOCKER ADDENDUM:** This spec remains SPEC ONLY and does not authorize implementation or execution. Since this spec was accepted, S0 price-input inspection proved that the local price input assumed by the original P1 path is not usable for non-YES/NO named binaries. `Store.load_prices()` exposes only `[condition_id, ts, yes_price]`; `yes_price` is YES/NO-only and unsafe for UP_DOWN / OVER_UNDER / NAMED_OTHER; no local per-side/per-token price artifact exists. Therefore the P1 path described below cannot be implemented from current local artifacts. P1 remains BLOCKED until a valid per-side / token-identity price source exists and is audit-checked. The next possible work, only if separately authorized, is a spec-only price-source build plan. Do not synthesize `side_0_price = yes_price` or `side_1_price = 1 - yes_price`. See `PRICE_INPUT_CONTRACT_named_binary_probe.md` and `DATA_CONTRACTS_named_binary_probe.md` (§6, [RESOLVED-B′]).

---

## 1. Objective and hypothesis

### 1.1 Objective
Define — without running — an offline, fully historical, ex-ante forecast-vs-price probe restricted to the **non-YES/NO named-binary universe** (UP_DOWN, OVER_UNDER, NAMED_OTHER). The probe asks one falsifiable question: does the Polymarket canonical-side price, or its train-only isotonic recalibration, carry any out-of-sample forecast skill against realized payout-vector outcomes, after honest time-based splitting?

This is the non-YES/NO analogue of the Rank 1A YES/NO price-recalibration probe — which closed **NEGATIVE** (CLOSED_FINDINGS: well-calibrated YES price, no fee-surviving edge, no actionable price bucket). The pessimistic prior is therefore: **non-YES/NO prices are also well-calibrated and no recalibration edge exists.** The probe is designed to *falsify a claimed edge cleanly*, not to manufacture one.

### 1.2 Primary hypothesis (the thing to try to kill)
- **H0 (null, expected to hold):** The canonical-side market price is calibrated on non-YES/NO named-binary conditions; no monotone recalibration of price produces out-of-sample Brier/log-loss improvement over the raw price baseline across time splits.
- **H1 (alternative, to be falsified):** Some specified, fit-on-train transform of canonical_side_price improves out-of-sample Brier vs. the raw-price baseline, consistently across splits and per-subclass.

### 1.3 What a result means
- This probe measures **forecast quality only**. A positive Brier delta is **not** a tradability claim, an edge claim, or a strategy. Cost/edge discipline (§8) and the non-authorization list (§12) govern any downstream interpretation.
- Consistent with project discipline: no conclusion from a single split, a single subclass, or an all-one-direction output. A negative or null result is a perfectly acceptable (and expected) outcome and should be reported as such.

---

## 2. Eligible universe

A condition is **probe-eligible** iff **all** hold:

1. **Non-YES/NO named-binary**, per the accepted classification contract `named_binary_classification_contract.json`, subclass ∈ {UP_DOWN, OVER_UNDER, NAMED_OTHER}. YES/NO (`YES_NO`) is **excluded** — it is the Rank 1A universe, closed negative, and its local resolution path is separate.
2. **Contract version matches** the pinned `nb_contract_version` exactly (`nb-contract-2026-06-28.1`). Version mismatch → hard stop (§9).
3. **Resolved by the Stage 3/4 resolution source** — the condition has a `RESOLVED_SINGLE_WINNER` row in `named_binary_resolution_source_rows.parquet` (exactly-one-nonzero payout slot).
4. **Not ambiguous, not missing.** Exclude `AMBIGUOUS_MULTIPLE_WINNERS` (split/tie/refund) and missing-source rows. These are excluded **and counted separately** (never silently dropped, never relabeled — per the Stage 4 missing-vs-ambiguous lesson in DECISION_LOG).
5. **Subclass is branch-scoreable.** Only subclasses whose Stage 4 `per_subclass_scoreable = true` (each ≥ 0.95 exact-winner rate) are eligible. On current data all three clear; the probe must re-read the gate rather than assume.

**Eligibility is computed from artifacts, never re-derived.** The probe does not classify markets, does not invent a named-binary definition, and does not compute its own winners. It reads the contract and the resolution-source rows.

---

## 3. Data inputs (all ex-ante)

| Input | Source artifact | Use | Leakage constraint |
|---|---|---|---|
| Classification + eligibility + subclass | `artifacts/named_binary/named_binary_classification_contract.json` | Define eligible universe; assert `nb_contract_version` | Static; no outcome content |
| Realized winners | `artifacts/named_binary/named_binary_resolution_source_rows.parquet` | Outcome labels (winning token/outcome_index) | **Used only as the scoring target**, never as a feature; never joined into the decision-time feature row |
| Branch gate state | `artifacts/named_binary/named_binary_audit_gate.json` (with `stage4_nonyesno_branch`) | Confirm `non_yesno_gate_state = CLEAR_WITH_WARNINGS`, per-subclass scoreable, `named_binary_probe_blocked` | Read-only precondition check |
| Prices | local `prices` (`condition_id, ts, yes_price`) via `load_prices` | Build canonical_side_price at the decision timestamp | **Strictly ex-ante**: only price rows at/just after warm-up; no price at/near resolution — **⚠ OBSOLETE / BLOCKED for non-YES/NO P1 (post-S0):** `load_prices()` returns only a single `yes_price` scalar, which is YES/NO-only and unsafe for UP_DOWN/OVER_UNDER/NAMED_OTHER; `canonical_side_price` needs two per-side prices and no local per-side/token price series exists. This row records the *originally assumed* input and must **not** be implemented from as-is. See `PRICE_INPUT_CONTRACT_named_binary_probe.md`. Do not synthesize `side_0_price = yes_price` / `side_1_price = 1 - yes_price`. |
| Trades | local `trades` via `load_trades` (hygiene-applied) | Establish first-trade time / warm-up anchor for the decision timestamp; activity filters | Ex-ante only; no post-decision trades enter features |

**Orientation:** canonical_side_price / oriented_price come from the validated semantics layer (`pm_research/semantics/named_binary.py`), which reads token identity, not display label (DECISION_LOG: orientation_correctness_rate = 1.0). The probe consumes this; it does not re-implement orientation.

**No wallet inputs. No OrdersMatched. No log_index. No Dune live queries at probe time** (resolution rows are already materialized as a versioned artifact).

---

## 4. Outcome semantics

- **Winners come ONLY from payout numerators** (exactly-one-nonzero slot), via the Stage 3/4 source. **Price convergence is never an outcome source** — forbidden as a definition (DECISION_LOG; DUNE_DATA_NOTES §11). Using the price to define the label would both leak and answer the wrong question.
- The binary target per eligible condition: `y = 1` if the canonical/oriented side is the realized winning side, else `y = 0`, where "winning side" maps via the resolution row's `resolved_winning_token_id` / `resolved_outcome_index` through the contract's slot→token mapping.
- Ambiguous/missing → excluded + counted (§2), never imputed.
- Token-id comparisons use the project's `canonical_int` discipline (string-safe, fail loud on scientific notation). Any precision-loss signature → hard stop (§9).

---

## 5. Forecast candidates

**V1 candidate set (frozen, exactly two):**

1. **C0 — Raw baseline (price-is-the-forecast).** `p_hat = canonical_side_price(decision_ts)`. This is the null baseline; everything is measured **against C0**, mirroring Rank 1A Rung 0.
2. **C1 — Monotone recalibration (fit-on-train).** Isotonic recalibration of C0 fit only on the training split (reuse `pm_research/calibration.py::IsotonicCalibrator`). This is the direct non-YES/NO analogue of Rank 1A Rung 1 (which was positive in only 1/5 splits → negative overall).

V1 ends here. **Any additional transforms (e.g. logit shrink toward 0.5, tail clipping) are out of scope for v1** and belong to a future, separately-reviewed spec addendum — not to this probe. Keeping v1 to {C0, C1} avoids multiple-comparisons fishing and matches the Rank 1A two-rung structure exactly.

**Explicitly out of scope for this spec:** any transform beyond C0/C1; wallet/flow features, OrdersMatched-derived features, maker/taker typing, any feature that requires Chat2 cohorts. The probe is price-only by design.

---

## 6. Splits

### 6.1 Strict time-based train/test
- **Rolling time-based train/test** via the shared `pm_research/splits.py::rolling_train_test_splits`. Conditions are ordered by their **decision timestamp** (an ex-ante anchor), not by resolution time.
- **No full-period fitting.** Every fitted object (isotonic map, transform params) is fit on the train window only and applied forward to the disjoint test window.
- A minimum of (recommended) 5 rolling splits, matching the Rank 1A harness, so per-split consistency can be judged (Rank 1A's "positive in 1/5" pattern is exactly the kind of signal this guards against).

### 6.2 Decision-timestamp definition (lookahead-safe)
- Reuse the Rank 1A policy class: **decision timestamp = first canonical-side price at ≥ warm-up after first trade** (`first_price_after_warmup`). This was chosen specifically because resolution time is convergence-derived and a fixed lead-before-resolution would leak the outcome (DECISION_LOG, Rank 1A decision policy).
- The probe must **not** use any fixed-lead-before-resolution timestamp, and must not read resolution time into feature construction at all.

### 6.3 Leakage rules
- Train window strictly precedes test window in time; no overlap.
- No resolution-time, no final/near-final price, no future outcome touches any feature or any fitted parameter.
- Recalibration/transform parameters never see test rows.
- Subclass-conditional fitting (if used) still obeys the same train/test boundary.

---

## 7. Metrics

Reported **per-subclass** (UP_DOWN, OVER_UNDER, NAMED_OTHER) and pooled, **per-split** and aggregated:

- **Brier score** (primary), and **Brier skill vs C0** baseline (the decision metric, as in Rank 1A).
- **Log loss**, reported only if numerically stable (guard against 0/1 saturated probabilities; apply a pre-registered epsilon clip used identically across candidates, or mark log loss `UNSTABLE` and fall back to Brier).
- **Calibration curve / reliability** (binned), per subclass — diagnostic, not a pass/fail gate by itself.
- **Per-subclass minimum sample counts** (§7.1) reported alongside every metric; any subclass below threshold is reported as `INSUFFICIENT_SAMPLE`, not scored as a finding.

### 7.1 Minimum sample counts (pre-registered)
- Per subclass per test split: a **floor of N_min eligible resolved conditions** below which that subclass-split is `INSUFFICIENT_SAMPLE` and excluded from any consistency claim. Proposed default `N_min = 200` test conditions per subclass-split. **Exact per-subclass and per-split resolved counts must be recomputed in Stage P0 from the Stage 4 artifacts (`named_binary_resolution_source_rows.parquet` + `named_binary_audit_gate.json`) before any scoring** — this spec makes no claim about current subclass sizes. Any subclass whose recomputed count falls below `N_min` is reported as `INSUFFICIENT_SAMPLE` and not force-scored.
- A finding ("edge" or "no edge") requires the metric to hold **across splits** and clear the sample floor; a single split is never sufficient (project discipline).

---

## 8. Cost/edge discipline

- **Forecast-quality metrics are separated from tradability.** Brier/log-loss/reliability describe forecast calibration only.
- **No strategy claim from Brier alone.** A positive Brier skill is not an edge; Rank 1A showed a calibrated price leaves no fee-surviving edge even where recalibration nominally helped a minority of splits.
- **No execution simulation, no fee model, no PnL, no sizing** in this probe. If forecast skill were ever found, an execution/cost study would be a **separate, separately-authorized phase** (and would still honor: no live trading, no paper trading, no wallet-copying).
- The report must state plainly that any positive forecast result is hypothesis-generating only and carries no tradability implication without a later, separately-approved cost study.

---

## 9. Stop conditions (hard halts — fail loud, conclude nothing)

The probe (when later built and run) must abort and emit a typed status rather than produce numbers if any hold:

1. **Missing outcome source** — `named_binary_resolution_source_rows.parquet` absent or empty → `STOP_MISSING_OUTCOME_SOURCE`.
2. **Stale contract version** — contract `nb_contract_version ≠ nb-contract-2026-06-28.1`, or resolution-source rows carry a different `nb_contract_version` than the contract → `STOP_STALE_CONTRACT`. (Chat-coupling discipline: consume one pinned version, assert equality.)
3. **Precision loss** — any scientific-notation / float-mangled token-id or payout signature on read → `STOP_PRECISION_LOSS` (`DATA_EXPORT_PRECISION_LOSS`), never reconstruct.
4. **Subclass too sparse** — eligible resolved test count below `N_min` for a subclass → that subclass marked `INSUFFICIENT_SAMPLE`; if all subclasses fail, `STOP_ALL_SUBCLASSES_SPARSE`.
5. **Leakage risk detected** — any decision timestamp ≥ resolution time, any feature row sourced after decision ts, any fitted parameter exposed to test rows, or any all-one-direction output without a passing validation case → `STOP_LEAKAGE_GUARD`. (All-one-direction outputs are a known red-flag class — 217/0, 10/0 — and must trigger validation, not interpretation.)
6. **Data gate regressed** — if `named_binary_audit_gate.json` does not show `non_yesno_gate_state = CLEAR_WITH_WARNINGS` with the required subclass scoreability → `STOP_DATA_GATE_NOT_CLEAR`. This is a data-quality halt, independent of authorization.
7. **Execution not authorized** — if the run was launched without the explicit user execution authorization (deliberate CLI flag or authorization manifest, per the header banner) → `STOP_NOT_AUTHORIZED`. The standing `named_binary_probe_blocked = true` marker does **not** need to be flipped for an authorized run; authorization is carried by the flag/manifest, not by editing that field.

---

## 10. Proposed artifacts and tests

### 10.1 Proposed artifacts (under `artifacts/named_binary_probe/`)
- `named_binary_probe.json` — machine-readable results: per-subclass × per-split Brier, Brier skill vs C0, log loss (or `UNSTABLE`), sample counts, statuses, `nb_contract_version`, gate snapshot, candidate list, decision-timestamp policy.
- `named_binary_probe.md` — narrative report: hypothesis, universe counts (eligible / resolved / excluded-ambiguous / excluded-missing), per-subclass results, explicit "edge / no-edge / insufficient" verdict per subclass, and the cost-discipline caveat (§8).
- `named_binary_probe_splits.csv` — one row per candidate × subclass × split (Brier, skill, n, status).
- `named_binary_probe_reliability.csv` — binned reliability points per subclass × candidate.
- `named_binary_probe_excluded.csv` — excluded conditions with reason (`AMBIGUOUS_MULTIPLE_WINNERS`, `MISSING_SOURCE`, `STALE_CONTRACT`, `INSUFFICIENT_SAMPLE`), counts reconciling to totals.

### 10.2 Proposed tests (`tests/test_named_binary_probe.py`) — written and passing *before* any real-data run, in the later phase
- **Eligibility:** YES_NO excluded; only RESOLVED_SINGLE_WINNER included; ambiguous/missing excluded + counted; non-scoreable subclass excluded.
- **Contract pin:** stale `nb_contract_version` → `STOP_STALE_CONTRACT`.
- **Outcome independence:** label derives only from payout-vector winner; a synthetic price-convergence "winner" is never used.
- **Leakage:** synthetic case where a near-resolution price would flip a result confirms it is excluded; decision ts ≥ resolution ts → `STOP_LEAKAGE_GUARD`.
- **Split integrity:** no train/test overlap; fitted isotonic map never sees test rows (parameter-identity check).
- **Precision:** scientific-notation token id → `STOP_PRECISION_LOSS`.
- **Sample floor:** subclass below `N_min` → `INSUFFICIENT_SAMPLE`, not scored.
- **Baseline math:** Brier skill vs C0 reduces to 0 when candidate == baseline.
- **All-one-direction guard:** an all-one-direction output triggers the validation path, not a conclusion.
- **Authorization vs data gate:** data-gate-not-clear → `STOP_DATA_GATE_NOT_CLEAR`; missing execution flag/manifest → `STOP_NOT_AUTHORIZED`; an authorized run does not require `named_binary_probe_blocked` to be flipped to false.

---

## 11. Exact implementation plan for a later phase (DO NOT IMPLEMENT NOW)

Staged, mirroring the project's staged discipline. Each stage closes with a Claude-to-Orchestrator handoff; nothing runs until separate explicit execution authorization.

- **Stage P0 — Preflight (read-only).** Load contract + gate + resolution-source rows; assert version equality and `non_yesno_gate_state = CLEAR_WITH_WARNINGS`; emit eligible/excluded counts only. No scoring. Verifies the universe before any metric exists.
- **Stage P1 — Decision-timestamp + feature assembly.** Build the ex-ante decision timestamp (`first_price_after_warmup`) and canonical_side_price per eligible condition; assemble C0/C1 inputs. Leakage guards active. No outcomes joined yet beyond the held-aside target. **⚠ CURRENTLY BLOCKED (post-S0):** P1 cannot proceed and cannot be implemented from current local artifacts, because the decision-time canonical-side price is not derivable — `load_prices()` supplies only a YES/NO-only `yes_price` scalar (unsafe for non-YES/NO), and no local per-side/token-identity price series exists (`PRICE_INPUT_CONTRACT_named_binary_probe.md`). P1 stays blocked until a per-side / token-identity price-source artifact is **specified, built, and audit-checked**. The next possible work, only if separately authorized, is a **spec-only price-source build plan** (no implementation, no data run/backfill, no scoring, no probe).
- **Stage P2 — Split + score harness.** Wire `rolling_train_test_splits`; fit C1 on train only; compute Brier / skill / log-loss / reliability per subclass × split; enforce sample floors and stop conditions. Tests from §10.2 pass first.
- **Stage P3 — Report + verdict.** Emit artifacts (§10.1); write the per-subclass edge/no-edge/insufficient verdict with the cost caveat. Pessimistic framing: default expectation is no out-of-sample edge (Rank 1A precedent).
- **Stage P4 (separate authorization only) — cost/tradability study.** Out of scope here; named solely to show where it would live. Not part of this probe.

Each stage is **gated**: P(n+1) does not start until P(n)'s handoff is reviewed.

---

## 12. Explicit non-authorization list

This spec authorizes **none** of the following. All remain prohibited:

- ❌ Running the probe / any data run (`named_binary_probe_blocked = true` stands).
- ❌ Any code, audit change, or gate change as part of *this* deliverable.
- ❌ Chat2 Dune wallet-cohort discovery (still BLOCKED; outcome-source scoreability does not unblock it).
- ❌ PnL-by-role / H1' (separate, unauthorized phase).
- ❌ Wallet-copying / copy-trade / follower-aggressor logic (permanently out of scope).
- ❌ Live trading. ❌ Paper trading.
- ❌ `log_index` backfill. ❌ Full indexer.
- ❌ Named-binary YES/NO recalibration reopen (Rank 1A closed negative — do not re-litigate).
- ❌ Reopening settled items (DECISION_LOG §Do Not Reopen): OrderFilled topic order, 217/0 & 10/0 artifacts, fee diagnostic, named-binary semantics/orientation, the resolution-source build + gate split.
- ❌ Any strategy/edge/tradability claim from Brier alone.
- ❌ Defining named-binary independently of `nb-contract-2026-06-28.1`.

**Execution requires a future, separate, explicit in-chat authorization** that names the probe run specifically. Accepting this spec is not that authorization.

---

## Risks / leakage checks (summary)

- **Outcome leakage via price** — mitigated: winners are payout-vector only; price never defines the label (§4).
- **Lookahead via decision timestamp** — mitigated: `first_price_after_warmup`, never a lead-before-resolution (§6.2); resolution time never enters features.
- **Train→test contamination** — mitigated: fit-on-train only, disjoint rolling windows, parameter-identity tests (§6.3, §10.2).
- **Multiple-comparisons fishing** — mitigated: frozen v1 candidate set {C0, C1} only, per-subclass + per-split consistency required, no free-form search, extra transforms deferred to a separately-reviewed addendum (§5, §7).
- **Sparse-subclass false signal** — mitigated: pre-registered `N_min`, exact per-subclass/per-split counts recomputed in P0 from Stage 4 artifacts before scoring, `INSUFFICIENT_SAMPLE` status for any subclass below floor (§7.1, §9).
- **Precision loss** — mitigated: string-safe token ids, fail-loud on scientific notation (§4, §9).
- **All-one-direction artifact** — mitigated: known red-flag class triggers validation, not conclusion (§9.5).
- **Stale-contract drift** — mitigated: version equality assertion, hard stop (§9.2).
- **Misreading CLEAR_WITH_WARNINGS as authorization** — mitigated: explicit non-authorization (§12), data-gate halt `STOP_DATA_GATE_NOT_CLEAR` and authorization halt `STOP_NOT_AUTHORIZED` kept as separate stop conditions (§9.6, §9.7).
