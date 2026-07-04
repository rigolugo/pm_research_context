# pm_research_context
Public Claude-readable context mirror for the Polymarket Research System

## Purpose

This repository exists only so Claude can read accepted project context without credentials.

The canonical source of truth is the private GitHub repository:

`rigolugo/pm_research`

This public repository is a mirror of selected accepted context files only.

If anything in this public mirror conflicts with the private canonical repo, the private repo wins.

## What this repo contains

Safe, accepted project context such as:

- `START_HERE.md`
- `project_context/START_HERE.md`
- guardrails
- project state
- decision log
- closed findings
- artifact index
- accepted data contracts
- accepted price/input contracts
- accepted specs
- accepted handoffs
- workflow/context-sync instructions

## What this repo must NOT contain

Do not commit:

- raw data
- generated artifacts
- parquet files
- CSV exports
- logs
- wallet lists
- API keys
- credentials
- `.env` files
- private keys
- Dune exports
- large output folders
- local runtime files
- code unless explicitly approved for public mirroring

## Bootstrap for Claude

Read in this order:

1. `START_HERE.md`
2. `project_context/START_HERE.md`
3. The required read order listed in `project_context/START_HERE.md`

Do not rely on old chats, memory, uploaded duplicate files, or archived files.

## Current standing state

At the time this mirror was created:

- P0 is accepted / `P0_CLEAR`.
- P1 is blocked on missing per-side / token-identity price input.
- P2 / P3 / probe execution are unauthorized.
- Chat2 Dune wallet-cohort discovery is blocked.
- The next possible task, only if explicitly authorized, is a SPEC-ONLY price-source build plan.

This snapshot is only informational. Always defer to `project_context/START_HERE.md` in the canonical private repo, and to the latest accepted mirror sync here.

## Guardrail reminder

This is a research project, not a trading product.

No live trading.  
No paper trading.  
No wallet-copying.  
No full indexer.  
No unauthorized `log_index` work.  
No unauthorized PnL-by-role work.  
No wallet discovery.  
No scoring.  
No P1/P2/P3 continuation.  
No probe execution.  
No use of `yes_price`, `1 - price`, or `1 - yes_price` to unblock named-binary pricing.

This mirror authorizes no implementation. It is context only.
