# COPILOT_HANDOFF_2026-06-29

**Audience:** New project orchestrator / reviewer.  
**Project:** Polymarket Research System.  
**Repository root:** `C:\b1\pm_research` on the Windows/Miniconda laptop.  
**Date:** 2026-06-29.  
**Implementation agent:** Claude Opus 4.8 High.  
**Prior orchestrator:** M365 Copilot.

> This handoff is written for an assistant taking over orchestration and critical review. The new assistant should not assume Claude’s outputs are correct merely because code exists or tests pass. The new assistant should require artifacts, commands, metrics, and falsification checks before accepting results.

---

## 1. Executive Summary

This project is a Polymarket-native research system intended to test, falsify, or refine hypotheses about whether information or execution patterns around Polymarket wallets and markets can produce a replicable edge. The project has deliberately pessimistic research culture: a negative, blocked, or instrument-failure result is considered useful if it is well validated.

The original wallet-copying thesis has been substantially weakened. Phase 1 tested delayed taker-style copy-trading of volume-ranked wallets and produced a NO-GO result. This finding was later qualified because the earlier dataset was silently OOM-truncated to roughly 388k trades, while the corrected re-backfill produced about 16.5M trades. The structural conclusion remains important, but exact Phase 1 numeric claims on the old dataset should not be treated as final without revalidation.

A large data-engineering and validation phase corrected the trade dataset. A re-backfill populated on-chain join keys (`tx_hash`, `token_id`, `outcome_index`) with about 99.6% coverage. Earlier analysis had used only roughly 2–3% of the now-available trade data. The audit tooling was rewritten to stream parquet files to avoid OOM, and execution moved from an AWS `t3.micro` to a local Windows/Miniconda laptop.

Rank 1A tested whether a simple price-only forecast could beat Polymarket market price on pooled YES/NO binary markets. The data gate eventually cleared after fixing null `condition_id`, deduplication, and concentration-gate issues. The actual Rank 1A probe was negative: Rung 0 sanity tied baseline, Rung 1 isotonic recalibration improved Brier in only 1/5 splits and was not tradable. A price-bucket diagnostic returned `NO_ACTIONABLE_BUCKET`. This falsifies broad price-only recalibration on the tested pooled YES/NO universe.

Rank 2 investigated whether on-chain maker/taker or economic-role reconstruction could explain wallet edge. Several false interpretations were caught and corrected: `OrderFilled` topic/order confusion, Dune exact-event mismatches, CSV precision loss of 78-digit token IDs, and all-one-role artifacts. The final validated instrument is `OrdersMatched`, with `takerordermaker` as the economic taker/aggressor. Expanded role recovery classified 117 trades across 96 wallets with 100% role recoverability and 0% ambiguity. The distribution was stable mixed / taker-leaning: 71 taker and 46 maker, with wallet-level bimodality. The original “wallets are mostly makers/liquidity providers” H1 is not supported as a population claim.

Rank 2 role recovery is now closed as answered. PnL-by-role, deeper per-wallet sampling, `log_index` work, and full indexer work are **not authorized**. They require explicit user approval as a new phase. The current preferred branch is **named-binary unlock via the `yes_price` / `canonical_side_price` rewrite**, with a parallel spec-only branch for Dune wallet discovery.

Two new Claude Project chats exist under a project named **Polymarket Research**. `Chat1 — Named-Binary yes_price Rewrite` owns named-binary semantics and is implementing the semantics/audit layer. `Chat2 — Dune Wallet Discovery for Named-Binary` is spec-only and must consume Chat1’s `named_binary_classification_contract` rather than define named-binary independently.

The immediate next focus is Chat1 validation: run tests, run the label-pair census, review/hand-classify the census head if needed, then run `audit_named_binary_semantics.py` and inspect the gate JSON. The most recent local precision check already confirmed `token_id` reads as `str` from trade parquet shards, eliminating the highest-priority precision risk before audit.

---

## 2. Project Context

### Project name

`Polymarket Research System`.

### Domain/problem area

Quantitative research on Polymarket prediction-market trades, wallets, market structure, binary/named-binary semantics, and wallet economic-role analysis.

### User goal

The user wants to determine whether a Polymarket-native research line can identify a replicable edge or falsify weak theses quickly and rigorously. The project should not drift into generic trading-bot building, live trading, or copy-trading revival.

### What success looks like

Success is not necessarily a positive trading signal. Success may be:

- a clean, reproducible negative result;
- a validated data foundation;
- a clear audit gate;
- an instrument being proven wrong;
- a hypothesis being sharpened or killed;
- a new branch being justified by evidence.

A positive result must survive out-of-sample splits, cost assumptions, leakage checks, semantic audits, and role/market-orientation validation.

### What failure/falsification means

Failure is useful if it is specific. Examples already accepted:

- delayed wallet-copying as taker is a NO-GO;
- pooled YES/NO price-only recalibration is negative;
- price-bucket diagnostics found no actionable bucket;
- original maker-dominated wallet H1 is unsupported.

### Current phase

Current phase: **Named-binary semantics rewrite and audit gate**. The current active implementation branch is Chat1. Chat2 is spec-only Dune wallet discovery and must wait for Chat1 artifacts.

### Constraints

- Research only.
- No live trading.
- No paper trading unless explicitly authorized later.
- No wallet-copying revival.
- No PnL-by-role unless explicitly authorized.
- No `log_index` work unless explicitly authorized.
- No full indexer unless explicitly authorized.
- Local execution on Windows/Miniconda, no admin assumptions.
- Heavy jobs must stream/chunk parquet data; do not load all 16.5M trades into a single DataFrame.
- Dune outputs must preserve 78-digit token IDs as strings.

### Important user preferences

- User wants concise but rigorous orchestration.
- User expects Claude to code and Copilot/new assistant to review and challenge.
- User values negative results.
- User prefers explicit gates and “do not proceed until X is proven.”
- User often pastes Claude handoffs and wants direct approval/rejection plus next prompt.

---

## 3. Role Architecture

### User

The user owns:

- final authorization to proceed to new phases;
- providing local command outputs;
- running scripts on the Windows laptop;
- running Dune queries / Dune API fetches when needed;
- moving files into the repository;
- deciding whether to redirect, stop, or fund a larger phase.

Decisions requiring user approval:

- implementation starts after spec/planning;
- named-binary probe after audit gate;
- Chat2 implementation;
- PnL-by-role;
- `log_index` backfill;
- full indexer;
- live/paper trading.

The user expects the orchestrator to challenge Claude, not rubber-stamp.

### Copilot / prior orchestrator

The prior orchestrator:

- kept Claude scoped to implementation;
- made gate decisions;
- reframed false positives as artifacts when evidence showed bugs;
- required artifact-level validation;
- separated data gates from strategy probes;
- insisted on not over-reading small samples;
- created project-chat structure: `Claude-old`, `Chat1`, `Chat2`.

If the prior orchestrator stayed, the immediate focus would be:

1. Validate Chat1 tests/census/audit.
2. Do not allow Chat2 implementation until Chat1 contract and gate exist.
3. Review named-binary audit gate JSON.
4. Decide whether named-binary semantics are clear, warning, or blocked.

### New Assistant / Orchestrator

The new assistant is not primarily the programmer. The new assistant must:

- review project state;
- challenge Claude’s logic;
- inspect metrics and artifacts;
- write precise prompts to Claude;
- decide whether work is valid, blocked, or cosmetic;
- maintain project memory;
- protect guardrails;
- never accept results solely because Claude claims tests pass.

### Claude Opus 4.8 High

Claude is the implementation agent. Claude has:

- written and revised audit scripts;
- implemented Rank 1A probe;
- implemented OrderFilled/OrdersMatched diagnostics;
- implemented the named-binary semantics layer and audit scripts;
- produced handoffs and implementation plans.

Claude should continue coding only within authorized scope. Claude should not:

- start probes without gates;
- implement wallet discovery before Chat1 artifacts;
- run PnL-by-role;
- start log_index/full indexer;
- revive wallet-copying;
- make strategy claims.

Claude outputs should be reviewed by requiring:

```text
1. Files changed
2. Summary of changes
3. Commands run
4. Full command output or relevant excerpts
5. New artifacts created
6. Metrics before vs after
7. Tests passed/failed
8. Known remaining issues
9. Any assumptions made
10. Recommended next step
```

---

## 4. Orchestrator Role and Review Standard

The orchestrator should require evidence for every claim.

For each important claim, ask:

```text
What is the claim?
What exact artifact supports it?
What command produced that artifact?
What data went into it?
What assumptions does it rely on?
What could make it false?
Has it been independently validated?
What should Claude prove next?
```

### How to challenge Claude

Claude has been strong but has made major self-corrected errors. The orchestrator must therefore:

- treat all all-one-class outputs as suspicious until opposite-class detection is validated;
- require exact log-index matching when comparing Dune vs local RPC events;
- require large integer fields to be strings, not floats;
- require independent schema checks before assuming field names;
- require sample-size and coverage gates before interpreting distributions;
- ensure metrics align with the unit of analysis.

### Detecting false progress

False progress examples seen in this project:

- `PASS_BINARY_ONLY` before proving suspicious rows did not overlap usable binary.
- `BLOCKED_BY_PROBE_OVERLAP` caused by raw-streaming path bypassing hygiene.
- `BLOCKED_BY_CONCENTRATION` caused by row-level concentration metric applied to condition-level probe.
- `217 maker / 0 taker` caused by join/decoder artifact.
- `10 taker / 0 maker` caused by Dune CSV precision loss.

A metric is trustworthy only after:

- schema is verified;
- units match the probe;
- rows are not duplicated or misjoined;
- enough sample coverage exists;
- artifacts are generated and reproducible;
- opposite-case validation passes.

### Bias/leakage checks

Always inspect:

- discovery vs holdout separation;
- no use of `winning_outcome` or resolution in selection/classification unless explicitly audited as non-leaking;
- decision timestamps are ex-ante;
- price convergence is corroboration only, not a resolution source;
- wallet cohorts are not selected on full-period PnL;
- named-binary classification comes from Chat1 contract, not Dune ad hoc labels.

---

## 5. Instructions for Managing Claude Opus 4.8 High

### Claude’s current responsibility

Chat1: named-binary semantics implementation and audit gate.

Recent Claude deliverables:

- `named_binary_semantics_spec(2).md`
- `named_binary_semantics_implementation_plan.md`
- `HANDOFF_named_binary_semantics.md`
- `ADDENDUM_schema_corrections.md`
- new semantics modules and scripts.

### What Claude implemented recently

Per handoff:

- `pm_research/semantics/__init__.py`
- `pm_research/semantics/lexicon.py`
- `pm_research/semantics/mapping_audit.py`
- `pm_research/semantics/resolution_schema.py`
- `pm_research/semantics/named_binary.py`
- `scripts/audit_named_binary_semantics.py`
- `scripts/named_binary_label_census.py`
- `tests/test_named_binary_semantics.py`

The schema correction addendum modified only the two scripts and left core semantics modules unchanged.

### What Claude should not touch without approval

- Rank 1A scripts.
- Rank 2 scripts.
- `schemas.py`.
- `store.py`, unless adding `iter_trades` is explicitly approved. Current recommendation: use pyarrow fallback first; do not add `iter_trades` yet.
- PnL-by-role.
- `log_index`.
- full indexer.
- wallet discovery implementation.
- named-binary probe.

### Prompt style that worked

Prompts that worked:

- explicitly state guardrails;
- require Claude-to-Copilot handoff format;
- require exact commands and paste-back fields;
- specify gate states and decision labels;
- ask for spec first, implementation later;
- ask Claude to not interpret results until validation passes.

Prompts that can produce problems:

- broad “continue” prompts;
- prompts that combine strategy, implementation, and analysis;
- prompts allowing Claude to define semantics in multiple branches;
- prompts failing to specify “no probe / no PnL / no trading.”

---

## 6. Repository Overview

Known repository root:

```text
C:\b1\pm_research
```

Inferred/current structure:

```text
C:\b1\pm_research\
  pm_research\
    data\
      store.py                         # existing data-load helpers; optional iter_trades not yet required
      schemas.py                       # referenced, not modified for named-binary rewrite
    semantics\                         # new package from Chat1
      __init__.py
      lexicon.py
      mapping_audit.py
      resolution_schema.py
      named_binary.py
    ...
  scripts\
    audit_market_structure.py           # prior streaming audit
    probe_fast_gate.py                  # Rank 1A gate
    forecast_vs_price.py                # Rank 1A probe
    rank1a_price_bucket_diagnostic.py   # Rank 1A diagnostic
    orderfilled_sample_join.py          # Rank 2 sample join
    orderfilled_semantic_validation.py  # Rank 2 semantic validation, if present
    ordersmatched_economic_role.py      # Rank 2 OrdersMatched probe
    ordersmatched_economic_role_expanded.py
    ordersmatched_maker_pairing_validation.py
    audit_named_binary_semantics.py     # new Chat1 script
    named_binary_label_census.py        # new Chat1 script
  tests\
    test_audit_market_structure.py
    test_orderfilled_sample_join.py
    test_ordersmatched_economic_role.py
    test_named_binary_semantics.py      # new Chat1 tests
  artifacts\
    probe_fast_gate.json / .md / csvs
    forecast_vs_price_rank1a.*
    rank1a_price_bucket_diagnostic.*
    ordersmatched_economic_role_expanded.*
    ordersmatched_maker_pairing_validation.*
    named_binary_*                      # expected after Chat1 audit
```

Important data root:

```text
C:\b1\data
  trades\*.parquet                     # per-wallet shards
  markets.parquet                       # condition_id, category, liquidity_usdc, created_at, resolution_at
  resolutions.parquet                   # condition_id, winning_outcome, resolved_at
  prices\*.parquet                      # schema needs verification if used by semantics orientation
```

Canonical files:

- `PROJECT_STATE.md`, `GUARDRAILS.md`, `DECISION_LOG.md`, `CLOSED_FINDINGS.md`, `ARTIFACT_INDEX.md` in the Claude Project.
- `DUNE_DATA_NOTES.md` as supporting reference only.
- `named_binary_semantics_spec(2).md` and `named_binary_semantics_implementation_plan.md` for Chat1.

Experimental/obsolete:

- Any intermediate conclusion that `OrderFilled` maker/taker alone is economic role.
- Any 217/0 or 10/0 role distribution artifact.
- Fee-field diagnostic as primary route.
- Old Rank 1A provisional gates before hygiene fixes.

---

## 7. Environment and Setup

### OS and environment

- Windows laptop.
- Miniconda.
- Environment name: `pmresearch`.
- No admin assumptions.
- No WSL/Docker assumption.

### Activation

```powershell
cd C:\b1\pm_research
conda activate pmresearch
$env:PYTHONPATH="C:\b1\pm_research"
```

### Known dependencies

Installed/used:

```powershell
conda install -c conda-forge pandas pyarrow certifi pytest numpy scipy scikit-learn matplotlib -y
conda install -c conda-forge pycryptodome -y
```

`scikit-learn` is the correct package name, not `scikit`.

### Dune

Dune API key should be environment variable only:

```powershell
$env:DUNE_API_KEY="<secret>"
```

Never hardcode secrets.

### Important precision constraint

`token_id` and Dune asset IDs are 78-digit strings. They must not become floats.

Verified locally:

```text
token_id dtype = str
example token_id preserved as full 78-digit string
```

Command used:

```powershell
python -c "import pandas as pd, glob; f=glob.glob(r'C:\b1\data\trades\*.parquet')[0]; df=pd.read_parquet(f, columns=['token_id']); print('file:', f); print(df['token_id'].dtype, type(df['token_id'].iloc[0]), df['token_id'].iloc[0])"
```

Output:

```text
file: C:\b1\data\trades\0x000c8cb93bb5fd4c9f292c6b4c5f7c021bb6bcf3.parquet
str <class 'str'> 54598622966860754711335168974270562407018040578529185444239033642682947522004
```

---

## 8. How to Run the Project

### Run unit tests for named-binary semantics

```powershell
cd C:\b1\pm_research
conda activate pmresearch
$env:PYTHONPATH="C:\b1\pm_research"
python -m pytest tests\test_named_binary_semantics.py -q
```

Expected: `34 passed`. If import/path errors occur, fix placement or PYTHONPATH before audit.

### Run named-binary label-pair census

```powershell
python scripts\named_binary_label_census.py --root C:\b1\data --out-dir artifacts
```

Expected artifact:

```text
artifacts\named_binary_label_pair_census.csv
```

Inspect:

```powershell
python -c "import pandas as pd; df=pd.read_csv(r'C:\b1\pm_research\artifacts\named_binary_label_pair_census.csv'); print(df.head(50).to_string())"
```

Purpose: frequency-ranked label-pair census for human review / lexicon refinement. Do not treat seed lexicon as final if census head reveals important patterns.

### Run named-binary audit

```powershell
python scripts\audit_named_binary_semantics.py --root C:\b1\data --out-dir artifacts
```

Optional version assertion:

```powershell
python scripts\audit_named_binary_semantics.py --root C:\b1\data --out-dir artifacts --lexicon-version nb-contract-2026-06-28.1
```

Expected artifacts:

```text
artifacts\named_binary_classification_contract.json
artifacts\named_binary_classification_contract.md
artifacts\named_binary_audit_gate.json
artifacts\named_binary_audit_gate.md
artifacts\named_binary_semantics_report.md
```

Extract paste-back fields:

```powershell
python -c "import json; d=json.load(open(r'C:\b1\pm_research\artifacts\named_binary_audit_gate.json')); print({k:d.get(k) for k in ['gate_state','total_conditions','named_binary_eligible','usable_named_binary_conditions','token_id_coverage','outcome_index_coverage','resolution_mapping_success_rate','orientation_correctness_rate','blocked_reason_counts','nb_contract_version']})"
```

### Rank 1A commands, for reference only

Do not rerun unless explicitly needed.

```powershell
python scripts\probe_fast_gate.py --root C:\b1\data --splits 5 --robust-pass-splits 4 --min-test-signals-per-split 50 --min-train-signals-per-split 100 --out-dir artifacts
```

```powershell
python scripts\forecast_vs_price.py --root C:\b1\data --decision-policy first_price_after_warmup --warmup-hours 1 --splits 5 --robust-pass-splits 4 --out-dir artifacts
```

### Rank 2 commands, for reference only

Do not continue Rank 2 without new authorization.

```powershell
python scripts\ordersmatched_economic_role_expanded.py --root C:\b1\data --n-tx 150 --out-dir artifacts
```

Dune API fetch pattern:

```powershell
curl.exe -H "x-dune-api-key: $env:DUNE_API_KEY" "https://api.dune.com/api/v1/query/<QUERY_ID>/results/csv?limit=5000" -o C:\b1\csv\dune_ordersmatched_expanded.csv
```

---

## 9. Data Sources and Data Model

### Local trade shards

Path:

```text
C:\b1\data\trades\*.parquet
```

Columns confirmed by Claude addendum:

```text
trade_id, wallet, condition_id, outcome, side, price, size_usdc, traded_at, tx_hash, token_id, outcome_index
```

Notes:

- One parquet shard per wallet.
- `outcome` is the outcome label and is anchored to the trade row identity.
- `token_id` is a 78-digit string.
- `outcome_index` may be float typed like `1.0`; code casts to `int(idx)`.
- Analysis load should drop null/blank `condition_id` rows.
- Dedup should prefer rows with populated semantic keys.

### markets.parquet

Path:

```text
C:\b1\data\markets.parquet
```

Confirmed columns:

```text
condition_id, category, liquidity_usdc, created_at, resolution_at
```

Important: this file does **not** contain outcome labels or token ordering.

### resolutions.parquet

Path:

```text
C:\b1\data\resolutions.parquet
```

Known columns:

```text
condition_id, winning_outcome, resolved_at
```

Claude addendum says `winning_outcome` is a label string. Needs audit confirmation on real run.

### Price data

Path:

```text
C:\b1\data\prices\*.parquet
```

Schema currently **Needs verification** before relying on orientation/time series logic.

### Dune

Supporting source for:

- OrdersMatched tables;
- OrderFilled tables;
- market activity via `polymarket_polygon.market_trades` if schema-inspected;
- wallet cohort discovery later.

Dune precision rule: cast all uint256-like fields to `varchar`.

---

## 10. Core Logic and Algorithms

### Rank 1A: condition-level price recalibration

Unit: one signal per condition. Decision policy: first price after 1-hour warmup. Rung 0 forecast = market price sanity. Rung 1 = isotonic recalibration of price fit on train only. Metrics: Brier score, Brier skill, EV after fees/spread.

Result: negative.

### Rank 2: OrdersMatched economic role

Core rule:

```text
wallet == takerordermaker -> economic taker/aggressor confirmed
wallet != takerordermaker AND wallet appears in paired OrderFilled context -> economic maker confirmed
```

Never infer maker by exclusion.

Expanded result: 71 taker, 46 maker, 117 trades, 96 wallets, 100% recoverable, 0 ambiguity. Role recovery complete. No PnL-by-role.

### Named-binary semantics rewrite

Goal: replace label-derived `yes_price` with identity-derived orientation.

General primitive:

```text
canonical_side_price / oriented_price
```

`yes_price` is only a YES/NO alias.

Subclasses:

```text
YES_NO
OVER_UNDER
UP_DOWN
TEAM_VS_TEAM
NAMED_OTHER
UNUSABLE
```

Mapping audit must prove:

- token_id ↔ condition_id stability;
- token_id ↔ outcome_index stability;
- outcome_index ↔ label stability;
- exactly two tokens;
- resolution maps to exactly one token.

Ambiguity means exclusion.

---

## 11. Important Results So Far

### Confirmed findings

| Finding | Evidence | Status |
|---|---|---|
| Corrected full trade dataset is about 16.5M trades vs older ~388k truncated dataset | Re-backfill/audit conversation and artifacts | Trusted at project level, but exact artifact path should be checked |
| Key coverage after re-backfill around 99.6% for tx_hash/token_id/outcome_index | Market-structure audit output | Trusted for prior gates |
| Rank 1A data gate cleared with warnings | `probe_fast_gate.json`, final verdict `CLEAR_WITH_WARNINGS_FOR_RANK1A` | Trusted for Rank 1A history |
| Rank 1A Rung 1 negative | `forecast_vs_price_rank1a.json`; 1/5 positive Brier splits; not tradable | Trusted |
| Price-bucket diagnostic `NO_ACTIONABLE_BUCKET` | `rank1a_price_bucket_diagnostic.json` | Trusted |
| OrdersMatched economic role recoverable | `ordersmatched_economic_role_expanded.json`; 117 rows, 100% recoverable, 0 ambiguity | Trusted for role recovery |
| Original maker-dominated H1 not supported | Expanded 71 taker / 46 maker distribution | Trusted as population role-distribution finding, not PnL finding |
| token_id local precision reads as string | Local dtype command output | Trusted |

### Provisional findings

| Finding | Why provisional |
|---|---|
| Named-binary semantics implementation works on full data | Tests not yet run locally; audit not yet run |
| Named-binary usable universe size | Unknown until audit |
| Chat2 Dune wallet discovery feasibility | Spec-only; depends on Chat1 artifacts and Dune schema inspection |
| PnL-by-role relevance | Not authorized/tested |

### Failed experiments / negative results

- Phase 1 delayed taker copy-trading: NO-GO.
- Rank 1A pooled YES/NO price recalibration: negative.
- Rank 1A price-bucket diagnostic: no actionable bucket.
- OrderFilled-only event role: insufficient for economic role.
- Fee-field role diagnostic: inconclusive and superseded.

### Misleading or obsolete results

- `217 maker / 0 taker` OrderFilled result: artifact.
- `10 taker / 0 maker` OrdersMatched result before asset-ID fix: artifact.
- Dune UI CSV with scientific notation: invalid.
- Any label-only `yes_price` inversion logic: obsolete.

---

## 12. Key Decisions Already Made

| Decision | Rationale | Status |
|---|---|---|
| Use Windows/Miniconda local execution for heavy audits | EC2 t3.micro OOM/slow | Active |
| Stream parquet files in chunks | Avoid OOM on 16.5M rows | Active |
| Treat missing `condition_id` rows as storage-valid but analysis-invalid | Cannot map to condition safely | Active |
| Prefer non-null semantic keys on dedup | Avoid dropping populated tx_hash/token_id/outcome_index | Active |
| Rank 1A unit is condition-level | Avoid row/trade concentration bias | Closed |
| Stop Rank 1A price-recalibration line | Negative + no actionable bucket | Closed |
| Use OrdersMatched for economic role | Trade-level event with takerordermaker | Closed for role recovery |
| Do not start PnL-by-role | Larger scope, explicit approval required | Active blocker |
| Chat1 owns named-binary semantics | Avoid drift with Chat2 | Active |
| Chat2 spec-only until Chat1 contract/audit exists | Prevent premature wallet cohort logic | Active |

---

## 13. Known Bugs, Gaps, and Risks

| Issue | Severity | Location | Symptoms | Likely Cause | Current Workaround | Recommended Fix |
|---|---|---|---|---|---|---|
| Dune large integer precision loss | High | Dune CSV / pandas | `5.20896e+76`, `0.0`, no asset overlap | uint256 exported/read as float | Cast to varchar; API CSV; dtype=str | Fail loud on scientific notation |
| markets.parquet lacks labels | High | `C:\b1\data\markets.parquet` | `load_label_hints` cannot work | Metadata file minimal | Use trade-row outcome labels | Keep anchored to token_id/outcome_index |
| Named-binary audit not yet run | High | Chat1 | No gate state yet | Implementation newly integrated | Run tests/census/audit | Review gate JSON before any probe |
| Chat2 may drift into own classification | High | Chat2 | Dune-side ad hoc named-binary rules | Contract not yet available | Spec-only blocked | Require contract version equality |
| Phase 1 old quantitative results on truncated data | Medium | Phase 1 artifacts | numbers may not generalize | 388k vs 16.5M data | Treat structural only | Optional full revalidation |
| PnL-by-role not tested | Medium | Rank 2 | role distribution but no profitability | Scope deferred | Do not infer PnL | New phase if authorized |
| Price convergence leakage | High | resolution audit | winner inferred from final price | leakage risk | convergence corroboration only | Use explicit winning_outcome primary |
| Per-wallet role labels thin | Medium | Rank 2 expanded sample | many wallets n=1 | sampling breadth | population claim only | deep sample if H1′ authorized |
| Dune schema drift | Medium | Chat2 | columns missing/different | old/V2/V3 tables differ | inspect schema before union | document included/omitted |

---

## 14. Validation and Anti-Bias Checklist

| Check | Inspect | Failure | Passing evidence |
|---|---|---|---|
| Look-ahead bias | decision timestamp, resolution use | uses `resolution_at` or final price for selection | ex-ante policy; resolution separated |
| Survivorship bias | wallet selection windows | selected on full-period winners | discovery/holdout split |
| Timestamp alignment | `traded_at`, `block_time`, `resolution_at` | timezone mismatch or post-resolution data | UTC normalized; ex-ante windows |
| Duplicate trades | `trade_id`, tx_hash | duplicate fills counted twice | dedup with key completeness |
| Missing markets | condition coverage | unknown conditions silently omitted | blocked_reason_counts |
| Bad resolutions | resolution audit | winning_outcome unmappable | resolution_mapping_success_rate high |
| Leakage from final prices | resolution corroboration | convergence defines outcome | convergence corroboration only |
| Wallet filtering | saved vs Dune cohorts | overfits PnL | cohort rules exclude PnL |
| Unrealistic fills | Rank probes | assumes tradeability from Brier | EV and cost checks separate |
| API pagination | Dune/local backfill | partial results | row counts/table coverage logs |
| Floating precision | token_id/asset_id | scientific notation | dtype=str/varchar casts |
| Invalid joins | condition/token/outcome | token maps multiple conditions | mapping_audit failures |
| Sample selection bias | expanded samples | one wallet/time dominates | stratification, coverage counts |
| Overfitting | feature/rung climbing | keeps testing until positive | pre-registered gates |
| Silent failure | scripts | no artifact, stale JSON | rerun commands, timestamps |
| Market orientation | named-binary | label flips price | no-label-only-inversion test |

---

## 15. Current Task Queue

### Immediate next task

Run Chat1 local tests, census, and audit.

#### Goal

Validate named-binary semantics implementation and produce the audit gate JSON.

#### Files involved

- `tests/test_named_binary_semantics.py`
- `scripts/named_binary_label_census.py`
- `scripts/audit_named_binary_semantics.py`
- `pm_research/semantics/*`
- `C:\b1\data\trades\*.parquet`
- `C:\b1\data\resolutions.parquet`

#### Steps

1. Run tests.
2. Run label-pair census.
3. Inspect census head.
4. If needed, ask Claude to update lexicon and bump `NB_CONTRACT_VERSION`.
5. Run audit.
6. Paste gate JSON.

#### Prompt to Claude

```markdown
### Prompt to Claude

Objective:
Review the local Chat1 test/census/audit outputs for the named-binary semantics layer.

Files to inspect:
- tests/test_named_binary_semantics.py
- artifacts/named_binary_label_pair_census.csv
- artifacts/named_binary_audit_gate.json
- artifacts/named_binary_classification_contract.json

Files to modify:
- Only pm_research/semantics/lexicon.py if census head requires lexicon updates.
- Do not modify audit semantics unless gate output proves a bug.

Constraints:
- No probe.
- No wallet discovery implementation.
- No PnL.
- No trading.
- No log_index.
- No full data materialization.

Expected output:
- Files changed, if any.
- Tests run.
- Audit command run.
- Gate JSON summary.
- Review of label-pair census head.
- Whether nb_contract_version changed.

Validation command:
python -m pytest tests\test_named_binary_semantics.py -q
python scripts\audit_named_binary_semantics.py --root C:\b1\data --out-dir artifacts

Evidence to return:
- gate_state
- total_conditions
- named_binary_eligible
- usable_named_binary_conditions
- token_id_coverage
- outcome_index_coverage
- resolution_mapping_success_rate
- orientation_correctness_rate
- blocked_reason_counts
- nb_contract_version

What not to change:
- Rank 1A / Rank 2 scripts
- schemas.py
- store.py unless explicitly approved
- Chat2 code

Stop conditions:
- token_id read as float
- scientific notation appears in token_id
- audit produces all-one-class suspicious result
- resolution mapping success unexpectedly low
```

### Short-term tasks

- Finalize label lexicon after census review.
- Review named-binary audit gate.
- If gate clears, decide whether Chat2 implementation can begin or whether named-binary probe spec is next.

### Later tasks

- Chat2 Dune wallet discovery implementation after Chat1 contract/audit exists.
- Named-binary forecast probe only after semantics gate clears and explicit authorization.
- Optional Phase 1 revalidation on 16.5M.
- Optional H1′ PnL-by-role only if user explicitly approves.

### Optional cleanup

- Add `ORCHESTRATOR_STATE.md`, `CLAUDE_TASK_QUEUE.md`, `VALIDATION_CHECKLIST.md`.
- Archive Rank 1A/Rank 2 closure memos.

---

## 16. Files the Next Assistant Should Inspect First

| Path | Why it matters | Status |
|---|---|---|
| `pm_research/semantics/named_binary.py` | Core orientation/classification glue | New, needs review |
| `pm_research/semantics/mapping_audit.py` | Core identity audit | New, critical |
| `pm_research/semantics/resolution_schema.py` | Resolution mapping/leakage risk | New, critical |
| `pm_research/semantics/lexicon.py` | Subclass rules/version | New, likely to change after census |
| `scripts/audit_named_binary_semantics.py` | Main audit entrypoint | New, must stream |
| `scripts/named_binary_label_census.py` | Census for lexicon | New |
| `tests/test_named_binary_semantics.py` | Validation suite | New |
| `artifacts/named_binary_label_pair_census.csv` | Human-review input | To be generated |
| `artifacts/named_binary_audit_gate.json` | Current gate | To be generated |
| `DUNE_DATA_NOTES.md` | Supporting Dune lessons | Project file supporting only |
| `PROJECT_STATE.md` | Truth source | Claude Project file |

---

## 17. Commands to Verify Current State

```powershell
cd C:\b1\pm_research
conda activate pmresearch
$env:PYTHONPATH="C:\b1\pm_research"
```

Git state if repo has git:

```powershell
git status
git log --oneline -10
```

Environment:

```powershell
python --version
python -c "import pandas, pyarrow, numpy, scipy, sklearn, matplotlib; print('imports OK')"
```

Token precision:

```powershell
python -c "import pandas as pd, glob; f=glob.glob(r'C:\b1\data\trades\*.parquet')[0]; df=pd.read_parquet(f, columns=['token_id']); print(df['token_id'].dtype, type(df['token_id'].iloc[0]), df['token_id'].iloc[0])"
```

Markets schema:

```powershell
python -c "import pandas as pd; df=pd.read_parquet(r'C:\b1\data\markets.parquet'); print(df.columns.tolist()); print(df.head(3).to_string())"
```

Named-binary tests:

```powershell
python -m pytest tests\test_named_binary_semantics.py -q
```

Census:

```powershell
python scripts\named_binary_label_census.py --root C:\b1\data --out-dir artifacts
python -c "import pandas as pd; df=pd.read_csv(r'C:\b1\pm_research\artifacts\named_binary_label_pair_census.csv'); print(df.head(50).to_string())"
```

Audit:

```powershell
python scripts\audit_named_binary_semantics.py --root C:\b1\data --out-dir artifacts
python -c "import json; d=json.load(open(r'C:\b1\pm_research\artifacts\named_binary_audit_gate.json')); print({k:d.get(k) for k in ['gate_state','total_conditions','named_binary_eligible','usable_named_binary_conditions','token_id_coverage','outcome_index_coverage','resolution_mapping_success_rate','orientation_correctness_rate','blocked_reason_counts','nb_contract_version']})"
```

---

## 18. Conversation History Summary

1. User copied project/data from AWS to Windows laptop due to OOM on `t3.micro`.
2. Dependency issue `scikit` resolved as `scikit-learn`.
3. Linux `.venv` copy was identified as invalid for Windows and excluded/deleted.
4. Full data re-backfill revealed old dataset was OOM-truncated (~388k vs 16.5M trades).
5. Streaming market-structure audit built and run.
6. `PASS_BINARY_ONLY` initially became provisional due to suspected overlap/dedup/concentration issues.
7. Fast gate found blockers; root causes were null condition_id artifacts, NA-key duplicates, and row-level concentration mismatch.
8. Load hygiene fixed: analysis drops null condition_id; dedup prefers populated semantic keys.
9. Fast gate updated for analysis hygiene and condition-level concentration; returned `CLEAR_WITH_WARNINGS_FOR_RANK1A`.
10. Rank 1A forecast-vs-price built for Rung 0/1 only. Result negative.
11. Price-bucket diagnostic found `NO_ACTIONABLE_BUCKET`.
12. Rank 2 OrderFilled sample joined but multiple semantic bugs occurred and were corrected.
13. Dune exact-index validation resolved topic order as topic2=maker, topic3=taker.
14. Operator-leg filtering showed event role recoverable but passive/aggressive unproven.
15. Fee diagnostic inconclusive.
16. OrdersMatched chosen as economic-role instrument.
17. Dune CSV precision loss caused false 10/0 maker split; fixed by varchar casts and API CSV path.
18. Maker-pairing validated.
19. Expanded OrdersMatched role recovery: 117 trades, 96 wallets, 71 taker/46 maker, 100% recoverable, 0 ambiguity.
20. Rank 2 role-recovery closed; original maker-dominated H1 not supported.
21. Project migrated into Claude Project with Chat1/Chat2.
22. Chat1 produced named-binary spec and implementation plan.
23. Chat1 implemented semantics layer; schema correction addendum adapted to real sharded trades and trade-row `outcome` labels.
24. Local token_id precision check passed.
25. Chat2 spec approved as blocked until Chat1 contract/audit exists.

---

## 19. Evidence Ledger

| Claim | Status | Evidence | Command/artifact | Risk | Next verification |
|---|---|---|---|---|---|
| Full dataset is 16.5M trades | Trusted | re-backfill results from conversation | Unknown exact artifact | Need artifact path | Locate data summary artifact |
| Old Phase 1 used truncated data | Trusted/provisional | 388k vs 16.5M comparison | conversation outputs | exact old run artifact unknown | Optional revalidation |
| Rank 1A negative | Trusted | `forecast_vs_price_rank1a.json`, split table | `forecast_vs_price.py` | assumes probe code valid | Preserve artifacts |
| Price buckets no actionable edge | Trusted | `rank1a_price_bucket_diagnostic.json` | diagnostic run | bucket thresholds | Preserve artifacts |
| OrdersMatched role recoverable | Trusted | expanded run 117 rows, 0 ambiguity | `ordersmatched_economic_role_expanded.json` | no PnL | No action unless H1′ authorized |
| Original maker H1 unsupported | Trusted as role distribution | 71T/46M mixed/taker-leaning | expanded artifact | not profitability | Do not overclaim PnL |
| Chat1 implementation ready | Provisional | Claude handoff + tests in sandbox | needs local pytest/audit | local integration might fail | Run tests/census/audit |
| token_id reads as string | Trusted | local dtype output | pandas command | only first shard checked | audit guard covers floats |
| Named-binary audit clears | Unknown | not run | N/A | many schema/mapping risks | Run audit |
| Chat2 can implement | Blocked | needs Chat1 artifacts | N/A | semantic drift | Wait for contract/gate |

---

## 20. Glossary

- **Polymarket:** Prediction market platform on Polygon.
- **CLOB:** Central limit order book.
- **Condition ID:** Market condition identifier, primary market/condition join key.
- **Token ID:** ERC1155 outcome token identifier; very large integer, must be string-safe.
- **Outcome index:** Numeric outcome side index; may read as float, cast to int carefully.
- **Outcome label:** Human-readable outcome string, e.g., YES, NO, OVER, UNDER, team name.
- **Named-binary:** Two-outcome market where labels are not necessarily YES/NO, e.g., OVER/UNDER, UP/DOWN, team-vs-team.
- **YES/NO binary:** Canonical binary market with YES and NO outcomes.
- **Canonical side:** Reference side for oriented price; YES for YES/NO, OVER for OVER/UNDER, UP for UP/DOWN, side_0 for arbitrary named pairs.
- **canonical_side_price / oriented_price:** Generalized primitive replacing label-derived `yes_price`.
- **yes_price:** YES/NO-only alias of canonical_side_price.
- **OrdersMatched:** Polymarket event used for trade-level economic role; `takerordermaker` identifies economic taker/aggressor.
- **OrderFilled:** Per-side/per-fill event useful for pairing and maker-side confirmation.
- **Gate:** Hard validation decision point that allows, warns, or blocks next work.
- **Audit artifact:** JSON/MD/CSV evidence output used to accept or reject a claim.
- **Data leakage:** Use of future outcome/resolution information in selection or features.
- **Holdout:** Out-of-sample evaluation window after discovery/cohort selection.

---

## 21. Handoff Notes for the New Assistant

The new assistant should be skeptical, precise, and gate-oriented. The project has repeatedly caught false progress only because validation was demanded before interpretation. Continue that pattern.

Do not assume:

- Claude’s successful tests imply real-data audit success.
- Dune fields preserve precision unless cast to varchar.
- labels define orientation.
- market_trades schema matches docs without inspection.
- role distribution implies profitability.
- `CLEAR_WITH_WARNINGS` admits ambiguous conditions.

Maintain these files after taking over:

### ORCHESTRATOR_STATE.md

Suggested initial contents:

```markdown
# ORCHESTRATOR_STATE.md
Current phase: Chat1 named-binary semantics audit pending.
Immediate blocker: run local tests/census/audit.
Chat2: spec-only, implementation blocked until Chat1 contract/gate.
Hard guardrails: no probe, no wallet discovery implementation, no PnL, no trading.
```

### CLAUDE_TASK_QUEUE.md

Suggested initial contents:

```markdown
# CLAUDE_TASK_QUEUE.md
1. Review test/census/audit outputs from Chat1.
2. If census head needs lexicon updates, update lexicon and bump NB_CONTRACT_VERSION.
3. If audit blocks, diagnose mapping/resolution/orientation failure.
4. Do not start Chat2 implementation until authorized.
```

### VALIDATION_CHECKLIST.md

Suggested initial contents:

```markdown
# VALIDATION_CHECKLIST.md
- token_id precision preserved
- outcome labels anchored to token_id/outcome_index
- winning_outcome mapped unambiguously
- no label-only inversion
- Dune uint256 cast to varchar
- discovery/holdout separation
- no PnL selection leakage
- no probe before gate
```

Output style most useful to the user:

- “Decision: approve/block/defer.”
- “What changed / what did not change.”
- “Do now / defer / stop.”
- “Prompt to Claude.”
- Exact commands and paste-back fields.

---

## 22. Machine-Readable Appendix

```yaml
project_name: Polymarket Research System
current_phase: named_binary_semantics_audit_pending
primary_goal: unlock named-binary market semantics safely via canonical_side_price/oriented_price before any named-binary probe or wallet discovery implementation
repo_root: C:\b1\pm_research
orchestrator_role: review Claude outputs, enforce gates, challenge assumptions, decide next prompts
implementation_agent: Claude Opus 4.8 High
canonical_entrypoints:
  named_binary_tests: python -m pytest tests\test_named_binary_semantics.py -q
  label_census: python scripts\named_binary_label_census.py --root C:\b1\data --out-dir artifacts
  named_binary_audit: python scripts\audit_named_binary_semantics.py --root C:\b1\data --out-dir artifacts
important_data_files:
  trades: C:\b1\data\trades\*.parquet
  markets: C:\b1\data\markets.parquet
  resolutions: C:\b1\data\resolutions.parquet
  prices: C:\b1\data\prices\*.parquet
important_output_files:
  rank1a: artifacts\forecast_vs_price_rank1a.json
  rank1a_buckets: artifacts\rank1a_price_bucket_diagnostic.json
  rank2_roles: artifacts\ordersmatched_economic_role_expanded.json
  named_binary_contract: artifacts\named_binary_classification_contract.json
  named_binary_gate: artifacts\named_binary_audit_gate.json
known_good_commands:
  token_precision: python -c "import pandas as pd, glob; f=glob.glob(r'C:\\b1\\data\\trades\\*.parquet')[0]; df=pd.read_parquet(f, columns=['token_id']); print(df['token_id'].dtype, type(df['token_id'].iloc[0]), df['token_id'].iloc[0])"
known_bad_assumptions:
  - markets.parquet contains outcome labels
  - Dune uint256 can be exported as numeric safely
  - OrderFilled alone defines economic role
  - wallet != takerordermaker implies maker
  - label strings can orient price
  - row-level concentration blocks condition-level probe
trusted_results:
  - Rank1A price-only recalibration negative
  - Rank1A price-bucket diagnostic NO_ACTIONABLE_BUCKET
  - OrdersMatched role recovery complete with stable mixed/taker-leaning distribution
  - token_id local dtype is str in checked shard
provisional_results:
  - named-binary semantics implementation until local tests/audit pass
  - Chat2 Dune wallet discovery spec until Chat1 contract exists
blocked_results:
  - Chat2 implementation blocked by Chat1 contract/audit
  - named-binary probe blocked by semantics audit gate
  - PnL-by-role blocked by explicit authorization
falsified_results:
  - delayed taker wallet-copying under Phase1 tested setup
  - pooled YES/NO price-only recalibration edge
  - price-bucket recalibration edge
  - original maker-dominated H1 as population claim
current_hard_blockers:
  - named_binary_audit_gate.json not yet produced/reviewed
  - named_binary_classification_contract.json not yet produced/reviewed
  - label-pair census not yet reviewed
next_action: run Chat1 tests, label census, and named-binary audit; paste gate JSON fields back to orchestrator
next_prompt_to_claude: ask Claude to review local tests/census/audit outputs and diagnose any block state without starting probes or Chat2 implementation
open_questions:
  - Does named-binary audit clear, warn, or block?
  - Does census head require lexicon update and NB_CONTRACT_VERSION bump?
  - What is usable named-binary universe size?
  - Can Chat2 implementation begin after Chat1 gate?
```
