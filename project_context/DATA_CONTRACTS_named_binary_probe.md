# DATA CONTRACTS — Named-Binary Probe

**Status:** Reference contract. Documentation only — authorizes no code, no P1/P2 continuation, no scoring, no probe execution. The named-binary probe remains blocked (`named_binary_probe_blocked = true`).

**Purpose.** Preserve the **exact, inspected** shapes of every artifact and code surface the probe (P0/P1) consumes, so no future chat infers them from memory. Every schema below was read from the real local files on `C:\b1` via `scripts/inspect_named_binary_probe_data_contracts.py` (read-only), not recalled. P1 paused because it assumed shapes that differ from these (see `HANDOFF_orchestrator_named_binary_probe_p1_REVIEW.md`); this file is the resolution of that pause.

**Legend.**
- **[AUTHORITATIVE]** — real, inspected, safe for P1 to depend on.
- **[DIAGNOSTIC-ONLY]** — present but not a probe input; do not build features/targets from it.
- **[ABSENT]** — a field P1 assumed that does **not** exist in the real artifact; must not be referenced.
- **[UNRESOLVED]** — a decision or dependency the Orchestrator must confirm before P1 is patched.

Inspection basis: `nb_contract_version = nb-contract-2026-06-28.1` across contract, resolution rows, and lexicon. Dataset: 288,684 conditions.

---

## 1. `artifacts/named_binary/named_binary_classification_contract.json`

- **Top-level type:** `dict` **[AUTHORITATIVE]**
- **Top-level keys:** `nb_contract_version`, `resolution_schema`, `conditions` **[AUTHORITATIVE]**
- **Version field:** `nb_contract_version = "nb-contract-2026-06-28.1"` (top level) **[AUTHORITATIVE]**
- **Where condition records live:** under the `conditions` key, a **list** of `288684` record dicts **[AUTHORITATIVE]**
- **Per-condition record keys:** `condition_id`, `nb_subclass`, `nb_eligible`, `nb_contract_version`, `exclusion_reason` **[AUTHORITATIVE]**

### Exact field names (this is what broke P1 Error 3)
| Concept | P1 assumed | **Real field** | Notes |
|---|---|---|---|
| condition id | `condition_id` | `condition_id` **[AUTHORITATIVE]** | `0x…` lowercase hex string |
| subclass | `subclass` | **`nb_subclass`** **[AUTHORITATIVE]** | the `subclass` key does **not** exist here **[ABSENT]** |
| eligibility | `eligible` (bool) | **`nb_eligible`** **[AUTHORITATIVE]** | value is the **string** `"True"` / `"False"`, not a JSON bool — compare as string or normalize explicitly |
| exclusion reason | — | `exclusion_reason` **[DIAGNOSTIC-ONLY]** | string; `"None"` when not excluded |

### Subclass vocabulary (exact, 5 values) **[AUTHORITATIVE]**
`YES_NO`, `UP_DOWN`, `OVER_UNDER`, `NAMED_OTHER`, `UNUSABLE`

Per-subclass record counts (all 288,684 conditions): `YES_NO 65,603`, `UNUSABLE 183,124`, `NAMED_OTHER 16,905`, `UP_DOWN 22,013`, `OVER_UNDER 1,039`.
`nb_eligible` distribution: `True 105,560` / `False 183,124`.

> **Note the `UNUSABLE` subclass** — P1 never modeled it. It is not in the probe's non-YES/NO set and must be excluded alongside `YES_NO`. The probe's eligible subclasses are exactly `{UP_DOWN, OVER_UNDER, NAMED_OTHER}`.

### Sample record (ids shortened) **[AUTHORITATIVE]**
```json
{
  "condition_id": "0xe033fd2300...8ce102",
  "nb_subclass": "YES_NO",
  "nb_eligible": "True",
  "nb_contract_version": "nb-contract-2026-06-28.1",
  "exclusion_reason": "None"
}
```

### Contract usage rule for P1 — **universe rule (accepted)**
The classification contract is **authoritative for classification and eligibility**. P1 builds its eligible universe by an explicit join of contract + resolution-source parquet, not from either alone:

1. Load the classification contract `conditions`.
2. **Normalize `nb_eligible` from string to bool** (`"True"` → `True`, `"False"` → `False`; treat any other value as a hard error, not as `False`).
3. Keep records where `nb_eligible == True` **and** `nb_subclass in {UP_DOWN, OVER_UNDER, NAMED_OTHER}` (excludes `YES_NO` and `UNUSABLE`).
4. Load `named_binary_resolution_source_rows.parquet`, keeping rows where `status == "RESOLVED_SINGLE_WINNER"` (all rows, on current data).
5. **Inner join on `condition_id`** (contract-eligible ∩ resolved).
6. **Cross-check** contract `nb_subclass == resolution-source `subclass`` per joined condition.
7. **Any subclass mismatch is a hard stop or an explicit-count exclusion — never silently accepted** (mirrors the project's verify-against-authoritative-data discipline and the missing-vs-ambiguous lesson).
8. Resolution-source rows supply the **resolved winner metadata** (`resolved_winning_token_id`, `resolved_winning_outcome_index`, `resolved_winning_label`), **`resolved_at`**, and the **source `subclass`** used for the cross-check. They do **not** override the contract for classification/eligibility.

> This supersedes the earlier "use the contract JSON only for the version pin" direction. The contract JSON is now a first-class universe input (normalized `nb_eligible` + `nb_subclass`), joined to the parquet on `condition_id`, with a subclass cross-check. The `nb_contract_version` equality pin (§P0/§version) still also applies. The join keys off `condition_id` (contract) ↔ `condition_id` (parquet); note the subclass column is named `nb_subclass` in the contract and `subclass` in the parquet — compare their **values**, which must be equal for oriented subclasses. Version pin still uses `nb_contract_version` and must remain `nb-contract-2026-06-28.1` on both sides.

---

## 2. `artifacts/named_binary/named_binary_resolution_source_rows.parquet`

- **Row count:** `39693` — exactly the pooled `resolved_single_winner` **[AUTHORITATIVE]**
- **Columns (9):** `condition_id`, `subclass`, `resolved_winning_token_id`, `resolved_winning_outcome_index`, `resolved_winning_label`, `resolved_at`, `source_table`, `status`, `nb_contract_version` **[AUTHORITATIVE]**
- **Dtypes:** every column is `str` **[AUTHORITATIVE]** (token id and outcome index are strings — honor the string-safe / `canonical_int` discipline; never cast to float)
- **`status` unique values (1):** `RESOLVED_SINGLE_WINNER` **[AUTHORITATIVE]** — the parquet is resolved-only by construction (ambiguous lives in the conflicts CSV, §3)
- **`subclass` unique values (3):** `UP_DOWN`, `OVER_UNDER`, `NAMED_OTHER` **[AUTHORITATIVE]** — note the column is `subclass` **here** (not `nb_subclass` as in the contract JSON); no `YES_NO`/`UNUSABLE` present
- **`source_table` unique (1):** `polymarket_polygon.ctf_evt_conditionresolution` **[DIAGNOSTIC-ONLY]**

### Winner columns **[AUTHORITATIVE]**
- `resolved_winning_token_id` — full-precision integer **string** (78-digit range)
- `resolved_winning_outcome_index` — **string** (e.g. `"0"`, `"1"`) — note the name is `resolved_winning_outcome_index`, **not** `resolved_outcome_index` (a P1 assumption used the shorter name) **[ABSENT: `resolved_outcome_index`]**
- `resolved_winning_label` — human label (e.g. `"THUNDER"`) **[DIAGNOSTIC-ONLY]** for target derivation; usable for eyeball/audit only, never as the target key

### `oriented_side_won`: **[ABSENT]**
`has_oriented_side_won = False`. There is **no** precomputed oriented-winner / "did the canonical side win" flag in this parquet. The held-aside target is derived by **token identity** against `OrientationContract.canonical_token` — see §6, **Target derivation (PINNED)**.

### `nb_contract_version`: `["nb-contract-2026-06-28.1"]` (single value) **[AUTHORITATIVE]**

### Per-subclass RESOLVED_SINGLE_WINNER counts **[AUTHORITATIVE]**
`UP_DOWN 22,012` · `OVER_UNDER 1,003` · `NAMED_OTHER 16,678` (sum = 39,693).

### Sample row (ids shortened) **[AUTHORITATIVE]**
```json
{
  "condition_id": "0x4175b1f3da...f3dc30",
  "subclass": "NAMED_OTHER",
  "resolved_winning_token_id": "869375323483...359964",
  "resolved_winning_outcome_index": "0",
  "resolved_winning_label": "THUNDER",
  "resolved_at": "2025-03-06 0...00 UTC",
  "source_table": "polymarket_polygon...conditionresolution",
  "status": "RESOLVED_SINGLE_WINNER",
  "nb_contract_version": "nb-contract-2026-06-28.1"
}
```

> **`resolved_at` is a string** like `"2025-03-06 00:00:00 UTC"`. P1 must parse it explicitly (it is not a datetime dtype). It is the **resolution** time — **[DIAGNOSTIC-ONLY / leakage-sensitive]**: it must NOT enter feature construction or the decision timestamp; the decision timestamp is derived from prices+trades (§5, §7). It may be used only for split ordering integrity checks if needed, never as a feature.

---

## 3. `artifacts/named_binary/named_binary_resolution_conflicts.csv`

- **Row count:** `253` — exactly the pooled `ambiguous_multiple_winners` **[AUTHORITATIVE]**
- **Columns (4):** `condition_id`, `subclass`, `status`, `payoutnumerators` **[AUTHORITATIVE]**
- **`status` unique (1):** `AMBIGUOUS_MULTIPLE_WINNERS` **[AUTHORITATIVE]**
- **`subclass` unique (3):** `UP_DOWN`, `OVER_UNDER`, `NAMED_OTHER` **[AUTHORITATIVE]**
- **`payoutnumerators`** — e.g. `"[1 1]"` (space-separated inside brackets) **[DIAGNOSTIC-ONLY]** (evidence of the tie; not a P1 input)

### Per-subclass ambiguous counts **[AUTHORITATIVE]**
`NAMED_OTHER 216` · `OVER_UNDER 36` · `UP_DOWN 1` (sum = 253).

These are **excluded** conditions. P1 never scores them; they exist to be counted, not consumed. (P0 already reconciles them.)

---

## 4. `artifacts/named_binary_probe/p0_preflight.json`

P1 must read this and require `p0_state == "P0_CLEAR"` and `authorized_scope == "P0_PREFLIGHT_ONLY"` before proceeding.

- **Top-level keys:** `stage`, `p0_state`, `authorized_scope`, `probe_execution_authorized`, `named_binary_probe_blocked_observed`, `nb_contract_version_expected`, `nb_contract_version_contract`, `nb_contract_version_resolution_source`, `gate_snapshot`, `counts_pooled`, `counts_by_subclass`, `excluded_counts`, `reconciliation`, `notes`, `created_at_utc` **[AUTHORITATIVE]**

### Fields P1 must consume **[AUTHORITATIVE]**
- `p0_state` → must equal `P0_CLEAR` (else P1 stops).
- `authorized_scope` → `P0_PREFLIGHT_ONLY`.
- `probe_execution_authorized` → `False` (P1 is feature assembly only; it does **not** flip this).
- `named_binary_probe_blocked_observed` → `True` (observed, never flipped).
- `nb_contract_version_contract` / `nb_contract_version_resolution_source` → both `nb-contract-2026-06-28.1`; P1 re-asserts equality.
- `counts_by_subclass` / `counts_pooled` → the eligible universe sizes P1 must reproduce (sanity check against its own load).

### Accepted P0 counts (real data) **[AUTHORITATIVE]**
Pooled: `contract_eligible 39957`, `source_rows 39946`, `resolved_single_winner 39693`, `ambiguous_multiple_winners 253`, `missing_source_rows 11`, `non_scoreable 0`, `final_p0_eligible 39693`.

By subclass:
| subclass | contract_eligible | source_rows | resolved | ambiguous | missing | final_p0_eligible |
|---|---|---|---|---|---|---|
| UP_DOWN | 22013 | 22013 | 22012 | 1 | 0 | 22012 |
| OVER_UNDER | 1039 | 1039 | 1003 | 36 | 0 | 1003 |
| NAMED_OTHER | 16905 | 16894 | 16678 | 216 | 11 | 16678 |

> `final_p0_eligible` (39,693) is the P1 candidate universe **before** the price/warm-up drop. P1 will further drop conditions with no price ≥ warm-up after first trade (§7); that count is a P1 output, not a P0 figure.

- `gate_snapshot` keys: `non_yesno_gate_state`, `non_yesno_scoreable`, `non_yesno_pooled_map_rate`, `pooled_threshold`, `subclass_threshold`, `per_subclass_map_rate`, `per_subclass_scoreable`, `base_gate_state`, `source_winner_count` **[AUTHORITATIVE]**. P1 re-confirms `non_yesno_gate_state == CLEAR_WITH_WARNINGS` and reads `per_subclass_scoreable` (all three true on current data).
- `reconciliation` keys: `exact`, `ambiguous_source`, `checked_fields`, `tautological_fields`, `mismatches`, `note` **[DIAGNOSTIC-ONLY]** for P1.

---

## 5. `pm_research.data.store`

- **Import path:** `from pm_research.data.store import Store` **[AUTHORITATIVE]**
- **Module file:** `C:\b1\pm_research\pm_research\data\store.py`
- **There are NO module-level `load_*` functions.** `load_prices`/`load_trades` are **methods on `Store`**, not module functions. **[ABSENT: `pm_research.data.store.load_prices` / `.load_trades` as free functions]** — this was P1 Error 1.

### Constructor **[AUTHORITATIVE]**
```python
Store(root: str | Path)
```

### Methods **[AUTHORITATIVE]**
`coverage_ok`, `has_prices`, `load_markets`, `load_prices`, `load_resolutions`, `load_trades`, `save_markets`, `save_prices`, `save_resolutions`, `save_wallet_trades`.

Confirmed loader shapes (from P1 REVIEW real run + schemas):
```python
Store(root).load_trades(wallets: list[str] | None = None) -> DataFrame
Store(root).load_prices() -> DataFrame
Store(root).load_resolutions() -> DataFrame
Store(root).load_markets() -> DataFrame
Store(root).has_prices(condition_id) -> bool
```
No per-condition push-down filter exists on the loaders; P1 filters to eligible `condition_id`s in memory.

### Returned columns (from `pm_research.data.schemas`) **[AUTHORITATIVE]**
- `PRICES_COLS = [condition_id, ts, yes_price]` — **no `token_id`, no `outcome_index` on price rows** **[ABSENT]** (P1 Error 2). Orientation cannot be cross-checked at the price-row level; it must come from the semantics layer keyed on condition. `yes_price` is a **YES/NO-oriented** view (see `OrientationContract` doc, §6).
- `TRADES_COLS = [trade_id, wallet, condition_id, outcome, side, price, size_usdc, traded_at, tx_hash, token_id, outcome_index]` **[AUTHORITATIVE]**. No `log_index` (consistent with project guardrails).
- `RESOLUTIONS_COLS = [condition_id, winning_outcome, resolved_at]` — this is the **local YES/NO-only** resolutions table (the one that blocks the legacy pooled gate); it is **[DIAGNOSTIC-ONLY]** for the probe. Non-YES/NO winners come from the resolution-source parquet (§2), never from here.
- `MARKETS_COLS = [condition_id, category, liquidity_usdc, created_at, resolution_at]` **[DIAGNOSTIC-ONLY]** for P1.

### Relevant trade columns for the first-trade timestamp **[AUTHORITATIVE]**
Use **`traded_at`** as the trade time and `condition_id` to group. First-trade time per condition = `min(traded_at)` over that condition's trades. (`token_id`/`outcome_index` exist on trades but are not needed for the decision-timestamp anchor.)

---

## 6. `pm_research.semantics.named_binary`

- **Module file:** `C:\b1\pm_research\pm_research\semantics\named_binary.py`
- **Public names:** `CANONICAL_LABEL`, `ClassificationRow`, `GATE_BLOCK_COVERAGE`, `GATE_BLOCK_MAPPING`, `GATE_BLOCK_ORIENTATION`, `GATE_BLOCK_RESOLUTION`, `GATE_CLEAR`, `GATE_CLEAR_WARN`, `GateInput`, `GateResult`, `GateThresholds`, `MAP_OK`, `MappingResult`, `NBSubclass`, `NB_CONTRACT_VERSION`, `ORIENTED_SUBCLASSES`, `OrientationContract`, `RES_OK`, `build_orientation`, `classify_condition`, `classify_label_pair`, `orientation_is_clean`, `run_audit_gate`.
  - **`canonical_side_price` and `yes_price` are METHODS on `OrientationContract`**, so they do not appear in the module-level public-name list. This is why the earlier public-surface scan reported `canonical_side_price` as "absent" — it is present, as a method (see the resolved price section below).

### `canonical_side_price`: **RESOLVED from source — it is a method on `OrientationContract`, and it does NOT take `yes_price`**

The public-name inspection missed it because it is **not a module-level function** — it is a **method on the `OrientationContract` dataclass**. Confirmed from `pm_research/semantics/named_binary.py`:

```python
class OrientationContract:
    condition_id: str
    nb_subclass: str
    side_0_token: str
    side_1_token: str
    canonical_token: Optional[str]     # None for non-oriented
    canonical_side_name: str           # 'YES'|'OVER'|'UP'|'side_0'
    has_yes_equivalent: bool

    def yes_price(self, side_0_price: float, side_1_price: float) -> Optional[float]:
        # Defined ONLY for YES_NO; None for every other subclass.
        if self.nb_subclass != NBSubclass.YES_NO.value:
            return None
        return self.canonical_side_price(side_0_price, side_1_price)

    def canonical_side_price(self, side_0_price: float, side_1_price: float) -> float:
        if self.canonical_token == self.side_0_token:
            return side_0_price
        if self.canonical_token == self.side_1_token:
            return side_1_price
        return side_0_price   # non-oriented: side_0 by convention
```

**Answers to the three sub-questions (from source, not memory):**

1. **Does an internal helper exist?** Yes — `OrientationContract.canonical_side_price(side_0_price, side_1_price)`. P1 can import and use `OrientationContract` and call the method; it is part of the accepted, audit-gated semantics layer. **[AUTHORITATIVE]**
2. **Exact signature / safety:** `canonical_side_price(self, side_0_price: float, side_1_price: float) -> float`. It selects the canonical side **by token identity** (`canonical_token == side_0_token` → `side_0_price`; `== side_1_token` → `side_1_price`; else `side_0_price`). Safe to call **iff** the caller supplies the two per-side prices in `outcome_index` order (`side_0` = index 0, `side_1` = index 1). **[AUTHORITATIVE]**
3. **The formula is NOT `p_canonical = yes_price` or `1 - yes_price`.** The accepted layer never derives the canonical price by flipping a single `yes_price`; it **selects between two per-side prices by identity**. There is no `1 - yes_price` anywhere in the orientation path. The earlier P1-REVIEW speculation about a sign flip is therefore **wrong by construction** — do not implement it.

**Orientation semantics (from the module docstring), per subclass** **[AUTHORITATIVE]**:
- `YES_NO` → canonical side = YES; `yes_price == canonical_side_price`; `has_yes_equivalent = True`.
- `OVER_UNDER` → canonical side = OVER.
- `UP_DOWN` → canonical side = UP.
- `NAMED_OTHER` (and TEAM_VS_TEAM) → `side_0`/`side_1` by `outcome_index` order; **no YES-equivalent**; `canonical_token = side_0_token` by convention; `canonical_side_name = "side_0"`; `has_yes_equivalent = False`.
- For non-oriented subclasses `yes_price(...)` returns `None` and the docstring states **callers must not synthesize it**.

### **[RESOLVED-B′ — BLOCKING / SOURCE-PROVEN NEGATIVE] The local `prices` table provides only one scalar, but `canonical_side_price` needs two per-side prices**

This is a genuine data gap, not a formula gap, and **S0 has now resolved it from source/data** (see `PRICE_INPUT_CONTRACT_named_binary_probe.md`, accepted). It is no longer pending inspection.

- `OrientationContract.canonical_side_price(side_0_price, side_1_price)` requires **both** per-side prices at the decision timestamp, in `outcome_index` order (index 0 = side_0, index 1 = side_1).
- `Store.load_prices()` returns `[condition_id, ts, yes_price]` — **a single scalar per (condition, ts)** (§5), with **no per-side/per-token price columns**.
- **`yes_price` is YES/NO-only and unsafe for `UP_DOWN` / `OVER_UNDER` / `NAMED_OTHER`.** S0 read the price-build source (`pm_research/data/backfill.py` `build_prices_from_trades`/`parse_price_history`): `yes_price` is built as `price where outcome=="YES" else 1 - price`, a YES-label flip that presupposes a YES/NO book. `schemas.py` flags this with an explicit CAVEAT (it is not the "correct index-/token-based" conversion), and `audit_market_structure.py` labels the named-binary `yes_price` conversion **unsafe** and states "yes_price is defined only for YES_NO." For non-YES/NO conditions the stored value is a mislabeled scalar, not `side_0_price` or `side_1_price` by identity.
- **No local per-side/per-token price artifact exists.** The S0 walk of `C:\b1\data` and `artifacts/` found only trades tables (which carry `token_id`/`outcome_index` but are not a price series) and diagnostic CSVs; there is no `side_0_price`/`side_1_price`/per-token price series anywhere local.

**The three candidate closures are now settled — all negative:**
  - (a) Is `yes_price` really the side_0 price for these conditions? **Disproven by source.** The build code is a YES-label flip explicitly declared unsafe for named binaries; it is not a side-identified price, so `side_0_price = yes_price` / `side_1_price = 1 - yes_price` is **not** admissible. **[FORBIDDEN-INFERENCE, source-contradicted.]**
  - (b) Is there a per-side price artifact elsewhere locally? **No** (S0 Q3, `PRICE_INPUT_CONTRACT_named_binary_probe.md`).
  - (c) Are non-YES/NO decision-time canonical prices reconstructable from local data? **No** — this is the accepted outcome, and it keeps the probe blocked on a price-input basis.

**Consequence:** the decision-time canonical-side price for the probe universe (`UP_DOWN`/`OVER_UNDER`/`NAMED_OTHER`) is **not derivable from current local artifacts**. The price *formula* is resolved (identity-select between two per-side prices); the required per-side *input* does not exist locally and cannot be synthesized from `yes_price`. **P1 stays blocked on the price input.** No further read-only inspection is needed — S0 already answered this. The only path forward, if separately authorized, is a **spec-only price-source build plan** for a per-side / token-identity-keyed price series (analogous to the Stage 0–4 resolution-source build); until such a source exists and is audit-checked, P1 cannot compute the price. Do **not** implement a `1 - yes_price` flip on faith — the source shows it is wrong for these subclasses.

### Relevant classes/functions (exact signatures) **[AUTHORITATIVE]**
```python
@dataclass
class ClassificationRow(
    condition_id: str, nb_subclass: str, nb_eligible: bool,
    nb_contract_version: str, exclusion_reason: Optional[str] = None)

@dataclass
class OrientationContract(
    condition_id: str, nb_subclass: str,
    side_0_token: str, side_1_token: str,
    canonical_token: Optional[str], canonical_side_name: str,
    has_yes_equivalent: bool)
    # doc: "Per-condition oriented price contract. yes_price is a YES/NO-only view."

def build_orientation(mapping: MappingResult, nb_subclass: str) -> OrientationContract
def classify_condition(mapping: MappingResult) -> ClassificationRow
def classify_label_pair(label_a: Optional[str], label_b: Optional[str]) -> NBSubclass
def orientation_is_clean(contract: OrientationContract) -> bool
    # doc: "An oriented subclass must have found its canonical token by identity."
```

### Token-identity / orientation audit fields — availability
- `OrientationContract` exposes, per condition **[AUTHORITATIVE]**: `side_0_token`, `side_1_token`, `canonical_token` (`Optional[str]`), `canonical_side_name`, `has_yes_equivalent`.
- `orientation_is_clean(contract)` confirms the canonical token was found by identity (mirrors the validated `orientation_correctness_rate = 1.0`).
- These are keyed on **condition**, consistent with prices lacking token columns (§5). Orientation is a condition-level contract, not a price-row attribute.
- `NB_CONTRACT_VERSION = "nb-contract-2026-06-28.1"` (also `lexicon.NB_CONTRACT_VERSION`, same value) **[AUTHORITATIVE]**.

### Target derivation (oriented winner) — **PINNED**
Because `oriented_side_won` is **[ABSENT]** (§2), the held-aside target is derived by **token identity** (this is now pinned, not a choice):

```
y_target_target_only_not_feature = 1
    iff canonical_int(resolved_winning_token_id) == canonical_int(OrientationContract.canonical_token)
    else 0
```

- **Primary rule = token identity** (above), consistent with orientation-by-identity (`orientation_correctness_rate = 1.0`).
- `resolved_winning_outcome_index` → `side_0_token` / `side_1_token` is a **cross-check only**, never the primary target key.
- **Exclusion:** if `OrientationContract.canonical_token is None` **or** `orientation_is_clean(contract)` is `False`, the condition is **excluded and counted as `ORIENTATION_NOT_DERIVABLE`**. It is **never defaulted to 0** (defaulting would fabricate a target and could trip/mask the all-one-direction guard).
- The name of the held-aside field is `y_target_target_only_not_feature` — it is a scoring target only and must never enter feature construction.

This resolves prior **[UNRESOLVED-A]**.

---

## 7. `scripts/forecast_vs_price.py` (Rank 1A) — warm-up / decision policy

- **Rank 1A DOES expose a reusable policy**, but as a **function parameter in hours**, not a shared seconds constant **[AUTHORITATIVE]**:
  - `build_condition_dataset(root: str, warmup_hours: float, usable_cids)` — decision_ts := first price ts ≥ first_trade_ts + `warmup_hours`.
  - CLI: `--decision-policy` default `first_price_after_warmup` (only choice); `--warmup-hours` type `float` **default `1.0`**.
  - Conditions with no price ≥ warm-up after first trade are dropped (cannot form a decision row).
- **There is no exported `warmup_seconds` constant** to import. **[ABSENT: shared `WARMUP_SECONDS`]**
- P1 currently defines its own `DEFAULT_WARMUP_SECONDS = 3600` (`named_binary_probe_p1_features.py:51`) and its own `first_price_after_warmup(...)`. **PINNED:** P1 uses **`warmup_seconds = 3600`**, which **equals Rank 1A `forecast_vs_price.py` default `--warmup-hours 1.0`**. No shared `WARMUP_SECONDS` constant exists to import; the **duplication is accepted**, and P1 must carry an explicit comment tying `3600` to Rank 1A's `--warmup-hours 1.0` so the two cannot drift unnoticed. This resolves prior **[UNRESOLVED-C]**.

> Decision-timestamp policy is `first_price_after_warmup` and is **lookahead-safe** (DECISION_LOG). P1 must not use any fixed-lead-before-resolution timestamp and must not read `resolved_at` into feature construction (§2).

---

## Cross-cutting: field-name reconciliation (the P1 failure map)

| Concept | P1 assumed | Real | Source |
|---|---|---|---|
| subclass in contract JSON | `subclass` | `nb_subclass` | §1 |
| eligibility in contract JSON | `eligible` (bool) | `nb_eligible` (string `"True"`/`"False"`) | §1 |
| subclass in resolution parquet | `subclass` | `subclass` (matches — parquet is the safe source) | §2 |
| winner index column | `resolved_outcome_index` | `resolved_winning_outcome_index` | §2 |
| oriented-winner flag | `oriented_side_won` present | **absent** | §2 |
| price token identity | `token_id`/`outcome_index` on price rows | **absent** | §5 |
| store loaders | module `load_prices`/`load_trades` | `Store(root).load_*()` | §5 |
| canonical price helper | `nb.canonical_side_price(...)` (module fn) | **method** `OrientationContract.canonical_side_price(side_0_price, side_1_price)`; selects by identity, needs TWO per-side prices | §6 |
| warm-up constant | shared `warmup_seconds` | Rank 1A `warmup_hours` float, default 1.0; no constant | §7 |

---

## Open decisions for the Orchestrator

- **[RESOLVED-A] Target derivation source.** Pinned to **token identity**: `y = 1 iff canonical_int(resolved_winning_token_id) == canonical_int(OrientationContract.canonical_token)`; `resolved_winning_outcome_index → side` is cross-check only; `canonical_token is None` or `orientation_is_clean == False` → excluded as `ORIENTATION_NOT_DERIVABLE`, never defaulted (§6).
- **[RESOLVED-B — formula] Canonical-side price formula.** Found in source: `OrientationContract.canonical_side_price(side_0_price, side_1_price)` selects the canonical side **by token identity** (not a `1 - yes_price` flip). `yes_price(...)` is `None` for all non-YES/NO subclasses; callers must not synthesize it (§6).
- **[RESOLVED-B′ — BLOCKING / SOURCE-PROVEN NEGATIVE] Canonical-side price *input*.** S0 is complete (`PRICE_INPUT_CONTRACT_named_binary_probe.md`, accepted). `canonical_side_price` needs **two per-side prices** in `outcome_index` order, but `Store.load_prices()` returns only `[condition_id, ts, yes_price]` — a single scalar (§5). `yes_price` is proven from source to be **YES/NO-only and unsafe** for `UP_DOWN`/`OVER_UNDER`/`NAMED_OTHER`, and **no local per-side/per-token price artifact exists**. The decision-time canonical price for the probe universe is therefore **not derivable from current local artifacts**. This is no longer pending inspection. **P1 stays blocked on the price input** (§6); the only forward path, if separately authorized, is a spec-only per-side price-source build.
- **[RESOLVED-C] Warm-up policy.** Pinned `warmup_seconds = 3600` (= Rank 1A `--warmup-hours 1.0`); no shared constant; duplication accepted with an explicit tying comment (§7).
- **[ACCEPTED] Universe rule.** Contract (normalized `nb_eligible` bool + `nb_subclass ∈ {UP_DOWN, OVER_UNDER, NAMED_OTHER}`) **inner-joined on `condition_id`** with resolved parquet rows (`status == RESOLVED_SINGLE_WINNER`); subclass cross-check `nb_subclass == subclass`; mismatch = hard stop / explicit-count exclusion. Contract remains authoritative for classification/eligibility (§1).

## Guardrails restated

This file changes no code, no tests, no gate. It does not authorize P1 continuation, P2, P3, scoring, splits, Brier/log-loss/calibration/reliability, wallet/OrdersMatched/log_index/PnL work, or probe execution. `named_binary_probe_blocked` stays `true`. The universe rule, target derivation [RESOLVED-A], warm-up [RESOLVED-C], and the canonical-price **formula** [RESOLVED-B] are pinned. The canonical-price **input** [RESOLVED-B′] is now **source-proven negative** (S0 complete): the local `prices` schema does not provide the two per-side prices `canonical_side_price` needs for non-YES/NO conditions, `yes_price` is YES/NO-only and unsafe for named binaries, and no local per-side/token price artifact exists. Therefore **P1 remains BLOCKED**: it cannot compute a decision-time canonical-side price from current local artifacts, and it cannot resume until a valid per-side / token-identity price source exists and is audit-checked. The next possible task, only if separately authorized, is a **spec-only price-source build plan** — no implementation, no data run, no scoring, no probe.
