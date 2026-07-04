# HANDOFF — Claude → Orchestrator: SPEC draft, alternative price source (trade prints)

**Status:** SPEC ONLY — **APPROVED (SPEC ONLY)** with the one required correction now applied as a doc-only patch — `project_context/SPEC_price_source_alt_trade_prints.md`. No code, no tests, no data run, no network fetch, no Pass 1 execution, no Pass 2, no S2 build, no backfill, no scoring, no probe, no gate change. `named_binary_probe_blocked` observed `true`, not flipped. Context read from the public mirror per the pointer (root START_HERE, project_context START_HERE, GUARDRAILS, PROJECT_STATE, DECISION_LOG, PRICE_INPUT_CONTRACT, SPEC_price_source_s1_coverage, HANDOFF_s1_pass1_RESULT).

## Correction applied (this revision)

Per the orchestrator decision, the precision-loss handling was scoped correctly:
- **`STOP_PRECISION_LOSS` and the precision-loss failure mode now apply to identifiers only** — `token_id`, `outcome_index`, and other integer-like identifiers where digit-exact identity is required. Scientific-notation formatting of an ordinary numeric `price` is **no longer implied to be precision loss**.
- **`price` is validated solely as a finite numeric value in `[0,1]`;** scientific-notation formatting of a price is accepted. Non-finite or out-of-range prices are per-row `PRINT_PRICE_INVALID` (counted), not a hard stop.
- Edits landed in §2 (price contract row), §6.4 (renamed "Identifier precision loss"), and §8.5 (stop condition). The `token_id` contract still fails loud on scientific notation, as that is an identifier. No other guardrail changed.

## Authorization status (explicit)

**Option A S1-ALT Pass 1 implementation remains UNAUTHORIZED** unless the user separately authorizes it in chat. Spec approval is not implementation authorization; delivery/commit of these files is not authorization. A future Pass 1 implementation task (tests-first, local-only) requires its own explicit in-chat go-ahead. Pass 2, Option B (network), any S2-analogue build, P1/P2/P3, and the probe all likewise remain unauthorized and separately gated.

## What the draft proposes

A coverage-only feasibility stage, **S1-ALT**, for the next candidate per-side price source after the accepted S1 negative (`S1_SOURCE_NOT_VIABLE`; UP_DOWN 0.38 / OVER_UNDER ≈ 0.5204 / NAMED_OTHER 0.65 vs the 0.95 Level-B bar — not re-litigated).

- **Primary candidate = Option A, local `Store.load_trades()` prints.** The trades parquet already carries `token_id` + `outcome_index` + `price` + `traded_at` per print (proven in the accepted S0 contract), so per-token price observations exist locally; what is missing is evidence they land inside the decision window for **both** side tokens. Pass 1 is local-only — no network at all.
- **Option B (Data API `/trades`) is gap-fill only,** separately authorized, user-run, with a mandatory API-vs-local equivalence check. **Option C (on-chain OrderFilled/OrdersMatched reconstruction) is listed but excluded by guardrail.**
- **Decision-price derivation without synthesis:** per-token isolation — side *k*'s observation is the first print of side *k*'s own token in `[first_trade_ts + 3600 s, resolved_at)`. No `yes_price`, no `1 − p`, no complement-fill, no use of `side` or labels to construct a price; identity selection only. S1-ALT persists no prices — statuses, counts, gap diagnostics only.
- **Test design mirrors S1:** Levels A/B/C, Level B binding, same pre-registered 0.95 per-subclass threshold, invalid-window exclusion, all-one-status guard, typed `S1ALT_*` verdicts and `STOP_*` halts. Pass 1 proposes reusing the **same 300-condition S1 sample** for per-condition comparability (with `STOP_SAMPLE_IRREPRODUCIBLE` if the manifest can't be reproduced). Pass 2 remains separately gated even though local.
- **Named failure modes:** sparse prints, one-side-only prints (flagged as the most plausible negative), stale first-in-window print / gap distributions (measured, not tolerated by fiat), token-id precision loss, API/local mismatch, keyless prints (~0.4%), unstable pairs, dedup conflicts.
- **P1 consequence made explicit:** no S1-ALT outcome unblocks P1. Even `S1ALT_SOURCE_VIABLE` only qualifies the candidate for a separately authorized build spec → build + audit gate → separately authorized P1 resume. Every branch keeps the probe unauthorized.

## Orchestrator decision (resolved)

- **Spec text: APPROVED — SPEC ONLY.** The one required correction (precision-loss scope) is applied.

## Resolved this revision

**Same-sample reuse is approved** for Option A S1-ALT Pass 1, for direct comparability with S1 Pass 1. If the exact S1 Pass-1 sample cannot be reconstructed from accepted artifacts, the run must halt with `STOP_SAMPLE_IRREPRODUCIBLE`; do not improvise a replacement sample inside the run.

## Still open (not decided by this approval)

1. Option B (Data API `/trades`) remains unauthorized until after a Pass 1 handoff, per spec.
2. Whether to separately authorize an Option A S1-ALT Pass 1 implementation task (tests-first, local-only) — **not assumed; requires explicit in-chat authorization.**

## Guardrails

Research only. Approval is SPEC ONLY. No implementation, no tests, no data run, no network, no Pass 1 execution, no Pass 2, no S2-analogue build, no P1/P2/P3 continuation, no probe, no scoring, no backfill, no wallet/OrdersMatched/`log_index`/PnL, no gate change, no `yes_price`/`1 − price`/`1 − yes_price` synthesis anywhere. `named_binary_probe_blocked` stays `true`. Option A Pass 1 implementation is unauthorized absent separate in-chat authorization. Delivery of these files is not authorization; the user commits the approved spec into the repo.
