# SPEC — Option B: Polymarket Data API `/trades` Feasibility (SPEC ONLY)

**Status:** SPEC ONLY — APPROVED at orchestrator review (SPEC ONLY). This document authorizes
nothing beyond the spec text: no implementation, no network/API run, no broad API backfill, no
full indexer, no Pass 2, no S2 build, no P1/P2/P3, no probe, no scoring, no
wallet/OrdersMatched/log_index/PnL, no gate change. `named_binary_probe_blocked` stays `true`
and is not flipped.
**Task chat:** Option B API/online trade-source feasibility, following the accepted Option A
negative.
**Authorization boundary:** This is a feasibility specification only. It does not authorize Phase B0,
Phase B1, a 300-condition Option B Pass 1, or any API request. Each future phase requires a
separate explicit in-chat authorization before code or execution.
**Predecessor findings (SETTLED — not re-litigated here):**
- S1 Pass 1 (CLOB `/prices-history`): ACCEPTED, `S1_SOURCE_NOT_VIABLE` (UP_DOWN 0.38 / OVER_UNDER
  ≈0.5204 / NAMED_OTHER 0.65 vs 0.95).
- S1-ALT Pass 1 Option A (local trade-print reconstruction via `Store.load_trades()`): ACCEPTED,
  `S1ALT_SOURCE_NOT_VIABLE` — UP_DOWN 13/50 = 0.26, OVER_UNDER 40/98 ≈ 0.4081632653, NAMED_OTHER
  71/100 = 0.71. **No subclass cleared 0.95; NAMED_OTHER, the closest, is still 24 points short.**
  P1 remains BLOCKED. No `yes_price` fallback. No `1 - price` synthesis.

---

## 0. Honest framing before anything else (pessimistic, not promotional)

The approved S1-ALT spec originally cast Option B as a **gap-fill candidate, "considered only if
Option A's coverage is close-but-short."** Option A's actual result is not close-but-short — it is
a **decisive rejection in all three subclasses**, with the best subclass 24 points below the bar.
That changes what Option B can honestly be asked to do here: it is not topping up a near-miss, it
is being asked whether a **different retrieval path onto the same underlying on-chain history**
does meaningfully better than the project's own local backfill. There are two very different
reasons it might not:

1. **Local backfill is simply incomplete** — the Data API is Polymarket's own first-party index
   and could cover trades the project's Dune-derived backfill missed or that postdate it. If so,
   Option B could plausibly help.
2. **The underlying markets themselves traded thinly on one side** — a real, structural fact about
   these conditions, not an artifact of how the data was retrieved. If so, Option B faces the
   **same** ceiling Option A did, because no retrieval path can produce a trade that never
   happened.

This spec's whole design (§3–§4) exists to tell these two apart on a small, bounded sample
**before** committing to anything larger. The default expectation stated here is neutral-to-
skeptical, not optimistic — Option A's failure pattern (one-side-only trading dominating the
NEITHER/ONE_SIDE buckets) is at least as consistent with explanation 2 as with explanation 1.

---

## 1. What exact API endpoint/source is proposed? (task item 1)

**Endpoint:** `GET https://data-api.polymarket.com/trades` — Polymarket's public Data API. Verified
directly against the official documentation at
`docs.polymarket.com/api-reference/core/get-trades-for-a-user-or-markets` and
`docs.polymarket.com/api-reference/rate-limits` (fetched for this spec; re-verify immediately
before any authorized run, since public docs can drift).

**Query parameters (verified, current as of this spec):**

| Param | Type | Notes |
|---|---|---|
| `limit` | integer | default 100, range 0–10000 |
| `offset` | integer | default 0, range 0–10000 |
| `takerOnly` | boolean | **default `true`** — see gotcha (a) below |
| `filterType` | enum `CASH`\|`TOKENS` | must pair with `filterAmount`; a size filter, not needed here |
| `filterAmount` | number ≥ 0 | pairs with `filterType` |
| `market` | string[] | comma-separated `0x`-prefixed 64-hex condition IDs; **mutually exclusive with `eventId`** — this is what lets a query be scoped to an exact, pre-registered condition list rather than a broad pull |
| `eventId` | integer[] | mutually exclusive with `market`; not used here (condition-scoped, not event-scoped) |
| `user` | string | 0x-prefixed 40-hex wallet; not used here (we want market-scoped, not user-scoped) |
| `side` | enum `BUY`\|`SELL` | diagnostic filter only; never used to construct or flip a price |

**Response fields (verified, per-trade record):** `proxyWallet`, `side` (`BUY`/`SELL`), `asset`
(token/asset id, type `string`, exact format not specified in the docs — see gotcha (c)),
`conditionId` (`0x`-prefixed 64-hex string), `size`, `price` (numeric, range not specified in the
docs — see gotcha (d)), `timestamp` (`int64`, units not specified — see gotcha (e)), `title`,
`slug`, `icon`, `eventSlug`, `outcome` (display label — **never** a substitute for token identity,
per this project's orientation-by-identity rule), **`outcomeIndex` (integer — directly present)**,
`name`/`pseudonym`/`bio`/`profileImage*` (trader profile metadata, irrelevant here),
`transactionHash`.

**Rate limit (verified):** Data API `/trades` = **200 req / 10 s**, Cloudflare-throttled (delayed,
not hard-rejected) on a sliding window, per the official rate-limits page.

**Critical shape gotchas — every one of these must be resolved empirically on a tiny known
sample (§4, Phase B0) before any count is trusted, exactly the discipline this project already
applies to CLOB `/prices-history` and to `outcome_index`:**

- **(a) `takerOnly` defaults `true`.** Calling without explicitly setting it returns only the
  taker leg of each trade. Whether that yields exactly one row per economic trade, or whether a
  single taker order that filled against multiple maker orders produces multiple taker-side rows
  for what should count as one trade, is **unknown** and must be checked before any row is
  counted as a "print." This is the same row-cardinality class of gotcha as "OrdersMatched fires
  once per trade, OrderFilled twice per trade" (DECISION_LOG, settled) — a lesson this project
  already paid for once.
- **(b) `conditionId` format.** Returned as `0x`-prefixed 64-hex. Whether this is byte-identical
  to local `Store.load_trades()`'s `condition_id` string form is **not asserted here** — it must
  be confirmed, not assumed, before any join.
- **(c) `asset` (token id) format.** The docs show only a placeholder `"<string>"` — the exact
  form (78-digit decimal string vs. something else) is unconfirmed. If confirmed to be the same
  decimal-string identifier space as local `token_id`, the same string-safe /
  `IdentifierPrecisionLoss` discipline applies unchanged: scientific notation or a bare float
  standing in for the id is `STOP_PRECISION_LOSS`, never silently coerced.
- **(d) `price` range/units.** The docs show only a placeholder numeric value (`123`) — whether
  this is a `[0,1]` fraction (matching local trades' `price` and this project's `valid_price`
  contract) or some other scale is unconfirmed.
- **(e) `timestamp` units.** Also a placeholder (`123`) — epoch seconds vs. milliseconds is
  unconfirmed; this changes every window-membership comparison downstream.
- **(f) No time-range query parameter is documented on `/trades` itself** (unlike, e.g., the
  `/activity` endpoint's `start`/`end`). Full historical retrieval for one condition therefore
  requires paging via `limit`/`offset` up to whatever page count that condition's trade history
  needs — this is the main reason an unbounded pull could silently turn into a broad scrape, and
  is why §5 puts a hard per-condition page cap in place.
- **(g) `outcomeIndex` is present directly in the response.** This is a genuine plus if the format
  gotchas above resolve cleanly: unlike Option A, Option B would not require reconstructing a
  token↔slot mapping from raw prints — the API states it directly. This does not, by itself,
  make Option B more likely to *cover* the gaps Option A missed; it only means less enumeration
  work if the response is otherwise trustworthy.

---

## 2. Does it plausibly contain trade prints missing from local `Store.load_trades()`? (task item 2)

**Plausible, not established.** Two specific, competing hypotheses, both consistent with the
same accepted Option A result, and only one of which Option B can fix:

- **H1 — local backfill incompleteness.** The local `trades` parquet is populated by this
  project's own Dune-derived backfill (`pm_research/data/backfill.py`), which has a
  documented ~99.6% key-coverage ceiling and reflects whatever the last backfill run captured.
  Polymarket's own Data API is a first-party live index that could be more complete and more
  current. **If H1 is true, Option B can plausibly help** — it would surface real trades the
  local table is simply missing.
- **H2 — genuine thin/one-sided trading.** Many sampled conditions may have truly had very
  little trading on one side in the real market, independent of how the data was retrieved. The
  dominance of `ONE_SIDE`/`NEITHER` in Option A's accepted result (Level-B counts: BOTH_SIDES
  124 / ONE_SIDE 65 / NEITHER 59, out of 248 measured) is at least as consistent with H2 as with
  H1. **If H2 is true, Option B faces the same ceiling** — no retrieval path invents a trade that
  never happened.

**This spec does not adjudicate H1 vs. H2 — that is exactly what §4's bounded test is for.** No
claim that Option B "should" perform better is made or implied here; the honest, falsifiable
prior is that Option B might simply re-surface the same limited history through a different door.

---

## 3. How would it be validated against local trades on overlapping conditions? (task item 3)

A mandatory **equivalence / reconciliation check** — already named as a requirement in the
approved S1-ALT spec (§1, "mandatory equivalence check") — run on conditions **already measured
locally**, before any coverage-extension claim is entertained:

- **Sample:** a small, fixed, pre-registered set of conditions Option A already classified
  `DECISION_PRICE_BOTH_SIDES` (i.e., conditions where local trades are known-good on both sides).
  Using already-covered conditions, not gaps, is deliberate: it tests trust on ground truth we
  can already check, before touching ground we can't.
- **Per condition, per token:** fetch via `market=<condition_id>` and reconcile the API's
  returned trades against the local `(condition_id, token_id)` print set: same trade count
  (accounting for the `takerOnly` cardinality question, gotcha (a)), same prices, same
  timestamps (accounting for the units question, gotcha (e)), same string-safe ids (gotcha (c)).
- **Two independent mismatch directions, both meaningful, both counted:**
  - `API_LOCAL_MISMATCH: API_HAS_EXTRA` — the API returns trades absent locally. This is the
    **only** finding that would support H1 (local incompleteness) and justify going further.
  - `API_LOCAL_MISMATCH: API_MISSING_KNOWN_LOCAL` — the API is missing trades the local table
    already has. This would cut against Option B's usefulness even before checking the gaps, and
    is reported with equal weight — this spec does not selectively look only for evidence that
    helps Option B's case.
- **Local remains authoritative** for anything both sources report, per the existing S1-ALT rule
  — this equivalence pass is exploratory validation of a new source, never a basis for preferring
  or overwriting local data.
- This check must complete, and must show the API is a **mechanically understood, trustworthy**
  source (row cardinality resolved, id/price/timestamp formats confirmed), **before** touching a
  single condition Option A could not measure. Testing trust on known ground first, unknown
  ground second, is the whole point of splitting this into two phases (§4).

---

## 4. What small, bounded, same-sample coverage test would be allowed if later authorized? (task item 4)

Two phases, each requiring its **own** separate explicit authorization — passing one does not
authorize the next:

### Phase B0 — equivalence / reconciliation pilot (§3, above)
- **Sample:** ~10–20 conditions, pre-registered, drawn from Option A's `DECISION_PRICE_BOTH_SIDES`
  set (known-good local ground truth).
- **Calls:** condition-scoped only (`market=<condition_id>`), one condition at a time; `takerOnly`
  explicitly tried both `true` and `false` on a handful of conditions specifically to resolve
  gotcha (a) empirically.
- **Hard cap:** ≤100 total API calls for the whole phase; ≤5 pages per condition at
  `limit` ≤ 1000.
- **Output:** a reconciliation report only (match/mismatch counts, resolved format questions) —
  no coverage verdict, no price artifact.

### Phase B1 — bounded coverage pilot (gated on B0 passing)
- Runs **only if** B0 shows the API is trustworthy **and** shows at least some
  `API_HAS_EXTRA` evidence (i.e., H1 has some support) — a B0 that shows the API merely mirrors
  local data, or is itself unreliable, closes the Option B line **without** proceeding to B1.
- **Sample:** a small stratified subset — **not** the full 300 — of specifically the conditions
  Option A classified `ONE_SIDE` or `NEITHER` (the actual gaps), e.g. 30–50 conditions across the
  three subclasses. This targets the question directly: does the API newly achieve
  `DECISION_PRICE_BOTH_SIDES` where local prints could not?
- **Method:** identical decision-window and Level-B logic already implemented for S1-ALT — reused
  unchanged, not reinvented, for direct comparability. No new coverage math.
- **Hard cap:** combined with B0, the whole pilot (both phases) stays in the low hundreds of API
  calls, well under the documented 200 req/10s ceiling, paced with explicit sleep/backoff between
  calls (polite-rate, resumable, user-run — same run-ownership terms as S1 and S1-ALT).
- **Not authorized by this spec:** a full 300-condition Pass 1 run of Option B. That would be a
  separate, later, explicitly authorized "S1-ALT-B Pass 1," gated on this bounded pilot's result,
  analogous to how S1-ALT itself required its own authorization after S1.

---

## 5. What halt conditions prevent it from becoming a broad scrape/backfill/indexer? (task item 5)

All typed, all hard stops, checked **before** any call and **during** the run:

- **`STOP_NOT_AUTHORIZED`** — default state. No call without an explicit two-part authorization
  (an execution flag **and** an explicit external-host confirmation), mirroring S1's
  `--i-authorize-...-network-run` + `--confirm-external-host` pattern exactly.
- **`STOP_CALL_BUDGET_EXCEEDED`** — a hard, pre-registered ceiling on total API calls for the
  entire pilot (both phases combined, §4). Exceeding it halts immediately mid-run, regardless of
  how promising partial results look.
- **`STOP_SAMPLE_SCOPE_EXCEEDED`** — any attempt to query outside the pre-registered fixed
  condition-id list. No `eventId`-based broad queries, no unscoped/default-parameter calls that
  would return "recent trades across all markets," no wildcard market lists.
- **`STOP_PAGINATION_UNBOUNDED`** — a hard per-condition page-count/row-count ceiling (e.g., ≤5
  pages or ≤5,000 rows per condition, given gotcha (f)'s lack of a server-side time filter). A
  condition needing more than this to retrieve its full history halts **that condition's** fetch
  and is reported `PARTIAL_RETRIEVAL`, never silently continued past the cap.
- **`STOP_RATE_LIMIT_HIT`** — a persistent throttle/429-equivalent response halts the run; no
  indefinite retry loop.
- **`STOP_SCHEMA_DEVIATION`** — if the observed response shape deviates from the pinned shape in
  §1 (renamed/removed field, new auth requirement, changed pagination contract), halt and report
  rather than silently adapting.
- **`STOP_PRECISION_LOSS`** — identical discipline to Option A: `asset`/token id must stay
  string-safe end to end (scoped to identifiers only, exactly as corrected for Option A —
  `price` is never subject to this).
- **No forbidden inference** — no `yes_price`, `1 - price`, `1 - p`, `1 - yes_price`, no
  side-complement synthesis, ever. Reused unchanged from Option A, not redefined.
- **No write path** — this pilot never writes to `prices/`, never persists a price series or
  artifact beyond a coverage/reconciliation report, never touches
  wallets/OrdersMatched/log_index/PnL, never modifies any gate, never flips
  `named_binary_probe_blocked`.
- **No autonomous escalation** — B0 passing does not auto-authorize B1; B1 passing does not
  auto-authorize a full Pass 1; nothing here authorizes Pass 2, an S2 build, P1/P2/P3, or the
  probe. Every phase transition needs its own fresh, explicit, in-chat authorization.

---

## 6. What result would be required to justify any later build spec? (task item 6)

Mirroring the S1 / S1-ALT precedent exactly — a later S2-style **build** spec (an actual
persisted per-side price artifact) is gated on **favorable full Pass-1 coverage**, not on a
favorable pilot, and not on "some improvement":

- **B0 must show the API is mechanically trustworthy** on known ground — row cardinality
  resolved, id/price/timestamp formats confirmed, no unexplained `API_MISSING_KNOWN_LOCAL`
  pattern. An untrustworthy or ambiguous B0 closes the line here, regardless of B1's hypothetical
  promise.
- **B1 must show material, non-anecdotal improvement** specifically on the `ONE_SIDE`/`NEITHER`
  conditions — not one or two isolated fixed conditions, but a large-enough pilot sub-sample to
  be meaningful, ideally trending toward the same 0.95 both-sides bar becoming newly plausible.
  The exact statistical bar is the Orchestrator's call, but it should be **no looser** than what
  S1 and S1-ALT were held to — this project does not lower its own bar for a candidate it wants
  to work.
- **Even a fully favorable B1 only justifies a separately-authorized full "S1-ALT-B" Pass 1 run**
  on the same 300-condition sample — not a build spec directly. The build-gate condition stays
  favorable full Pass-1 coverage, exactly as it was (and was not met) for S1 and S1-ALT.
- **A negative or ambiguous B0/B1 result closes Option B** on the same terms S1 and S1-ALT were
  closed: documented, accepted, `DECISION_LOG`-recorded. At that point the per-side price line for
  P1 would have exhausted every candidate considered in this branch. On-chain event reconstruction
  is out of scope under the current guardrails and is not authorized or proposed by this spec.
  That exhaustion should be recorded honestly, not framed as more options remaining than
  actually do.

---

## 7. Guardrail restatement

This spec writes no code, runs no data, fetches nothing from Polymarket, and builds no artifact
beyond this document. It forbids `yes_price`, `1 - price`, `1 - p`, `1 - yes_price`, and any
side-complement synthesis. It reuses — never redefines — the accepted decision-window policy,
the S1/S1-ALT Level-B logic and 0.95 per-subclass threshold, the accepted 300-condition Pass-1
sample, and the precision-loss discipline (scoped to identifiers, never to price). No broad API
backfill, no full indexer, no Pass 2, no S2 build, no P1/P2/P3, no probe, no scoring, no
wallet/OrdersMatched/log_index/PnL work, no gate change is authorized by this document. Executing
Phase B0, Phase B1, or any part of this plan requires separate, explicit, in-chat authorization;
accepting this spec is not that authorization. `named_binary_probe_blocked` stays `true`.
