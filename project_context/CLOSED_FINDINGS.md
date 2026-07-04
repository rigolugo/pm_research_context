# CLOSED FINDINGS

*Settled results. Each is closed; do not reopen without explicit authorization (see DECISION_LOG §Do Not Reopen).*

---

## Phase 1 — Delayed wallet-copying: NO-GO (structural)

Latency-bound taker-style copying of monitored wallets is not viable. The structural conclusion holds.

*Caveat:* specific ROI/Sharpe numbers were computed on ~2–3% of data (pre-rebackfill). A full **quantitative revalidation** on the 16.5M-trade dataset is **optional / deferred** — required only before repeating any numeric Phase-1 claim.

---

## Rank 1A — Pooled YES/NO price recalibration: NEGATIVE

Isotonic recalibration of the market YES price gave no out-of-sample, fee-surviving edge.
- Rung 0 sanity tied the price baseline (harness valid).
- Rung 1 (isotonic recalibration, fit-on-train) positive in only 1/5 splits.
- Per-split Brier skill clustered slightly negative.
- **Conclusion:** the Polymarket YES price is well-calibrated; recalibration carries no edge.

## Rank 1A price-bucket diagnostic: NO_ACTIONABLE_BUCKET

Segmenting by decision price found no region (including longshot tails) with persistent Brier+EV edge across splits. Market efficiency is **uniform across the probability range**. The price-calibration forecast line is closed negative.

---

## Rank 2 — Economic role recovery: COMPLETE

**OrdersMatched is the correct economic-role instrument.**
- `takerordermaker` = economic taker (aggressor), identified directly.
- Maker side confirmed only by positive OrderFilled-leg pairing (never by exclusion).
- Topic order verified against exact-index Dune rows: `topic1=orderHash, topic2=maker, topic3=taker`.
- Maker-pairing logic validated against known-maker cases (MAKER_PAIRING_VALIDATED, 4/4).

**Expanded result (150 stratified tx):**
- 117 OrdersMatched trades classified, 96 wallets covered, both old contracts, 4 time buckets.
- 71 ECONOMIC_TAKER / 46 ECONOMIC_MAKER (taker_share 0.607 / maker_share 0.393).
- **100% recoverable, 0% ambiguity, 0 precision-loss.**
- **log_index NOT needed for role recovery** (multi-fill classified as cleanly as single-fill).
- **Bimodal wallet-level pattern:** most wallets are pure-taker or pure-maker at shallow depth; the pooled 61/39 masks two distinct wallet populations. Per-wallet depth (mostly n=1) is too thin for individual wallet-type labels.

**Original H1 ("profitable wallets are makers / liquidity providers"): NOT SUPPORTED** as a population claim. Wallets are mixed/taker-leaning, not liquidity-provider-dominated.

**Reframed hypothesis H1' (NOT tested, NOT authorized):** do maker-type and taker-type wallet populations differ in realized PnL? This is the only meaningful Rank 2 continuation and is a separate, larger, explicitly-gated phase (would likely require log_index for clean per-trade PnL attribution).

---

## Net thesis status

Three independent angles now unsupported: Phase 1 (taker-copying), Rank 1A (price-forecasting), Rank 2 (maker-edge / liquidity-provider). The wallet-edge thesis is substantially weakened. Effort redirected to the named-binary universe (see PROJECT_STATE).
