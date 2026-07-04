# DUNE_DATA_NOTES.md

Status: Supporting reference only.

This file does not override:
- PROJECT_STATE.md
- GUARDRAILS.md
- DECISION_LOG.md
- CLOSED_FINDINGS.md
- ARTIFACT_INDEX.md

Purpose:
Preserve Dune-specific operational lessons from Rank 2 and Dune export/API work.

## 1. Role of Dune

Dune is acceptable for:
- bounded diagnostic queries
- wallet cohort discovery
- event-schema validation
- small/medium historical exports
- cross-checking decoded on-chain events

Dune is not currently the canonical production data pipeline.

Do not use Dune for:
- live trading
- paper trading
- wallet-copying
- strategy claims
- PnL claims without holdout
- replacing local audit gates

Dune-derived outputs must be versioned and treated as research artifacts.

## 2. OrdersMatched vs OrderFilled

OrdersMatched is the validated economic-role instrument.

Use OrdersMatched for trade-level economic-role analysis.

Core rule:
- takerordermaker = economic taker / aggressor order owner

OrderFilled is useful for paired context and maker-side confirmation.

OrderFilled alone is not sufficient to define economic role.

Correct role logic:
- wallet == takerordermaker -> economic taker confirmed
- wallet != takerordermaker AND wallet appears in paired OrderFilled context -> economic maker confirmed

Do not infer maker/passive by exclusion.

## 3. Historical Dune Event Tables

Historical samples used old CTF and old NegRisk event tables:

- polymarket_polygon.ctfexchange_evt_ordersmatched
- polymarket_polygon.negriskctfexchange_evt_ordersmatched
- polymarket_polygon.ctfexchange_evt_orderfilled
- polymarket_polygon.negriskctfexchange_evt_orderfilled

Newer V2/V3 tables may exist and may use different schemas.

Before unioning tables:
- inspect schema
- normalize column names
- cast large numeric fields to varchar
- document included/omitted tables and why

## 4. Dune Precision Rules

Dune uint256-like fields must be cast to varchar before export or API retrieval.

Always cast fields such as:

- makerassetid
- takerassetid
- makeramountfilled
- takeramountfilled
- tokenid
- asset_id
- makerAssetId
- takerAssetId
- makerAmountFilled
- takerAmountFilled

Example:

cast(makerassetid as varchar) as makerassetid,
cast(takerassetid as varchar) as takerassetid,
cast(makeramountfilled as varchar) as makeramountfilled,
cast(takeramountfilled as varchar) as takeramountfilled

## 5. Precision-Loss Failure Mode

78-digit token IDs cannot safely round-trip through CSV, Excel, or pandas as numeric floats.

Bad:
- 5.20896e+76
- 0.0

Good:
- full integer string
- 0

If scientific notation appears, fail loudly with:
DATA_EXPORT_PRECISION_LOSS

Do not attempt to reconstruct exact 78-digit token IDs from scientific notation. Precision is already lost.

## 6. Dune API vs UI Export

Dune API CSV retrieval preserved full integer strings when SQL cast uint256 fields to varchar.

Manual UI or Excel-style export can mangle large integers.

Preferred flow:
1. Dune query with varchar casts
2. Dune API CSV endpoint
3. local script reads asset/amount columns as strings
4. local script rejects scientific notation

Never open and re-save precision-sensitive Dune CSV files in Excel.

## 7. Dune API Key Handling

Do not hardcode Dune API keys.

Use:

$env:DUNE_API_KEY="your_api_key_here"

Example:

curl.exe -H "x-dune-api-key: $env:DUNE_API_KEY" "https://api.dune.com/api/v1/query/<QUERY_ID>/results/csv?limit=5000" -o C:\b1\csv\dune_export.csv

The results/csv endpoint fetches the latest result. It does not guarantee the query was freshly executed.

Safe sequence:
1. update Dune query
2. run query in Dune
3. fetch latest result through API CSV
4. verify no scientific notation
5. feed CSV to local script

## 8. Chat2 Dependency on Chat1

Chat2 is Dune Wallet Discovery for Named-Binary.

Chat1 is Named-Binary yes_price / canonical_side_price Rewrite.

Chat1 owns named-binary semantics.

Chat2 must consume:
- artifacts/named_binary_classification_contract.json
- artifacts/named_binary_classification_contract.md
- nb_contract_version

Chat2 must not define named-binary independently.

If Chat1 artifacts are unavailable, Chat2 must mark named-binary classification as a dependency and remain spec-only or provisional.

## 9. Named-Binary Wallet Discovery Rules

Use Dune for wallet-universe discovery and cohort construction only.

Do not use Dune to select wallets by full-period profitability.

Do not revive wallet-copying.

Use discovery window vs holdout window.

No named-binary testing can happen until Chat1’s named-binary audit gate clears.

Wallet cohort construction should consider:
- named-binary activity depth
- unique named-binary conditions
- active days/weeks
- role sample depth
- maker/taker role share if using OrdersMatched
- market diversity
- maximum concentration in one condition
- holdout-safe cohort freezing

No wallet cohort claim is allowed until out-of-sample evaluation.

## 10. Guardrails

Dune-related work must honor all project guardrails:
- no live trading
- no paper trading
- no wallet-copying
- no PnL-based selection without holdout
- no category-specialist claim before audit gates
- no named-binary test before Chat1 semantics gate clears
- no full indexer unless explicitly authorized
- no log_index work unless explicitly authorized
- no strategy claim from diagnostic outputs

## 11. Named-Binary Resolution Source — operational carry-forward (Stages 0–4)

Lessons from building the non-YES/NO realized-outcome source (`ctf_evt_conditionresolution` payout vectors). All confirmed on real runs.

- **Uploaded condition_id may be typed as varbinary.** When a contract/condition list is uploaded to Dune as a dataset, Dune can type a clean `0x`-lowercase hex `condition_id` column as **varbinary**, even though the preview renders it as text. Joins then fail with `Cannot apply operator: varbinary = varchar`. Fix: cast/normalize explicitly on the uploaded side — `cast(c.condition_id as varchar) = r...` (or `'0x' || lower(to_hex(...))` on both sides). Verify with `select condition_id, typeof(condition_id) from dune.<team>.<table> limit 3` before trusting a join; a format mismatch produces a misleading 0% coverage rather than an error.
- **Dune API CSV paginates / caps results.** The `results/csv` endpoint capped single responses well below the requested `limit` in practice (observed 5,000, then 32,000) regardless of `?limit=`. For full result sets use `limit`+`offset` and concatenate the parts (keep one header), or split the query by subclass. The Stage 3 builder accepts a glob via `--resolution-csv "nb_part_*.csv"` and concatenates automatically. Always verify the row count after fetch (`sum(1 for _ in open(...)) - 1`) against the expected total before running a build — a round number (5,000 / 32,000) is a truncation red flag.
- **payoutnumerators is array(uint256) — cast element-wise to varchar.** `cast(payoutnumerators as array(varchar))` preserves each element as a full-precision integer string. Reading via pandas/`dune-client` dataframe risks floating a uint256 into scientific notation per element; force `dtype=str` and fail loud on `e+`.
- **Winners come ONLY from payout numerators, never price convergence.** The binary winner is the single non-zero payout slot (exactly-one-nonzero-slot). Zero or multiple non-zero slots = ambiguous (excluded + counted), NOT a price tiebreak. Price-derived winners are forbidden as a definition (a convergence-based winner leaks/serves a different purpose). The NautilusTrader review reinforced this: borrow the one-winner-or-reject control flow, never the price path.
- **Trino editor mechanics.** No trailing `;` in the Dune query editor; run one statement at a time (pasting multiple `select`s at once errors). The API serves the LAST EXECUTION, so edit → run in Dune → then fetch CSV.
- **Per-condition vs aggregate CSV.** A per-subclass *aggregate* coverage CSV will NOT drive the Stage 3 builder (it has no per-condition `payoutnumerators`). Export a per-condition shape (Query-B-style, no LIMIT, varchar-cast).

## 12. Prompting Guidance

When using this file in Claude prompts, say:

Use PROJECT_STATE.md, GUARDRAILS.md, DECISION_LOG.md, CLOSED_FINDINGS.md, and ARTIFACT_INDEX.md as source of truth.
Use DUNE_DATA_NOTES.md only as supporting reference for Dune schemas, query/export/API lessons, OrdersMatched/OrderFilled lessons, and precision pitfalls.
Do not rely on Claude-old chat history.
