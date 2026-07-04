# GUARDRAILS

*Hard constraints for every chat in this project. Read these first, every message.*

## This is a RESEARCH project — not a product

Pessimistic, falsification-minded research on Polymarket market structure. The goal is to test hypotheses and falsify them cleanly, not to build or run anything that trades. Despite any project title, this is **research only**.

## Absolute constraints

- **No live trading.**
- **No paper trading.**
- **No wallet-copying** (no copy-trade logic, no follower/aggressor execution, no reviving the Phase-1 copy thesis).
- **No full indexer.**
- **No `log_index` backfill** unless **explicitly authorized** in-chat.
- **No PnL-by-role analysis** unless **explicitly authorized** in-chat (it is a separate, larger phase — see DECISION_LOG).
- **No named-binary testing** until the yes_price semantics rewrite is complete and audit-gated.
- **No wallet-cohort claims without a holdout.** No PnL-based selection on the discovery set.

## Working discipline

- **Verify before concluding.** Check decoders/claims against an authoritative external source (Dune exact-index match, Polygonscan) before stating a conclusion. Never conclude from a single row or from an all-one-role output without a validation pass on a known case.
- **Spec before implementation.** Produce an implementation spec and get it reviewed before writing code for a new branch.
- **Never silently reverse a prior decision.** If reversing, say so explicitly and show the evidence.
- **Close tasks with a Claude-to-Orchestrator handoff memo.**

## Environment reminder

Work runs locally on Windows/Miniconda. Claude delivers files for the user to copy into `C:\b1\pm_research`; Claude does not assume network/pytest/keccak in its own sandbox. See PROJECT_STATE.md §Environment.
