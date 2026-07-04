# DECISION LOG

*Corrected history. The most important file in the project: it records what was tried, what was wrong, and what is now settled — so settled questions are not re-litigated.*

---

## Settled decisions (with corrected history)

### Rank 1A decision policy: `first_price_after_warmup`
Chosen over fixed-lead-before-resolution because resolution time is derived from price convergence — a fixed lead would leak the outcome. Decision timestamp = first YES price >= 1h after first trade. Lookahead-safe.

### OrderFilled topic order: topic2=maker, topic3=taker (RESOLVED)
**History (3 reversals before settling):**
1. Initial decode assumed topic2=maker/topic3=taker.
2. A buggy mid-pass showed "both roles" -> reversed to topic2=taker/topic3=maker.
3. Reversal was based on an **invalid comparison** (local log_index 1227 vs Dune evt_index 1229 — different logs in a multi-fill tx).
4. **Resolved** by exact-log-index Dune match across BOTH contracts: all rows showed Dune maker == topic2. Reverted to original (topic2=maker). Matches the CTF Exchange V2 source.
**Settled:** `topic1=orderHash, topic2=maker, topic3=taker`. Verified, both contracts.

### The 217/0 and 10/0 "all-taker" results were ARTIFACTS, not findings
- **217/0** (early OrderFilled sample-join): a join-logic artifact compounded by the topic swap.
- **10/0** (first OrdersMatched run): an **asset-id precision-loss bug** — Dune web-UI CSV export floated 78-digit token IDs into scientific notation (`5.20896e+76`) and turned `0` into `0.0`, breaking asset matching so no maker could be confirmed.
**Lesson:** all-one-role outputs are a red flag; validate against a known case before interpreting. The maker-pairing validation harness caught the 10/0 before any conclusion.

### Asset-id precision fix (two parts)
- **Part A:** Dune query casts uint256 asset/amount fields to varchar.
- **Part B:** read those columns as `dtype=str`; compare via `canonical_int` (normalizes "0"/"0.0"/whitespace/full-precision integer strings); raise `DataExportPrecisionLoss` loudly on scientific notation.
After fix + Dune-API CSV path: MAKER_PAIRING_VALIDATED (4/4), expanded run 100% recoverable / 0 precision-loss.

### Fee-field diagnostic: SUPERSEDED
Inconclusive (NEED_MORE_SAMPLE; fee sparse, ~9% nonzero, weak separator). Superseded by OrdersMatched, which gives economic role directly via `takerordermaker`. Artifact preserved; line closed.

### Data hygiene (store contract)
- Missing `condition_id` rows are **persisted in raw parquet but DROPPED at analysis load** (Option A). `save_wallet_trades` keeps them; `load_trades` drops null/blank condition_id.
- Trade-id dedup **prefers rows with populated semantic keys** (tx_hash x4 + token_id x2 + outcome_index x1, stable mergesort).

### Reference cross-checks (external, corroborated)
- antflow Dune queries: OrdersMatched fires **once per trade**, OrderFilled **twice per trade**; volume = `LEAST(maker,taker amountFilled)/1e6`.
- warproxxx/poly_data: confirms `/1e6` scaling, USDC side = `assetId == 0`, and the **operator-leg filter** (exchange contract appears as taker on CTF mint/burn legs — treat as intermediary, not counterparty).
- ghost-hunter: off-chain matches can revert on-chain — a *live-execution* risk only; our on-chain event data excludes ghosts by construction.

### Named-binary semantics validated; probe BLOCKED on missing non-YES/NO outcomes (SETTLED)
- **Semantics/orientation validated.** The yes_price → canonical_side_price rewrite is complete and audit-gated. Orientation reads token identity (not display label); orientation_correctness_rate = 1.0; token_id/outcome_index coverage = 0.99601. Classification contract pinned (`nb_contract_version = nb-contract-2026-06-28.1`); the frequency-ranked label-pair census needed zero hand-rules beyond the seed lexicon.
- **Audit-reporting Issue A fixed.** `determine_schema` previously required one interpretation to clear 99% across the entire mixed eligible universe, returned `chosen_schema=None`, and reported `resolution_mapping_success_rate=0.0` — masking a perfect YES/NO result. Now schema is selected on the cleanly-mappable subset; unmappable rows stay blocked but no longer veto selection. Same all-or-nothing artifact class as 10/0: an all-zero output that was a bug hiding a real signal, caught by per-subclass validation before concluding.
- **local resolutions.parquet is YES/NO-only.** `winning_outcome` takes only `NO` (16,920) / `YES` (5,099) across 22,019 rows. No team / UP-DOWN / OVER-UNDER winner value appears.
- **non-YES/NO named-binary outcomes are unresolvable locally.** Of eligible conditions with a resolution row, winner-maps-to-a-side: YES_NO 8,521/8,521 (100%); NAMED_OTHER 0/1,814; OVER_UNDER 0/53; UP_DOWN 0/535. The non-YES/NO "with-resolution" rows resolve to `YES`/`NO`, matching neither named side (join coincidences, not usable outcomes).
- **named-binary probe remains BLOCKED** pending a validated non-YES/NO realized-outcome source (see SPEC_named_binary_resolution_source.md). Caught at the audit gate; no probe run, none authorized. A naive probe would have scored ~105k markets against a YES/NO winner and produced plausible-looking garbage.

### Named-binary non-YES/NO outcome source implemented and audit-gated — Stage 4 ACCEPTED (SETTLED)
The blocker above is now resolved as a data-availability matter. A Dune-sourced non-YES/NO realized-outcome pipeline (Stages 0–4) was built and accepted.
- **Source.** `polymarket_polygon.ctf_evt_conditionresolution` exposes `payoutnumerators` (array(uint256)) keyed by `conditionid`; it covers the non-YES/NO named-binary universe (Stage 1 coverage ~1.0). Winners derive ONLY from the payout vector (exactly-one-nonzero-slot), never from price convergence. Corroboration: `ctf_evt_payoutredemption`.
- **Build.** Stage 3 produced 39,693 RESOLVED_SINGLE_WINNER rows; 253 AMBIGUOUS_MULTIPLE_WINNERS (split/tie/refund payouts) excluded + counted; zero MALFORMED / PRECISION_LOSS / SLOT_TOKEN_MAPPING_MISSING / TOKEN_INDEX_CONFLICT / duplicate-different-payout.
- **Gate (Stage 4).** Additive `--resolution-source` flag overlays the winners. The **legacy pooled-all `gate_state` remains BLOCKED_BY_RESOLUTION_MAPPING** — correct and honest, because local `resolutions.parquet` is YES/NO-only and YES_NO maps at only ~0.13, holding the pooled rate below the 0.99 floor. A **separate non-YES/NO branch gate is `CLEAR_WITH_WARNINGS`**: non_yesno_pooled_map_rate 0.99339 (≥0.99) and each subclass ≥0.95 (UP_DOWN 0.99995, NAMED_OTHER 0.98657, OVER_UNDER 0.96535). Per-subclass counts separate `missing_source_rows` from `AMBIGUOUS_MULTIPLE_WINNERS` (NAMED_OTHER 11 missing + 216 ambiguous; OVER_UNDER 0 + 36; UP_DOWN 0 + 1) — missing rows are NOT labeled ambiguous.
- **Probe still blocked.** `named_binary_probe_blocked = true` in all states. CLEAR_WITH_WARNINGS means the outcome source/audit is usable; it does NOT authorize a probe. The named-binary probe remains blocked pending separate explicit authorization.
- **Self-correction during the build (meta).** Three errors were caught and fixed by reading real output, not tooling: a wrong NAMED_OTHER-clears-0.99 claim (it is ~0.987, clears 0.95 not 0.99); a stale gate_policy_note that contradicted the branch result; and a count-labeling bug that conflated missing-source-rows with ambiguous payouts. Consistent with the project's verify-against-authoritative-data discipline.

### S1 Pass 1 price-source coverage: `S1_SOURCE_NOT_VIABLE` (ACCEPTED, sampled) — SETTLED
The first stage of the per-side price-source build plan (`SPEC_price_source_s1_coverage.md`) was implemented as a coverage-only, network-hard-gated, user-run test and **completed with a NEGATIVE Pass-1 sampled result.**
- **Result.** On the accepted run: sample 300/300; **248 valid-window** conditions measured, **52 invalid-window** excluded + reported; **496** side-token fetches; endpoint shape parsed cleanly (`GET /prices-history`, `history` list, point keys `p`/`t`, no deviation). **Level-B both-sides coverage (the binding criterion) cleared 0.95 in no subclass:** UP_DOWN 19/50 = 0.38, OVER_UNDER 51/98 ≈ 0.5204, NAMED_OTHER 65/100 = 0.65. Verdict **`S1_SOURCE_NOT_VIABLE`**. Figures reconcile: 248 + 52 = 300; 248 × 2 = 496; per-subclass measured (50 + 98 + 100) = 248.
- **Consequence.** CLOB `/prices-history` does **not** provide a usable decision-window per-side price for both sides across the sampled P0 universe. **P1 remains BLOCKED with no `yes_price` fallback.** This is **Pass 1 sampled coverage only** — not Pass 2 full-universe coverage; the sampled negative was accepted as the finding. A later S2 build spec was gated on *favorable* S1 coverage, which did not occur, so **S2 does not follow** on this evidence. No Pass 2, S2, P1/P2/P3, probe, scoring, backfill, or gate change is authorized; `named_binary_probe_blocked` stays `true`.
- **Artifact-vs-finding discipline (meta).** The verdict was only trusted **after** the measurement was made healthy. Three earlier `S1_SOURCE_NOT_VIABLE` outputs were **artifacts, not findings**, each caught before acceptance: (1) a request-window bug that collapsed the endpoint query to a 1-second window (all `SERIES_EMPTY`); (2) an invalid-decision-window case where `resolved_at`-missing conditions were coerced into 1-second queries and mislabeled `DECISION_PRICE_NEITHER` (fixed by excluding + reporting them, and returning `S1_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE` for an all-invalid sample rather than a false NOT_VIABLE); (3) a `parse_ts` gap that rejected the real `resolved_at` string form `"YYYY-MM-DD HH:MM:SS.mmm UTC"`, blanking every window. Request-window diagnostics (`request_window_summary`, per-condition window columns) and the decision-window-validity split were added specifically so a degenerate window can never masquerade as a coverage negative. The accepted run shows wide valid windows and 248 genuinely measured conditions — consistent with the project's "never conclude from an all-one output without a validation pass" rule. An earlier attempt to record NOT_VIABLE from the stale 1-second artifact was correctly refused until the healthy-window figures were produced.

---

## DO NOT REOPEN unless explicitly asked

- Rung 1 price recalibration (closed negative).
- Fee-field diagnostic (inconclusive, superseded).
- OrderFilled topic-order debate (resolved: topic2=maker, Dune-verified).
- The 217/0 and 10/0 taker artifacts (explained: join artifact + precision-loss bug, both fixed).
- PnL-by-role / H1' (not authorized — separate phase).
- Live / paper trading (permanently out of scope).
- Named-binary semantics/orientation (validated) and Issue-A reporting fix (settled). Do not re-derive.
- Named-binary non-YES/NO outcome source + Stage 4 gate integration (ACCEPTED). Do not re-derive the source, the build, or the gate-policy split. The legacy pooled-all gate stays BLOCKED_BY_RESOLUTION_MAPPING (YES/NO sparsity); the non-YES/NO branch is CLEAR_WITH_WARNINGS. The only open named-binary work is whatever the user explicitly authorizes downstream (e.g. a spec for an offline historical probe) — and the probe itself remains unauthorized.
- S1 Pass 1 sampled coverage result (`S1_SOURCE_NOT_VIABLE`, ACCEPTED). Do not re-derive or re-litigate the sampled negative or the per-subclass rates (UP_DOWN 0.38 / OVER_UNDER ≈ 0.5204 / NAMED_OTHER 0.65 vs 0.95). It is Pass 1 sampled coverage only; a Pass 2 full-universe run (to test whether the sampled negative generalizes) or any alternative price source is a **separate, explicitly-authorized** step. P1 stays blocked with no `yes_price` fallback; the probe stays unauthorized.

---

## Self-correction discipline (meta)

This project corrected itself four times (217/0, topic swap, topic-order mismatch, asset-id precision loss). Each was caught by **validating against an authoritative source on exactly-matched data before concluding**, not by trusting tooling. Maintain this: one row or one all-one-role output is never sufficient to conclude.
