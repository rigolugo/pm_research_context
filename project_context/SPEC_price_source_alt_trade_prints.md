# SPEC — S1-ALT Alternative Price-Source Coverage Test: Trade-Print Reconstruction

**Status:** SPEC ONLY — **APPROVED at orchestrator review (SPEC ONLY)**, with one required correction now applied (precision-loss scope; see §2 / §6.4 / §8.5). Approval covers the spec text only; it authorizes nothing further — no implementation, no tests, no data run, no network fetch, no coverage run, no Pass 1 execution, no Pass 2, no S2-analogue build, no P1/P2/P3, no probe execution, no scoring, no backfill, no gate change. `named_binary_probe_blocked` stays `true` and is not flipped.

> **Doc-only patch (this revision).** Corrects the precision-loss handling so that it applies to **identifiers only** (`token_id`, `outcome_index`, other integer-like identifiers requiring digit-exact identity) and **not** to `price`. For `price` the sole requirement is a **finite numeric value in `[0,1]`**; scientific-notation formatting of a numeric price is not, by itself, precision loss or an error. No other guardrail changes.
**Task chat:** Alternative price-source plan after S1 Pass 1 negative.
**Predecessor finding (SETTLED — not re-litigated here):** S1 Pass 1 sampled coverage completed and was accepted as `S1_SOURCE_NOT_VIABLE`. CLOB `/prices-history` cleared the 0.95 Level-B both-sides bar in **no** subclass (UP_DOWN 19/50 = 0.38; OVER_UNDER 51/98 ≈ 0.5204; NAMED_OTHER 65/100 = 0.65). S2 (a build spec gated on *favorable* S1 coverage) does not follow. P1 remains BLOCKED with no `yes_price` fallback.
**Scope:** A coverage-only feasibility plan for the next candidate per-side / token-identity price source: **per-token trade prints**, primarily the already-local `trades` parquet exposed by `Store.load_trades()`. Like S1, this stage tests **whether the candidate source can cover the P0 universe with a usable decision-window per-side price for both sides** — it does not build a price artifact and computes no canonical-side price.
**Consumes (does not redefine):** the accepted named-binary classification contract (`nb-contract-2026-06-28.1`), the P0 eligible universe (`final_p0_eligible = 39,693`), the resolution-source parquet (winners from payout vectors only), the accepted `first_price_after_warmup` decision policy (warmup 3600 s), and the S1 taxonomy (Level A/B/C, typed statuses, 0.95 per-subclass Level-B threshold, all-one-status guard).
**Reconciles with:** GUARDRAILS.md, PROJECT_STATE.md, DECISION_LOG.md, PRICE_INPUT_CONTRACT_named_binary_probe.md, SPEC_price_source_s1_coverage.md, HANDOFF_orchestrator_s1_pass1_RESULT.md.

---

## Epistemic tiers (carried over from S1)

- **[PROVEN-LOCAL]** — established from source or data already inspected and accepted (S0 price-input contract; trades schema as quoted in the accepted S0 report).
- **[DOCUMENTED SHAPE]** — supported by official Polymarket documentation; still not behavior.
- **[CANDIDATE BEHAVIOR]** — must be measured by the (later, separately authorized) S1-ALT run. Coverage, density, and one-side-only rates are empirical.
- **[ACCEPTED PRICE SOURCE]** — reached only after a later, separately authorized build + audit gate. Nothing in S1-ALT confers it.

---

## 0. Why this candidate exists (and why it might succeed where `/prices-history` failed)

S0 proved **[PROVEN-LOCAL]** that `OrientationContract.canonical_side_price(side_0_price, side_1_price)` needs two per-side prices in `outcome_index` order, that local `Store.load_prices()` supplies only the YES/NO-only `yes_price` scalar, and that **no local per-side price artifact exists** — but S0 also recorded that the **`trades` parquet does carry `token_id` and `outcome_index` per print** (`trade_id, wallet, condition_id, outcome, side, price, size_usdc, traded_at, tx_hash, token_id, outcome_index`). S0 correctly noted that raw prints "are trades, not a price series" — the gap is that no *series artifact* has been derived from them, not that per-token price observations are absent locally.

S1 established **[SETTLED]** that the CLOB `/prices-history` *server-side series* is systemically sparse/absent in the decision window for old resolved markets. Trade prints are a **different observation channel**: they are the executions themselves, already backfilled locally for 16.5 M trades / ~288.7 k conditions, and the decision timestamp is anchored at `first_trade_ts + 3600 s` — i.e., one hour after trading demonstrably began in this very table. Whether prints exist **in-window for BOTH side tokens** is unknown and is exactly what S1-ALT measures. The S1 negative does not predict the S1-ALT answer in either direction; a second clean negative is an acceptable, expected-possible outcome (falsify cheaply).

**S1-ALT answers one question:** *For the 39,693 P0-eligible non-YES/NO conditions, do per-token trade prints provide, for BOTH side tokens, at least one usable price observation inside the leakage-safe decision window `[first_trade_ts + 3600 s, resolved_at)` — and where do they fail?*

---

## 1. Candidate source options (§task item 1)

### Option A — Local `Store.load_trades()` prints (PRIMARY candidate)
- **What it is [PROVEN-LOCAL]:** the existing local trades parquet, loaded with the accepted store hygiene (null/blank `condition_id` dropped at load; trade-id dedup prefers rows with populated semantic keys).
- **Why primary:** no network, no allowlist question, no rate policy, fully reproducible on the Windows box, and it is the same table already locked as the S1 token-pair enumeration basis (`trades` distinct `(condition_id, token_id, outcome_index)`), so token identity and price observations come from one consistent source.
- **Known limits to measure [CANDIDATE BEHAVIOR]:** key coverage (`tx_hash`/`token_id`/`outcome_index`) is ~99.6%, not 100% — prints with missing `token_id`/`outcome_index` cannot be assigned to a side and must be excluded per-row (counted); print density inside the decision window per token is unknown; one-side-only trading is plausible.

### Option B — Polymarket Data API `/trades` (SECONDARY, gap-fill only, network, user-run)
- **What it is:** Polymarket's public Data API exposes a trades endpoint returning historical trade records per market/asset. Treat the exact route, parameters, and field names as **[DOCUMENTED SHAPE — must be pinned verbatim from official docs before any run]**; this spec does not assert them from memory. Behavior on old resolved tokens is **[CANDIDATE BEHAVIOR]**.
- **Role:** strictly a **gap-fill / cross-check** candidate, considered only if Option A's coverage is close-but-short and the orchestrator separately authorizes a network pass. It is user-run on the Windows box (Claude does not fetch Polymarket data; no allowlist change implied), polite-rate, resumable — same run-ownership terms as S1.
- **Mandatory equivalence check:** on a spread validation sample, API-returned prints must be reconciled against local prints for the same `(condition_id, token_id)` (same trades up to dedup rules, string-safe ids, consistent timestamps). Divergence is a finding (`API_LOCAL_MISMATCH`, §6.5), and local remains authoritative for anything both sources cover.

### Option C — Other Polymarket-native channels (LISTED, NOT candidates without separate authorization)
- On-chain `OrderFilled` / `OrdersMatched` event expansion could in principle reconstruct per-token prints from logs, but it sits inside guarded territory (`log_index` backfill, OrdersMatched expansion, full-indexer risk) and is **excluded from S1-ALT by guardrail**. Recorded here only so the option space is honest; invoking it would require its own explicit authorization and spec.
- CLOB authenticated trade endpoints (own-orders scope) are structurally unsuitable for a historical universe and are not candidates.

**Locked ordering:** S1-ALT Pass 1 runs on **Option A only**. Option B enters only via a separate explicit authorization after the Pass 1 handoff. Option C is out of scope entirely.

---

## 2. Required data contract (§task item 2)

Every print consumed by S1-ALT must supply, with typed handling for absence:

| Field | Requirement | Handling if missing/invalid |
|---|---|---|
| `condition_id` | non-null, non-blank (store already drops at load) | row never enters (store contract) |
| `token_id` | string-safe 78-digit integer string; `canonical_int` normalization; **scientific notation anywhere → fail loud** | row → `PRINT_KEY_MISSING` (counted per condition); systemic → `STOP_PRECISION_LOSS` |
| `outcome_index` | ∈ {0, 1}; consistent with the condition's validated token pair | row → `PRINT_KEY_MISSING`; token/index conflict → condition `TOKEN_PAIR_UNRESOLVED` |
| trade timestamp (`traded_at`) | parseable to ms-precision UTC (reuse the accepted `parse_ts` handling incl. the real `"YYYY-MM-DD HH:MM:SS.mmm UTC"` form and pandas-Timestamp values) | row → `PRINT_TS_UNPARSEABLE` (counted); never coerced |
| `price` | **finite numeric value in `[0,1]`** — this is the only price requirement; scientific-notation *formatting* of an ordinary numeric price is **not** an error and is **not** precision loss (a price is a small float, not an identity, so no digit-exact integer representation is at stake). Recorded as a **source-defined per-token print price** (an execution price for the queried token, not an asserted canonical probability — semantic labeling deferred to the build stage, exactly as S1 deferred the meaning of `p`) | non-finite (NaN/inf) or outside `[0,1]` → row `PRINT_PRICE_INVALID` (counted); scientific-notation formatting alone is accepted, not excluded |
| `side` | optional; **diagnostic only** (e.g., BUY/SELL mix near the decision timestamp as a price-quality signal). `side` is **never** used to flip, complement, or re-orient a price and never substitutes for token identity | absence is fine; no row excluded for missing `side` |

Token-pair enumeration reuses the S1 locked basis unchanged: `trades` distinct `(condition_id, token_id, outcome_index)`, validated to exactly two stable, string-safe side tokens per condition **before** any measurement; `resolved_winning_token_id` remains **forbidden** as a pair source (outcome-conditioned). Missing/unstable/`!=2` → `TOKEN_PAIR_UNRESOLVED` (excluded + counted); a large unresolved fraction → `STOP_TOKEN_ENUMERATION_UNRELIABLE`.

---

## 3. Deriving a decision-time per-side price with no synthesis (§task item 3)

The rule is per-token isolation. Each side's price observation comes **only from prints of that side's own token**, selected by token identity in `outcome_index` order:

1. `first_trade_ts = min(traded_at)` over the condition's local prints (ex-ante anchor; accepted policy).
2. Decision window `W = [first_trade_ts + 3600 s, resolved_at)` — identical to S1; strict `< resolved_at`.
3. For side *k* (token `side_k_token`): the candidate decision-time observation is the **first print of `side_k_token` with `traded_at ∈ W`** (`first_price_after_warmup`, applied per token). Its price is `side_k_price_observation`; the gap `traded_at − (first_trade_ts + 3600 s)` is recorded as a density diagnostic.
4. Condition classification (S1 taxonomy reused verbatim): `DECISION_PRICE_BOTH_SIDES` / `DECISION_PRICE_ONE_SIDE` / `DECISION_PRICE_NEITHER` / `NO_TRADE_ANCHOR`, plus the S1 invalid-window exclusion `NO_VALID_DECISION_WINDOW_AFTER_WARMUP` (`resolved_at` missing or `<= first_trade_ts + warmup`) — excluded + reported, never a coverage negative.

**Explicitly forbidden, by construction and by test (§8):**
- `yes_price`, `1 − price`, `1 − yes_price` in any role;
- deriving one side's price from the other side's print (`side_1 = 1 − side_0_print` or any complement/synthesis) — even where coincident prints look complementary, complementarity stays a Level-C **diagnostic**, never a fill rule;
- using `side` (BUY/SELL) or the display label to construct or flip a price — identity selection only;
- any use of `resolved_at` other than as the strict window upper bound; any use of winners/payouts as a feature or a selector.

S1-ALT itself computes **no** `canonical_side_price` and persists **no** price series — statuses, counts, and gap diagnostics only. Whether a *print* price is an acceptable decision-price semantics for the probe (execution price vs quote/mid; staleness tolerance; nearest-vs-first policy alternatives) is a design question the ledger must inform but a later build spec must decide.

---

## 4. Coverage test design before any build (§task item 4)

Same three-level structure as S1, adapted to a local table:

- **Level A — Print availability per token.** Per `(condition_id, token_id)`: `PRINTS_PRESENT` (≥1 valid print for that token anywhere), `PRINTS_ABSENT`, `PRINTS_ALL_KEYLESS` (prints exist for the condition but none carry usable `token_id`/`outcome_index` for this side). Condition rollup both/one/neither.
- **Level B — Decision-window density (BINDING).** Per §3: does each side token have ≥1 valid print in `W`? Threshold **pre-registered and unchanged from S1: a subclass is adequately covered iff ≥ 0.95 of its measured conditions are `DECISION_PRICE_BOTH_SIDES`, and viability requires all three subclasses (UP_DOWN, OVER_UNDER, NAMED_OTHER) to clear independently.** Diagnostics: per-side gap from `first_trade_ts + 3600 s` to first in-window print; in-window print counts per side; window width distribution (`request_window_summary` analogue, so a degenerate window can never masquerade as a coverage negative — S1 lesson, carried forward).
- **Level C — Integrity spot-checks on a pre-registered spread sample** (20–40 conditions across subclasses and time buckets): prices in `[0,1]`; token/index pair stability; timestamp monotonicity and parse fidelity; string-safety end-to-end; complementarity distribution of coincident side prices **as diagnostic only**; and — if Option B is ever authorized — the API-vs-local reconciliation. An all-one-status aggregate at any level triggers `STOP_VALIDATION_REQUIRED`, not a verdict (217/0 & 10/0 lesson).

**Passes.** **Pass 1 (mandatory first, local-only):** the same stratified sampling design as S1 Pass 1 — and, for direct comparability, the **same 300-condition Pass-1 sample** (same per-subclass strata: 50 UP_DOWN / 98 OVER_UNDER / 100 NAMED_OTHER measured after invalid-window exclusion, if the sample manifest is reproducible from the S1 artifacts) plus the same invalid-window exclusion rule. Using the identical sample makes the two candidates' Level-B rates directly comparable per condition. **Pass 2 (full universe, local-only, cheap but still gated):** requires a separate explicit go/no-go after the Pass 1 handoff; Pass 1 success does not auto-authorize it. Because Option A is local, Pass 2 has no network cost, but the gate stands anyway — staged discipline is about review, not bandwidth.

**Verdicts (typed):** `S1ALT_SOURCE_VIABLE`, `S1ALT_SOURCE_PARTIAL` (state exactly which subclass and how much), `S1ALT_SOURCE_NOT_VIABLE`, `S1ALT_INCONCLUSIVE_NO_VALID_DECISION_WINDOW_SAMPLE`, `S1ALT_INCONCLUSIVE_SHAPE` (Option B only), plus typed `STOP_*` halts (§7).

---

## 5. Leakage controls (§task item 5)

- **Decision timestamp:** `first_trade_ts + 3600 s` (accepted `first_price_after_warmup` policy, warmup pinned at 3600 s). No alternative anchor is introduced.
- **`resolved_at`:** used **only** as the strict exclusive upper bound of `W` and to classify invalid windows for exclusion + reporting. Never a feature, never a selector, never a tiebreak.
- **No resolution info as feature:** winners, payout vectors, and `resolved_winning_token_id` never touch token enumeration, sampling, row selection, or classification. Sampling strata are subclass × time bucket only.
- **No post-resolution prints:** any print with `traded_at >= resolved_at` never counts toward Level B (a unit test injects one exactly at `resolved_at` and asserts rejection).
- **No forbidden inference:** any code path computing `1 − price` on a per-side value, reading `yes_price`, or filling one side from the other fails a test by construction → `STOP_LEAKAGE_OR_FORBIDDEN_INFERENCE` at runtime if reached.

---

## 6. Failure modes (§task item 6) — each named, counted, and mapped to a status

1. **Sparse trade prints.** A market may trade in an early burst and then go quiet, leaving zero prints in `W` for one or both tokens despite `PRINTS_PRESENT`. Surfaced by Level B (`DECISION_PRICE_ONE_SIDE`/`NEITHER`) and the gap diagnostics; stratify by time bucket to see whether sparsity concentrates in older conditions.
2. **One-side-only prints.** Volume may concentrate on one token of the pair (the market convention side), leaving the other token print-less in-window. This is the single most plausible failure and it is **not** repairable by complementing — a `DECISION_PRICE_ONE_SIDE` condition is not usable, full stop. Report the one-side rate per subclass explicitly; a high rate is a clean negative for this candidate.
3. **Stale last trade / large decision gap.** The first in-window print may sit hours or days after the decision timestamp. S1-ALT does not decide a staleness tolerance; it **measures** the gap distribution so a later build spec can set (and justify) a maximum-gap rule. An apparently-viable Level-B rate with pathological gap distributions must be flagged in the narrative, not smoothed over.
4. **Identifier precision loss.** Precision loss is an **identity** failure only: `token_id`, `outcome_index`, and any other integer-like identifier where digit-exact string/integer identity is required. 78-digit ids mangled anywhere (parquet dtype drift, CSV round-trip, scientific-notation collapse of an integer id) → `SERIES_MALFORMED`-class row handling and, if systemic, `STOP_PRECISION_LOSS`. Never reconstruct an id. **This failure mode does not apply to `price`:** a numeric price is a small float in `[0,1]`, not an identity, so scientific-notation formatting of a price is normal and is not precision loss (see §2 and §8.5).
5. **API/local mismatch (Option B only).** Same `(condition_id, token_id)` yielding different print sets/prices/timestamps across API and local → `API_LOCAL_MISMATCH` finding; local authoritative for overlap; divergence itself is a trust signal about the gap-fill channel.
6. **Keyless prints.** Rows missing `token_id`/`outcome_index` (~0.4% expected) cannot be side-assigned; excluded per-row and counted (`PRINT_KEY_MISSING`); a condition whose in-window prints are all keyless is `PRINTS_ALL_KEYLESS`, not silently `ABSENT`.
7. **Unstable or `!=2` token pairs.** `TOKEN_PAIR_UNRESOLVED` (excluded + counted); systemic → `STOP_TOKEN_ENUMERATION_UNRELIABLE`.
8. **Dedup ambiguity.** Duplicate `trade_id`s with conflicting price/timestamp after the store's preference rule → counted (`PRINT_DEDUP_CONFLICT`) and excluded from the decision-price pick rather than resolved arbitrarily.

---

## 7. Required artifacts and audit gates (§task item 7)

All under `artifacts/named_binary_probe/price_source_s1_alt/`; **coverage only — no price series persisted**:

- `price_source_s1_alt_coverage.json` — machine-readable: source option(s) exercised, pass number, sample manifest (and whether it reproduces the S1 Pass-1 sample), per-subclass + pooled Level-A/B counts, decision-window-validity split, gap-distribution summaries, Level-C results, exclusion/keyless/dedup counters, `nb_contract_version`, `p0_state` snapshot, typed `s1alt_verdict`, `pass2_available` flag.
- `price_source_s1_alt_coverage.md` — narrative with pessimistic framing: state the null ("local trade prints do not adequately cover the decision window for both sides") and whether it was falsified; name exactly which subclasses/conditions remain uncovered and why; include a side-by-side Level-B comparison against the accepted S1 rates on the shared sample.
- `price_source_s1_alt_coverage_by_condition.csv` — one row per measured condition: subclass, both token_ids (string-safe), Level-A per side, Level-B class, per-side first-in-window gap, in-window print counts. Statuses and gaps only, **no price values**.
- `price_source_s1_alt_excluded.csv` — every excluded condition with reason (`NO_TRADE_ANCHOR`, `TOKEN_PAIR_UNRESOLVED`, `NO_VALID_DECISION_WINDOW_AFTER_WARMUP`, `PRINTS_ALL_KEYLESS`, precision-loss), reconciling exactly: measured + excluded = sample (Pass 1) or = 39,693 (Pass 2). Absent ≠ keyless ≠ error ≠ uncovered-window.
- `price_source_s1_alt_source_shape.md` — the exact local schema/dtypes as observed at run time; if Option B is authorized, the documented-vs-observed API shape verbatim plus the equivalence-check result.

**Audit-gate discipline:** pre-registered threshold (0.95 per subclass, Level B binding) fixed before any run; reconciliation is fail-loud; all-one-status guard mandatory before any aggregate is reported; pure-logic tests (injected shapes, no data) written and passing before any real read — including the leakage test, the forbidden-inference test, the `resolved_at`-boundary test, the keyless-print test, and the precision-loss test. Acceptance of the resulting ledger is an orchestrator decision, not an artifact property.

---

## 8. Explicit stop conditions (§task item 8)

Typed halts; conclude nothing, emit no verdict:

1. `STOP_NOT_AUTHORIZED` — run launched without a separate, explicit, in-chat S1-ALT execution authorization (accepting this spec is not that authorization).
2. `STOP_P0_NOT_CLEAR` — `p0_preflight.json` absent or `p0_state != P0_CLEAR`.
3. `STOP_STALE_CONTRACT` — `nb_contract_version != nb-contract-2026-06-28.1` anywhere consumed.
4. `STOP_TOKEN_ENUMERATION_UNRELIABLE` — large `TOKEN_PAIR_UNRESOLVED` fraction.
5. `STOP_PRECISION_LOSS` — scientific-notation / float-mangled **identifier** anywhere: `token_id`, `outcome_index`, or any other integer-like identifier requiring digit-exact identity. **Applies to identifiers only, never to `price`** — scientific-notation formatting of an ordinary numeric price is not precision loss and never triggers this stop; a price is validated solely as a finite numeric in `[0,1]` (non-finite / out-of-range → per-row `PRINT_PRICE_INVALID`, not a hard stop). Never reconstruct a mangled identifier.
6. `STOP_LEAKAGE_OR_FORBIDDEN_INFERENCE` — any path touching `yes_price` / `1 − p` / side synthesis, any Level-B count using a print at/after `resolved_at`, any outcome-conditioned selection.
7. `STOP_VALIDATION_REQUIRED` — all-one-status aggregate at any level before the Level-C sample passes.
8. `STOP_SOURCE_SHAPE_UNRECOGNIZED` — (Option B only) real API responses deviate from the pinned documented shape; record the deviation, do not coerce.
9. `STOP_SAMPLE_IRREPRODUCIBLE` — the S1 Pass-1 sample manifest cannot be reproduced **and** no pre-registered replacement stratification was approved; stop rather than improvise a sample mid-run.

---

## 9. What would unblock P1 — and what would still keep it blocked (§task item 9)

**Nothing in S1-ALT unblocks P1.** Precisely:

- `S1ALT_SOURCE_NOT_VIABLE` → P1 stays blocked; the trade-print candidate closes negative on this evidence; the remaining option space (Pass 2 generalization checks, Option B gap-fill, or a different source entirely) is a fresh, separately authorized decision. `named_binary_probe_blocked` stays `true`.
- `S1ALT_SOURCE_PARTIAL` → P1 stays blocked. A partially covered universe does not license scoring a truncated universe; whether a subclass-restricted path is even admissible is an orchestrator decision outside this spec.
- `S1ALT_SOURCE_VIABLE` → **P1 still stays blocked.** A favorable coverage ledger only qualifies the candidate for a **separately authorized S2-analogue build spec** (spec-only), which would define the actual per-side price artifact `[condition_id, ts, outcome_index/token_id, side_price]`, its semantics decision for print prices (staleness/gap policy, execution-vs-quote labeling), and its own audit gate — analogous to the Stage 0–4 resolution-source build. Only after that build exists, passes its audit gate, and is accepted does the source become an **[ACCEPTED PRICE SOURCE]**; and even then, resuming P1 is itself a separate explicit authorization. The chain is: coverage finding → (authorize) build spec → (authorize) build + audit gate → (authorize) P1 resume. No link implies the next.
- In every branch: no probe, no scoring, no P2/P3, no wallet/OrdersMatched/`log_index`/PnL, no gate change, `named_binary_probe_blocked` not flipped.

---

## 10. Guardrail restatement

This spec writes no code, runs no data, fetches nothing, builds no artifact beyond this document, and computes no price. It forbids `yes_price`, `1 − price`, `1 − yes_price`, and any side synthesis or complement-fill; sides are priced only from their own token's prints, selected by token identity in `outcome_index` order. `resolved_at` bounds the window and nothing else; winners never select or feature. It reuses — never redefines — the classification contract, the P0 universe, orientation-by-identity, payout-vector winners, the `first_price_after_warmup` policy, the S1 status taxonomy, and the pre-registered 0.95 per-subclass Level-B threshold. Option A (local prints) is the sole Pass-1 candidate; Option B (Data API `/trades`) requires its own authorization and is user-run; Option C (on-chain event reconstruction) is out of scope by guardrail. Executing any pass requires separate, explicit, in-chat authorization; accepting this spec is not that authorization. `named_binary_probe_blocked` stays `true`.
