# CLAUDE PROJECT SETTINGS

Purpose:
Record Claude project capability settings so new Claude and ChatGPT chats do not infer or re-ask them.

Status:
Operational settings only. These settings do not change research status, do not authorize implementation, and do not override GUARDRAILS.md, PROJECT_STATE.md, DECISION_LOG.md, CLOSED_FINDINGS.md, ARTIFACT_INDEX.md, DATA_CONTRACTS_*.md, or active SPEC_*.md.

## Network access

Claude network egress may be enabled, but current allowlist should remain narrow.

Current recommendation:
- Domain allowlist: Package managers only.
- Additional allowed domains: none.
- Do not add wildcard domains.
- Add external domains only one exact domain at a time, for a named task, with the reason stated in the Claude prompt.

Network access is allowed only for bounded, read-only reference checks. It does not authorize:
- Dune/Polymarket data fetching by Claude,
- probe execution,
- P1/P2/P3 continuation,
- wallet discovery,
- PnL analysis,
- paper/live trading,
- log_index work,
- full indexer work,
- dependency additions without explicit approval.

Dune API/key work remains local/user-run unless explicitly authorized. Follow DUNE_DATA_NOTES.md precision and API rules.

## Skills

Enabled / recommended:
- doc-coauthoring — use for project documentation, specs, handoffs, documentation locks, and reader-testing.

Available only on demand:
- learn — use only when the user asks Claude to explain a concept or code path. Do not use it for implementation, artifact review, or decision tasks.

Disabled / not recommended:
- mcp-builder — disable unless the user explicitly authorizes building an MCP server.
- skill-creator — leave off unless explicitly creating a custom project skill.
- web-artifacts-builder, canvas-design, theme-factory, brand-guidelines, internal-comms, algorithmic-art, slack-gif-creator — not relevant to current project work.

## Current project implication

These settings do not unblock the project.

Current accepted status:
- P0 preflight accepted / P0_CLEAR.
- P1 remains blocked on price input.
- Local prices have only condition_id, ts, yes_price.
- yes_price is unsafe for non-YES/NO named binaries.
- No local per-side/per-token price artifact exists.
- Next legitimate work, only if explicitly requested, is SPEC ONLY for a valid per-side price-source build.
