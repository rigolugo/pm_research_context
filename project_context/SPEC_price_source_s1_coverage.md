# SPEC — S1 Price-Source Coverage Test: CLOB `/prices-history` per token

**Status:** SPEC ONLY — **APPROVED at orchestrator review (SPEC ONLY)**. Approval covers the spec text only; it authorizes nothing further — no S1 implementation, no network/data run, no S2, no P1/P2/P3, no probe execution, no scoring, no backfill, no artifact build beyond this document. `named_binary_probe_blocked` stays `true` and is not flipped.
**Task chat:** Price-source build plan, Stage S1 (coverage).
**Scope:** First stage of the "next possible step" in `START_HERE.md` / `PROJECT_STATE.md` — a per-side / token-identity-keyed decision-time price series to unblock the (still-blocked) named-binary P1. This stage tests **only whether a usable per-token price source can cover the P0 universe**; it does not build the price artifact and does not compute any canonical-side price.
**Consumes (does not redefine):** the accepted named-binary classification contract (`nb-contract-2026-06-28.1`), the P0 eligible universe (`final_p0_eligible = 39,693`), and the resolution-source parquet. It **does not** redefine named-binary, orientation, or winners.
**Reconciles with:** GUARDRAILS.md, PROJECT_STATE.md, DECISION_LOG.md, CLOSED_FINDINGS.md, ARTIFACT_INDEX.md, DATA_CONTRACTS_named_binary_probe.md, PRICE_INPUT_CONTRACT_named_binary_probe.md, DUNE_DATA_NOTES.md, CLAUDE_PROJECT_SETTINGS.md.

---

## Epistemic tiers (used throughout this spec)

Per the orchestrator's evidence packet, S1 distinguishes three levels of confidence about the CLOB price source. These labels are used explicitly wherever a claim about the endpoint appears:

- **[DOCUMENTED SHAPE]** — supported by official Polymarket documentation. The **request/response contract** is documented (see §5). This means S1 does **not** have to reverse-engineer the parameter names or route; it may write the run against the documented shape. It does **not** establish how the endpoint *behaves* on this project's specific historical/resolved tokens.
- **[CANDIDATE BEHAVIOR]** — must be **measured by S1** on the user's box. Whether the documented endpoint actually returns usable, both-sided, decision-window, precision-safe data for the P0 universe is an empirical question S1 answers. Documentation is not behavior; a documented endpoint can still return blank/sparse/coarse histories for old closed markets.
- **[ACCEPTED PRICE SOURCE]** — a status reached **only after** S1 coverage measurement *and* a later S2 build + audit gate pass. Nothing in S1 confers it. Until then, the CLOB source is a *candidate*, never the accepted per-side price input, and `named_binary_probe_blocked` stays `true`.

**Consequence for this revision:** the endpoint shape is no longer written as an "unsupported assumption." It is **[DOCUMENTED SHAPE]**. What remains empirical is **[CANDIDATE BEHAVIOR]** — the eight verification points in §5.1, which are exactly what S1 measures.

---

## 0. Why this stage exists (the block it addresses)

S0 proved from source (`PRICE_INPUT_CONTRACT_named_binary_probe.md`, accepted) that:
- `OrientationContract.canonical_side_price(side_0_price, side_1_price)` needs **two per-side prices** in `outcome_index` order;
- local `Store.load_prices()` supplies only `[condition_id, ts, yes_price]` — one scalar;
- `yes_price` is a YES-label flip (`price where outcome=="YES" else 1-price`), **YES/NO-only and source-declared unsafe** for UP_DOWN / OVER_UNDER / NAMED_OTHER;
- **no local per-side/per-token price artifact exists.**

The only forward path (if authorized) is to **source a per-token price series**. Polymarket's CLOB `/prices-history` endpoint returns a time series **keyed by `token_id`** (`market=<token_id>`), which is exactly the per-side discriminator the local `prices` table lacks. **Before** anyone specifies or builds such an artifact, we must know whether that endpoint can actually cover the P0 universe with usable, decision-time-relevant data. That is S1: **a coverage test, not a build.**

**S1 answers one question:** *For the 39,693 P0-eligible non-YES/NO conditions (each with two token_ids), can CLOB `/prices-history` return a per-token price series with enough coverage, and enough density around the decision timestamp, to feed `canonical_side_price` — and where does it fail?* A NEGATIVE or partial answer is an acceptable, expected outcome and must be reported plainly (project discipline: falsify cleanly).

---

## 1. Explicit non-goals (what S1 is NOT)

S1 does **not**:
- build or write any price artifact (`side_price` series, per-token parquet, etc.);
- backfill prices, invoke `backfill_prices_from_clob`, or persist any fetched series into the store;
- compute `canonical_side_price`, `yes_price`, or any oriented price;
- synthesize `side_0_price = yes_price` or `side_1_price = 1 - yes_price` (source-forbidden — Q2 of PRICE_INPUT_CONTRACT);
- score anything (no Brier / log-loss / calibration / reliability / splits / edge / PnL);
- continue P1/P2/P3 or run the probe;
- touch wallets / OrdersMatched / `log_index`;
- modify the audit gate or flip `named_binary_probe_blocked`;
- select or rank conditions by any outcome or profitability signal.

S1 is a **read-only feasibility measurement** whose only output is a coverage report + machine-readable coverage tables. Whether to then spec a build (S2) is a **separate, user-authorized** decision informed by S1's numbers.

---

## 2. Authorization & environment boundaries (must be honored by the S1 run, when later authorized)

- **This spec authorizes no run.** Executing S1 requires a separate, explicit, in-chat user authorization naming the S1 coverage run. (Mirrors SPEC_named_binary_probe §12 / §9.7 discipline.)
- **Network:** the S1 run hits an external host (`clob.polymarket.com`) that is **not** on Claude's package-managers-only allowlist (CLAUDE_PROJECT_SETTINGS.md). Therefore **S1 is a user-run task on the Windows/Miniconda box** (`C:\b1\pm_research`, env `pmresearch`), exactly like the Dune data stages — **Claude does not fetch Polymarket data**. Any network access for Claude would need a separate, one-domain, named-task approval, and even then data fetching by Claude is not currently authorized. Assume **user-run** throughout.
- **Read-only w.r.t. project state:** S1 writes only its own coverage artifacts under `artifacts/named_binary_probe/price_source_s1/`. It does not write into `prices/`, does not alter the store, does not alter the gate.
- **Rate/etiquette:** external public endpoint — the run must be polite (bounded rate, backoff, resumable), because the P0 universe implies a large number of token requests (§4.2). Exact limits are unknown to Claude and must be treated as **[ASSUMPTION — verify in-run]**, not asserted.

---

## 3. Inputs (all already-materialized artifacts; none re-derived)

| Input | Source artifact | Use in S1 | Leakage note |
|---|---|---|---|
| P0 eligible universe | `artifacts/named_binary_probe/p0_preflight.json` (`counts_*`, `p0_state == P0_CLEAR`) + the join it certifies | The 39,693 condition set to coverage-test; assert `P0_CLEAR` before running | Static; count-only |
| Per-condition subclass + token identities | `named_binary_classification_contract.json` (`conditions`: `condition_id`, `nb_subclass`) **+** the two per-condition token_ids | Enumerate the `(condition_id, token_id)` pairs to probe | Static classification; no outcome content |
| Resolved winners / `resolved_at` | `named_binary_resolution_source_rows.parquet` | **Only** to compute the decision-window reference and to *report* whether price coverage exists strictly before resolution; **never** as a feature and **never** to select tokens | `resolved_at` is leakage-sensitive (DATA_CONTRACTS §2) — used here only for a **coverage diagnostic** (does price exist before resolution?), not for any score |
| First-trade time per condition | local `trades` via `Store(root).load_trades()` → `min(traded_at)` per `condition_id` | Anchor the warm-up / decision timestamp used to test **decision-window density** | Ex-ante anchor only |
| Warm-up policy | pinned `warmup_seconds = 3600` (= Rank 1A `--warmup-hours 1.0`, DATA_CONTRACTS §7) | Define the decision timestamp `first_price_after_warmup` for density measurement | Lookahead-safe (DECISION_LOG) |
| Token identities per condition | **[DEPENDENCY — see §3.1]** | Provide the two `token_id`s (in `outcome_index` order) that `/prices-history` is keyed on | Static identity |

### 3.1 Token-pair source — **[RESOLVED — orchestrator review]**
`/prices-history` is keyed on a **single token_id** (`market=<token_id>`), so S1 must enumerate, per eligible condition, its **two** side token_ids in `outcome_index` order (index 0 = side_0, index 1 = side_1).

**Locked decision:**
- **Primary enumeration basis = `trades` distinct `(condition_id, token_id, outcome_index)`.** The `trades` table carries `token_id` + `outcome_index` per print (DATA_CONTRACTS §5); its distinct `(condition_id, token_id, outcome_index)` tuples are the candidate source of both side tokens per condition.
- **Not assumed valid.** This basis is *candidate*, not accepted. The **first S1 implementation step (if later authorized)** must **validate that each P0-eligible condition yields exactly two stable, string-safe side tokens** — two distinct `token_id`s, one per `outcome_index ∈ {0,1}`, stable across the condition's trades, string-safe (78-digit; `canonical_int`, fail loud on scientific notation — DUNE_DATA_NOTES §5). Validation precedes any fetch.
- **`resolved_winning_token_id` remains forbidden as a pair source** — it is a single side and is **outcome-conditioned**; selecting or enumerating tokens from it would leak the outcome. It may be used only as the winner label downstream (in a later, separately-authorized stage), never to build the side pair.
- **Failure handling:** a condition whose pair is missing, unstable, or `!= 2` distinct tokens → `TOKEN_PAIR_UNRESOLVED` (counted, excluded from coverage math, reconciled in `price_source_s1_excluded.csv`). A **large** unresolved fraction → `STOP_TOKEN_ENUMERATION_UNRELIABLE` (§8.3): the enumeration basis is not trustworthy and S1 stops rather than measuring coverage on a shaky universe.

This resolves the prior open dependency; §11 Q1 is closed.

---

## 4. What "coverage" means here (the measurements S1 produces)

Coverage is measured at **three levels**, each reported per-subclass (UP_DOWN, OVER_UNDER, NAMED_OTHER) and pooled. No level is collapsed into a single pass/fail number without the underlying counts.

### 4.1 Level A — Token-endpoint reachability (does the series exist at all?)
For each `(condition_id, token_id)` pair, record the endpoint outcome as a typed status:
- `SERIES_PRESENT` — endpoint returned a non-empty time series for this token;
- `SERIES_EMPTY` — endpoint returned successfully but with zero points;
- `SERIES_ERROR_TRANSIENT` — network/5xx/timeout after the retry budget (counted, retryable — NOT a coverage verdict);
- `SERIES_ERROR_NOTFOUND` — endpoint indicates no such market/token (4xx of the "unknown market" kind);
- `SERIES_MALFORMED` — returned but unparseable / precision-loss signature (fail loud, do not coerce).

Report: per-subclass and pooled counts of each status; **condition-level** rollup (both sides present / one side present / neither) since `canonical_side_price` needs **both** per-side prices — a condition with only one reachable side is **not** price-usable.

### 4.2 Level B — Decision-window density (is there a price *at the decision timestamp*?)
The probe's decision timestamp is `first_price_after_warmup` = first price ≥ (first_trade_ts + 3600 s). For each eligible condition where **both** sides are `SERIES_PRESENT`:
- compute `first_trade_ts = min(traded_at)` from local trades;
- for **each** side token series, determine whether there is ≥ 1 price point at/after `first_trade_ts + warmup` and strictly before `resolved_at` (leakage-safe window);
- classify the condition:
  - `DECISION_PRICE_BOTH_SIDES` — both sides have a usable point in the window (price-usable for the probe);
  - `DECISION_PRICE_ONE_SIDE` — only one side does (not usable; `canonical_side_price` needs both);
  - `DECISION_PRICE_NEITHER` — neither side has an in-window point;
  - `NO_TRADE_ANCHOR` — condition has no local trades to anchor warm-up (report separately; do not force a timestamp).
- Also record, as a **diagnostic distribution only** (not a score): granularity actually returned (e.g. hourly vs minute vs sparse), and the time gap between the decision timestamp and the nearest available price point per side. This tells S2 whether the series is dense enough to trust a decision-time read or whether interpolation/nearest-point rules would be needed (a design input, not decided here).

> **Leakage discipline (non-negotiable).** `resolved_at` is used **only** to bound the window and to *report* pre-resolution availability. No price at/after resolution is counted as decision-window coverage. S1 produces **no** oriented price and **no** target; it only asks "does a usable per-side point exist in the ex-ante window." (Mirrors SPEC_named_binary_probe §6.2 lookahead-safety.)

### 4.3 Level C — Series integrity spot-checks (is the returned data trustworthy?)
On a **small, pre-registered validation sample** (e.g. 20–40 conditions spread across the three subclasses and across time buckets), verify — against an authoritative cross-check, never by assumption:
- returned price values are in `[0,1]` and the two sides are **plausibly complementary** at coincident timestamps (record the distribution of `side_0 + side_1`; do **not** enforce exact `=1` — CLOB books need not sum to 1 intraday, and this is a *diagnostic*, not a correctness gate, and explicitly **not** a license to reconstruct one side from the other);
- `token_id` echoed / addressable matches the requested token (identity integrity);
- timestamps are monotone and parseable;
- **precision:** any 78-digit id or large integer field is handled string-safe; scientific notation anywhere → `SERIES_MALFORMED`, fail loud (`DATA_EXPORT_PRECISION_LOSS` discipline, DUNE_DATA_NOTES §5).

Level C is a **trust check on a sample**, following the project rule: never conclude coverage from a single row or an all-one-status output — validate a known/spread sample before believing the aggregate. An all-`SERIES_PRESENT` or all-`SERIES_EMPTY` result is a red flag to validate, not a result to report.

---

## 5. The `/prices-history` request/response shape — **[DOCUMENTED SHAPE]**

Per the orchestrator's evidence packet, the endpoint contract is supported by official Polymarket documentation and is therefore **[DOCUMENTED SHAPE]**, not an unsupported assumption. S1 may write its run against this documented contract. Documentation supports:

- **`GET /prices-history`** — retrieves historical price data.
  - **`market`** (required) — the **market asset ID** (i.e. the token id) to query.
  - **`startTs`**, **`endTs`** — time range bounds.
  - **`interval`** — time interval selector.
  - **`fidelity`** — resolution/granularity of returned points.
- **`POST /batch-prices-history`** — documented batch variant.
  - **`markets`** — a **list** of market asset IDs.
  - **`start_ts`**, **`end_ts`**, **`interval`**, **`fidelity`** — batch analogues (note the snake_case vs the single-token camelCase `startTs`/`endTs` — record which each variant actually requires).
- **Response:** a historical time series of points, each carrying a timestamp and a price `p`.

> The documented shape removes the earlier "reverse-engineer the parameters first" risk. What it does **not** remove is the need to confirm the endpoint's **behavior** on this project's specific historical, resolved, often old/closed tokens — that is **[CANDIDATE BEHAVIOR]** and is the substance of S1 (§5.1). Documentation establishes the contract; only measurement establishes coverage. The run must still record the *actual* observed shape verbatim in `price_source_s1_endpoint_shape.md` and fail loud (not coerce) if a real response deviates from the documented contract — a deviation is a finding, and its own signal about source trustworthiness.

### 5.1 Empirical questions S1 must answer — **[CANDIDATE BEHAVIOR]** (measured on the user's box)

These are the eight verification points from the orchestrator's packet. Each is a measurement, not an assumption, and each maps to a coverage level in §4:

1. **Do historical/resolved P0 token IDs return any data at all?** (Old, closed, resolved markets may return nothing.) → Level A (§4.1), `SERIES_PRESENT` vs `SERIES_EMPTY` / `SERIES_ERROR_NOTFOUND`.
2. **Do BOTH `side_0_token` and `side_1_token` return data?** `canonical_side_price` needs both; one-sided coverage is not usable. → Level A condition-level rollup (both / one / neither).
3. **Does a point exist at or after `first_trade_ts + 3600 s`** (the decision timestamp), strictly before `resolved_at`? → Level B (§4.2), the **binding** viability criterion.
4. **Do returned timestamps have usable fidelity?** Record the actual granularity returned (and how `interval`/`fidelity` affect it); a coarse/sparse series may miss the decision window even when "present." → Level B diagnostic (granularity distribution, nearest-point gap).
5. **Is the price `p`'s meaning sufficient, or must it be labeled source-defined?** S1 does **not** assert `p` equals a canonical per-side probability. It records `p` as a **source-defined CLOB history price for the queried token**, in `[0,1]` if so observed, and defers any semantic conversion to S2. No conversion, no `1 - p`, is performed in S1. → Level C (§4.3), value-range + complementarity *diagnostic only*.
6. **Do old/closed markets produce blank, sparse, or coarse histories?** Report this explicitly by market age / time bucket — it is the single most likely failure mode for a universe of resolved conditions. → Level A + Level B, stratified by time bucket.
7. **Does `POST /batch-prices-history` behave equivalently to single-token `GET /prices-history`?** If batch is used to reduce request count (§6), S1 must confirm — on a spread sample — that batch returns the same per-token series as the single-token call (same points, same fidelity, correct token attribution). Any divergence → prefer the single-token call for correctness and record the discrepancy. Batch is a cost optimization, never a source of truth over the documented single-token route until proven equivalent. → Level C validation-sample cross-check.
8. **Do all token IDs remain string-safe with no precision loss** end-to-end (request construction, response parse, ledger write)? 78-digit ids; scientific notation anywhere → `SERIES_MALFORMED` / `STOP_PRECISION_LOSS`, never reconstruct (DUNE_DATA_NOTES §5). → Level C + stop condition (§8.4).

A NEGATIVE answer on points 1–3 (systemic absence, one-sided-only, or no decision-window point) for a subclass is a legitimate **[CANDIDATE BEHAVIOR]** finding that the source does not cover that subclass — report it plainly; do not paper over it with the documented shape.

---

## 6. Sampling strategy (bounded first, full only if warranted)

Because the universe is large (39,693 conditions × 2 tokens ≈ ~79k token requests) and the endpoint is an external public service with unknown limits, S1 runs in **two gated passes**:

- **Pass 1 — stratified probe (small, mandatory first).** A pre-registered stratified sample across the three subclasses and across time buckets (e.g. a few hundred conditions total, both tokens each). Purpose: exercise the **[DOCUMENTED SHAPE]** (§5), get first-order Level-A/B/C numbers, measure the eight **[CANDIDATE BEHAVIOR]** points (§5.1) — including the batch-vs-single equivalence check (point 7) — and estimate request cost + failure modes **before** committing to the full sweep. Pass 1 alone may already answer "is this source viable at all?" — if Pass 1 shows systemically empty/absent series (likely for old closed markets, §5.1 point 6) or unusable decision-window density, S1 reports **`S1_SOURCE_NOT_VIABLE`** and stops (no full sweep needed — falsify cheaply).
- **Pass 2 — full universe sweep (only after a separate explicit go/no-go).** Enumerate all eligible `(condition_id, token_id)` pairs, resumable/checkpointed, with the polite rate policy. May use `POST /batch-prices-history` to cut request count **only if** Pass 1 proved batch-equals-single (§5.1 point 7); otherwise single-token `GET`. Produces the full coverage tables.

**Pass 2 gating — [LOCKED — orchestrator review]:** Pass 1 is **mandatory**. Pass 2 requires a **separate, explicit go/no-go decision after the Pass 1 handoff is reviewed**. **Pass 1 success does not auto-authorize Pass 2** — encouraging Pass 1 numbers are necessary but not sufficient; the full sweep is a larger data pull and a fresh decision. This mirrors the project's staged, gated discipline (each stage's handoff reviewed before the next begins).

---

## 7. Output artifacts (coverage only — no prices persisted)

All under `artifacts/named_binary_probe/price_source_s1/`:

- `price_source_s1_coverage.json` — machine-readable: endpoint shape as verified (§5), pass (1 or 2), per-subclass + pooled Level-A status counts, Level-B decision-window classifications, Level-C validation-sample results, request-cost stats, `nb_contract_version`, `p0_state` snapshot, run bounds, and a typed `s1_verdict`.
- `price_source_s1_coverage.md` — narrative: what shape the endpoint really has, the coverage numbers, the honest verdict (viable / partially-viable-with-caveats / not-viable), and precisely **which conditions/subclasses would remain uncovered** and why. Pessimistic framing: state the null ("this source does not adequately cover the universe") and whether S1 falsified it.
- `price_source_s1_coverage_by_condition.csv` — one row per eligible condition: subclass, both token_ids (string-safe), Level-A status per side, condition-level both/one/neither, Level-B decision-window class, nearest-point gap per side. **This is a coverage ledger, not a price series** — it carries statuses and gaps, **not** price values.
- `price_source_s1_endpoint_shape.md` — the **[DOCUMENTED SHAPE]** (§5) alongside the **verbatim observed** request/response as returned on the box, so a future S2 build spec consumes a *verified* shape, not an assumed one (the DATA_CONTRACTS pattern). Records: which time params each variant actually requires (`startTs`/`endTs` vs `start_ts`/`end_ts`), observed `interval`/`fidelity` behavior, the source-defined meaning of `p` (§5.1 point 5), and the batch-vs-single equivalence result (§5.1 point 7).
- `price_source_s1_excluded.csv` — conditions dropped from measurement with reason (`NO_TRADE_ANCHOR`, `TOKEN_PAIR_UNRESOLVED`, `PRECISION_LOSS`, `SERIES_ERROR_NOTFOUND_BOTH_SIDES`), reconciling to the 39,693 total.

Counts must **reconcile to the P0 universe** (39,693) exactly, with every condition landing in exactly one coverage bucket — the same reconcile-or-fail discipline P0 uses (missing ≠ ambiguous; here: absent ≠ empty ≠ error ≠ uncovered-window).

## 7.1 Typed `s1_verdict` values
- `S1_SOURCE_VIABLE` — both-sides decision-window coverage clears a **pre-registered** threshold across all three subclasses (threshold proposed below, §7.2), validation sample clean.
- `S1_SOURCE_PARTIAL` — usable for some subclass(es) but not others, or usable only outside the decision window; report exactly which and how much.
- `S1_SOURCE_NOT_VIABLE` — systemic empty/absent/low-density; the source does not unblock P1. (Acceptable, expected-possible outcome.)
- `S1_INCONCLUSIVE_SHAPE` — real responses deviate from the **[DOCUMENTED SHAPE]** (§5) in a way that prevents reliable measurement; stop and report the deviation rather than coerce.
- typed `STOP_*` halts (§9).

### 7.2 Coverage threshold — **[LOCKED — orchestrator review]**
- A subclass is **adequately covered iff ≥ 95%** of its P0-eligible conditions are `DECISION_PRICE_BOTH_SIDES`.
- **Pooled viability requires all three subclasses to clear** the 95% bar independently — a large subclass never masks a starved one (OVER_UNDER is small — 1,003 resolved — and must not be hidden by UP_DOWN's 22,012). This mirrors the Stage 4 two-tier logic (pooled + per-subclass floors).
- **Level B (decision-window both-sides density) is the binding criterion, not Level A (mere series existence).** A token series that exists but has no usable point at/after `first_trade_ts + 3600 s` and before `resolved_at` does **not** count toward coverage. Viability is about a usable price *at the decision timestamp*, not about some history existing.
- The threshold is **pre-registered and fixed before any run** so the verdict is not fit to whatever comes back.

---

## 8. Stop conditions (hard halts — fail loud, conclude nothing)

The S1 run (when authorized) aborts with a typed status rather than emit numbers if:

1. **P0 not clear** — `p0_preflight.json` absent or `p0_state != P0_CLEAR` → `STOP_P0_NOT_CLEAR`.
2. **Stale contract** — contract / resolution-source `nb_contract_version != nb-contract-2026-06-28.1` → `STOP_STALE_CONTRACT`.
3. **Token pair unresolved** — cannot obtain exactly two stable, string-safe, outcome-independent side token_ids for a condition from the locked enumeration basis (`trades` distinct `(condition_id, token_id, outcome_index)`, §3.1) → that condition is `TOKEN_PAIR_UNRESOLVED` (counted, excluded from coverage math); if this affects a large fraction, `STOP_TOKEN_ENUMERATION_UNRELIABLE`.
4. **Precision loss** — scientific-notation / float-mangled token id or price anywhere → `STOP_PRECISION_LOSS`, never reconstruct (DUNE_DATA_NOTES §5).
5. **Endpoint shape deviates from documented contract** — a real response cannot be parsed to the **[DOCUMENTED SHAPE]** (§5) → `STOP_ENDPOINT_SHAPE_UNRECOGNIZED` (do not coerce; record the deviation as a finding — it is itself a signal about source trustworthiness).
6. **All-one-status output** — an aggregate that is all `SERIES_PRESENT`, all `SERIES_EMPTY`, or all-one-decision-class **before** the Level-C validation sample passes → treat as a red flag, run/inspect the validation sample, and `STOP_VALIDATION_REQUIRED` rather than reporting the aggregate as a finding (217/0 & 10/0 lesson).
7. **Leakage guard** — if any decision-window measurement would require a price at/after `resolved_at`, or any attempt is made to derive a side price from `yes_price` / `1 - yes_price` → `STOP_LEAKAGE_OR_FORBIDDEN_INFERENCE`.
8. **Not authorized** — run launched without the explicit S1 execution authorization → `STOP_NOT_AUTHORIZED`.

---

## 9. Proposed tests (written & passing before any real fetch, in the later authorized phase)

Pure-logic, no network — mirroring the project's "tests pass on injected shapes first" pattern (and its caveat that fixtures must later mirror the *verified* real shape, HANDOFF P1_REVIEW):
- **Universe load:** only P0-eligible non-YES/NO conditions enter; YES_NO / UNUSABLE never appear; counts reconcile to 39,693.
- **Token-pair enumeration:** exactly two distinct string-safe token_ids per condition from the chosen outcome-independent source; missing/!=2 → `TOKEN_PAIR_UNRESOLVED`, never guessed.
- **Status mapping:** injected endpoint responses map to the correct Level-A status (present / empty / notfound / transient / malformed), including a precision-loss body → `SERIES_MALFORMED` / `STOP_PRECISION_LOSS`.
- **Decision-window logic:** injected series + trade anchors classify Level-B correctly; a point exactly at `first_trade_ts+warmup` counts, a point at/after `resolved_at` never counts (leakage test).
- **Both-vs-one-side rollup:** condition with one side present → `DECISION_PRICE_ONE_SIDE`, not usable.
- **Reconciliation:** every condition lands in exactly one bucket; buckets sum to 39,693 (fail loud otherwise).
- **All-one-status guard:** an all-one-status aggregate triggers `STOP_VALIDATION_REQUIRED`.
- **Forbidden inference:** any code path attempting `side = 1 - yes_price` (or `1 - p` to synthesize the other side) fails a test by construction (the run must not contain it).
- **`p` labeling:** the ledger records `p` as a source-defined CLOB history price, never as an asserted canonical probability; no S1 code converts `p` to a canonical side price (§5.1 point 5).
- **Batch equivalence:** injected batch vs single-token responses for the same tokens must be compared point-for-point; a divergence flags the token and forces single-token as authoritative (§5.1 point 7).
- **No-write guarantee:** the run touches nothing under `prices/`, does not modify the gate, does not flip `named_binary_probe_blocked`.

---

## 10. How S1 feeds the next stage (and what it deliberately leaves open)

- If `S1_SOURCE_VIABLE` / `S1_SOURCE_PARTIAL`: the coverage ledger + verified endpoint shape become the inputs to a **separately-authorized S2 price-source build spec** (spec-only), which would define the actual `[condition_id, ts, outcome_index/token_id, side_price]` artifact, its audit gate, and its own coverage/precision acceptance — analogous to the Stage 0–4 resolution-source build. Only when S2's build + audit gate pass does the CLOB source become an **[ACCEPTED PRICE SOURCE]**; S1 never confers that status. S1 does **not** design the S2 artifact; it only certifies whether the source can support it and flags density/granularity/`p`-meaning constraints S2 must handle.
- If `S1_SOURCE_NOT_VIABLE`: P1 stays blocked; the named-binary probe remains unreachable from local+CLOB data, and that is a clean, reportable negative. No fallback to `yes_price` is permitted.
- Either way, S1 changes **no** project status by itself: `named_binary_probe_blocked` stays `true`, the probe stays unauthorized, and P2 build is a fresh decision.

---

## 11. Design decisions — **[RESOLVED at orchestrator review]**

All prior open questions are now locked. Recorded here for traceability:

1. **Token-pair source (§3.1) — RESOLVED.** Primary enumeration basis = `trades` distinct `(condition_id, token_id, outcome_index)`; **candidate, not assumed valid** — the first S1 implementation step (if authorized) must validate exactly two stable, string-safe side tokens per P0-eligible condition before any fetch. `resolved_winning_token_id` is **forbidden** as a pair source (outcome-conditioned). Missing/unstable/`!=2` → `TOKEN_PAIR_UNRESOLVED`; large failure rate → `STOP_TOKEN_ENUMERATION_UNRELIABLE`.
2. **Coverage threshold (§7.2) — LOCKED.** Subclass adequately covered iff ≥ 95% of P0-eligible conditions are `DECISION_PRICE_BOTH_SIDES`; pooled viability requires all three subclasses to clear; **Level B decision-window both-sides density is the binding criterion, not Level A existence.** Pre-registered before any run.
3. **Run ownership / network — LOCKED.** S1 is **user-run on the Windows/Miniconda box**; **Claude does not fetch Polymarket data**; **no change to the network allowlist is implied** (CLAUDE_PROJECT_SETTINGS.md unchanged).
4. **Pass-2 gating (§6) — LOCKED.** Pass 1 mandatory; Pass 2 requires a **separate explicit go/no-go after the Pass 1 handoff**; **Pass 1 success does not auto-authorize Pass 2.**
5. **Binding criterion (§7.2) — LOCKED.** Level B (decision-window both-sides density), not Level A (mere existence), determines viability.

---

## 12. Guardrail restatement

This spec writes no code, runs no data, fetches nothing, builds no price artifact, and computes no price. It does not continue P1/P2/P3, does not score, does not touch wallets/OrdersMatched/log_index/PnL, does not modify the audit gate, and does not flip `named_binary_probe_blocked` (stays `true`). It forbids `side_0_price = yes_price` / `side_1_price = 1 - yes_price` and any `1 - p` side-synthesis (source-proven unsafe, S0). It reuses — never redefines — the named-binary contract (`nb-contract-2026-06-28.1`), orientation-by-identity, payout-vector winners, and the `first_price_after_warmup` policy. The endpoint request/response contract is **[DOCUMENTED SHAPE]** (official Polymarket docs); what S1 measures is **[CANDIDATE BEHAVIOR]** (the eight §5.1 points); an **[ACCEPTED PRICE SOURCE]** status is reached only after a later S2 build + audit gate, never by S1. Executing S1 requires separate, explicit, in-chat user authorization; accepting this spec is not that authorization.
