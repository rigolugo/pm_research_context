# HANDOFF — Claude → Orchestrator: GitHub Bootstrap Validation

**Task:** Bootstrap validation of the canonical GitHub repo. Read-only. No implementation.
**Outcome:** `BOOTSTRAP VERIFIED`
**Date:** 2026-07-02

---

## Session record

- **GitHub repo used:** `rigolugo/pm_research` (canonical source of truth).
- **Commit observed:** `5dac606110f01b326878777a9876ede78d6fced8`
  ("Initial pm_research governance and source snapshot", 2026-07-02 01:26 UTC).
  Single-commit history; re-confirmed unchanged during this session.
- **Native GitHub connector:** not available/used (connector registry showed no
  GitHub entry). Access was a plain read-only clone over the allowlisted
  `github.com` domain.

## Access path

- **Temporary public, token-free, read-only.** The repo was made public by the
  user solely as a credential-free read workaround; treated as an access path
  only, never as authorization to implement or run anything.
- **No PAT used.** No credentials of any kind entered the sandbox or this chat.

## Guardrail attestation (what did NOT happen)

- No code was run.
- No data was fetched or inspected.
- No implementation, probe, scoring, wallet work, PnL, OrdersMatched expansion,
  `log_index`, paper/live trading, or full-indexer work occurred.
- No files modified, no commits, no tests, no gate changes.
- `named_binary_probe_blocked` observed `true`, not flipped. No file in the repo
  sets it `false` or sets `probe_execution_authorized = true` (verified by
  read-only grep; empty result).

## Confirmed state (verified against GitHub, not memory)

- **P0 accepted / `P0_CLEAR`** — pooled counts: contract_eligible 39,957 /
  resolved_single_winner 39,693 / ambiguous 253 / source_rows 39,946 /
  missing 11 / final_p0_eligible 39,693.
- **P1 (feature assembly) BLOCKED** on missing per-side / token-identity price
  input. `OrientationContract.canonical_side_price(side_0_price, side_1_price)`
  needs two per-side prices; `Store.load_prices()` returns only
  `[condition_id, ts, yes_price]`. `yes_price` is proven YES/NO-only and unsafe
  for UP_DOWN / OVER_UNDER / NAMED_OTHER; no local per-side/per-token price
  series exists.
- **P2 / P3 / probe execution UNAUTHORIZED.**
- **Chat2 Dune wallet-cohort discovery BLOCKED** (distinct phase; outcome-source
  scoreability does not unblock it).
- **Next possible task, only if explicitly authorized:** a SPEC-ONLY price-source
  build plan for a per-side / token-identity-keyed decision-time price series.
  No implementation, data run, backfill, scoring, or probe.

## Bootstrap chain read (canonical order, from repo)

Root `START_HERE.md` → `project_context/START_HERE.md` (Rule 0 + required read
order + current branch state) → all 9 required-read-order files present and
mutually consistent, plus supporting `DUNE_DATA_NOTES.md` and
`CLAUDE_PROJECT_SETTINGS.md`. All five expected-state claims verified.

## Minor doc-hygiene findings (non-blocking; no action taken)

1. `ARTIFACT_INDEX.md` should eventually state explicitly that **GitHub is
   canonical** and that any pinned Claude Project Files are **cache/mirror
   copies only** (they lose to GitHub on conflict).
2. `ARTIFACT_INDEX.md` should eventually add a **"Phase-1 legacy modules —
   closed / not active"** section covering the unindexed modules and scripts
   (`backtest.py`, `wallet_classifier.py`, `lifecycle_pnl.py`,
   `walk_forward_backtest.py`, `exit_mirror_backtest.py`, `consensus.py`,
   `latency.py`, `selection.py`, and related scripts). Their docstrings identify
   them as Phase-1 tooling tied to the closed NO-GO findings; they are not live
   surface and not authorization for anything.
3. `START_HERE.md` should optionally add **`WORKFLOW.md`** as a supporting
   discipline item or read-order entry (it carries the "verify schemas against
   DATA_CONTRACTS before any implementation stage; else schema-inspection only"
   rule).
4. `Archive/SPEC_price_source_s1_coverage.md` **exists** (approved SPEC-ONLY, S1
   CLOB coverage). It remains **archived / non-authoritative** unless a current
   source-of-truth file explicitly promotes it. Not promoted here.

## Naming note

Role name is **Orchestrator** going forward. "Copilot" retained only in historical
archived material where it already appears (e.g. `COPILOT_HANDOFF_2026-06-29.md`,
where M365 Copilot is named as the prior orchestrator).
