# PRICE INPUT CONTRACT — Named-Binary Probe (S0 inspection)

**Status:** S0 read-only inspection **COMPLETE** — price provenance and `yes_price` meaning are now **proven from source**. Documentation only — no code, no P1/P2, no scoring, no gate change. `named_binary_probe_blocked` stays `true`. **Verdict: P1 remains BLOCKED on the price input** (see Q5).

**Question this file answers:** can the local `prices` artifact support **non-YES/NO** canonical-side price assembly for the probe? `OrientationContract.canonical_side_price` needs **two per-side prices**; local `load_prices()` returns **one scalar** (`yes_price`). S0 read the price-build source and a data sample; the answer is now settled, not pending.

**Legend:** **[PROVEN]** from source/data read via S0 · **[FORBIDDEN-INFERENCE]** a conclusion the task barred without proof (and which the source in fact contradicts).

---

## What is PROVEN now (from `named_binary.py`, accepted semantics layer)

**The consumer's exact requirement** — `OrientationContract.canonical_side_price(side_0_price, side_1_price) -> float` **[PROVEN]**:
```python
def canonical_side_price(self, side_0_price, side_1_price):
    if self.canonical_token == self.side_0_token:  # side_0 is canonical
        return side_0_price
    if self.canonical_token == self.side_1_token:  # side_1 is canonical
        return side_1_price
    return side_0_price                            # non-oriented: side_0 by convention
```
- It consumes **two positional per-side prices**, `side_0_price` and `side_1_price`, defined as the prices of `side_0_token` / `side_1_token`, ordered by `outcome_index` (index 0 = side_0, index 1 = side_1).
- It selects by **token identity**, never by label. It never computes `1 - x`.
- `yes_price(side_0_price, side_1_price)` returns `None` for every non-YES/NO subclass and otherwise delegates to `canonical_side_price`. Docstring: for non-YES/NO subclasses `yes_price` is UNDEFINED and **callers must not synthesize it**. **[PROVEN]**
- Per-subclass canonical side: `YES_NO→YES`, `OVER_UNDER→OVER`, `UP_DOWN→UP`, `NAMED_OTHER→side_0 by index (no YES-equivalent)`. **[PROVEN]**

**The local supply** — `Store.load_prices() -> [condition_id, ts, yes_price]` **[PROVEN, from prior inspection + `schemas.PRICES_COLS`]**:
- One numeric column, `yes_price`, per `(condition_id, ts)`. **No** `token_id`, `outcome_index`, `side_0_price`, or `side_1_price` on price rows.

**The structural gap [PROVEN]:** the consumer needs two per-side prices in index order; the supply provides one scalar with no per-side discriminator. Both possible ways to close it are now resolved **negative** by S0: (i) there is **no** local per-token/per-side price artifact (Q3), and (ii) `yes_price` is **not** a proven side-identified half of a complementary book — the build code shows it is a YES-label flip that is explicitly **unsafe** for named binaries (Q1/Q2). Therefore `canonical_side_price` **cannot** be fed correctly for non-YES/NO conditions from current local data, and P1 stays blocked on the price input.

---

## Q1 — Where is `prices` generated? **[PROVEN]**

The `prices` parquet is built and saved locally by **`pm_research/data/backfill.py`**, written through `Store.save_prices(condition_id, df)` (`pm_research/data/store.py:165`, one parquet per condition under `prices/<condition_id>.parquet`). The `yes_price` column is constructed in two places, both in `backfill.py`:

- **`build_prices_from_trades(market_trades, ...)`** (`backfill.py:223`) — the preferred builder. The value line (`backfill.py:234`):
  ```python
  t["yes_price"] = t["price"].where(t["outcome"] == "YES", 1.0 - t["price"])
  ```
  i.e. take the trade price if the trade's `outcome == "YES"`, else `1 - price`. Then resample to hourly bars (last print per bar). Driven by `backfill_prices_from_prints(...)` → `save_prices`.
- **`parse_price_history(raw, condition_id, token_is_yes, ...)`** (`backfill.py:211`, value at `:217`):
  ```python
  "yes_price": float(h["p"]) if token_is_yes else 1.0 - float(h["p"])
  ```
  same YES-keyed `1-p` flip, gated on a `token_is_yes` flag. Used by `backfill_prices_from_clob(condition_id, token_id, ...)` (`backfill.py:455`).

Schema doc (`pm_research/data/schemas.py`): `PRICES_COLS = ["condition_id","ts","yes_price"]`; comment "PRICES — hourly YES price per market (**NO price = 1 - yes_price**)"; and an explicit **CAVEAT**: the existing conversion is *not* the "correct (index-/token-based) yes_price conversion."

---

## Q2 — What does `yes_price` mean for non-YES/NO conditions? **[PROVEN] — it is a YES/NO construction and is UNSAFE / undefined for named binaries**

The build code keys the value on the literal `outcome == "YES"` label (or a `token_is_yes` flag) and otherwise applies `1 - price`. This presupposes a market that **has a YES outcome** — i.e. a YES/NO book. For `UP_DOWN` / `OVER_UNDER` / `NAMED_OTHER` there is no "YES" outcome, so the `where(outcome=="YES", …, 1-price)` branch has no correct meaning: every non-YES/NO trade falls into the `1 - price` else-branch by default, which is a **mislabeling, not a canonical-side price**.

The project's own code says so explicitly (not just this analysis):
- `schemas.py`: the existing `yes_price` conversion is flagged with a **CAVEAT** and is contrasted with the "correct (index-/token-based) yes_price conversion" it does **not** perform.
- `audit_market_structure.py`: markets that are a "named-outcome binary" have a `yes_price` conversion that is **"unsafe"** (source comment `:14`); the report line `:590` labels named_binary markets "**unsafe for yes_price**"; and `:822` states "**yes_price is defined only for YES_NO**; other subclasses carry [a different/unsafe meaning]."
- `named_binary.py` (accepted layer): "`yes_price` is UNDEFINED (None)" for non-oriented subclasses and "callers must not synthesize it."

**Meaning for non-YES/NO conditions: not a valid canonical-side price — YES/NO-only, explicitly unsafe.** The value present in the `prices` table for these conditions is whatever the YES-keyed flip produced, which does not correspond to `side_0_price` or `side_1_price` by token identity.

**[FORBIDDEN-INFERENCE] — and now contradicted by source:** one must **not** treat `yes_price` as `side_0_price` nor reconstruct `side_1 = 1 - yes_price` for the probe universe. The task barred this without price-build proof; the price-build proof shows the conversion is a YES-label flip that is *unsafe* for exactly these subclasses. Doing the flip anyway would fabricate a per-side price from a mislabeled scalar.

---

## Q3 — Any local artifact with per-token / per-side prices? **[PROVEN] — No**

The S0 walk of `C:\b1\data` and `artifacts/` found **no** price-bearing table with a per-side/per-token discriminator. The only tables carrying `token_id` / `outcome_index` are:
- **`trades/<…>.parquet`** — trade records (`trade_id, wallet, condition_id, outcome, side, price, size_usdc, traded_at, tx_hash, token_id, outcome_index`). These are **trades, not a price series**; a per-print `price` with `token_id` is not an hourly per-side price series and is not what the probe's decision-time price needs.
- Diagnostic CSVs (`orderfilled_sample_join.csv`, `probe_blockers_*.csv`, `market_structure_csvs/*.csv`) — carry `token_id` for auditing, not prices.

The `prices` table itself has exactly `[condition_id, ts, yes_price]` (confirmed on the sample: 158 rows across 8 non-YES/NO conditions, 35–74 hourly rows each, **no** columns beyond the three, dtype `yes_price: float64`). There is **no** `side_0_price` / `side_1_price` / per-token price artifact anywhere local.

> **In principle** `backfill_prices_from_clob(condition_id, token_id, …)` could fetch a *specific token's* CLOB series, but (a) it still writes the collapsed single `yes_price` column via the same YES-keyed flip, and (b) it is a re-backfill operation, not an existing artifact. It does not constitute a currently-available per-side price series, and invoking it would be new data work, not S0.

---

## Q4 — Can `canonical_side_price(side_0_price, side_1_price)` be fed correctly from existing local data? **[PROVEN] — No**

- Q3 found **no** per-token/per-side price series → cannot supply the two identity-keyed prices directly.
- Q1/Q2 proved `yes_price` is a **YES-label flip**, YES/NO-only, explicitly **unsafe** for named binaries → it is neither a reliable `side_0_price` nor half of a proven complementary pair for `UP_DOWN`/`OVER_UNDER`/`NAMED_OTHER`. The one condition under which the flip would be admissible (source-proven, side-identified complementarity) is **disproven** by the source, which flags the conversion as unsafe for exactly these subclasses.

Therefore `canonical_side_price` **cannot** be fed correct per-side prices for the probe universe from current local artifacts.

---

## Q5 — Verdict (final, source-proven)

**P1 remains BLOCKED on the price input.** The decision-time canonical-side price for the non-YES/NO probe universe (`UP_DOWN`/`OVER_UNDER`/`NAMED_OTHER`) is **not derivable from current local artifacts**:
- the price *formula* is resolved (DATA_CONTRACTS §6: identity-select between two per-side prices), but
- the required **per-side price input does not exist locally** (Q3), and the available `yes_price` scalar is a YES/NO-only, source-declared-**unsafe** conversion for these subclasses (Q1/Q2).

`named_binary_probe_blocked` stays `true`. This is a genuine data-availability finding, consistent with the project's history of blocking on missing non-YES/NO inputs rather than fabricating them (cf. the earlier YES/NO-only *resolutions* blocker that was only unblocked by sourcing a proper non-YES/NO outcome table from Dune).

### What unblocking would require (NOT authorized here — future, separate work)
To give the probe a valid non-YES/NO decision-time price, someone would need to **build a per-side (token-identity-keyed) price series** for these conditions — e.g. a new artifact `[condition_id, ts, outcome_index/token_id, side_price]` sourced from CLOB/print history per token (the `backfill_prices_from_clob(condition_id, token_id, …)` path exists but currently collapses to the unsafe single column). That is a **price-source build task** analogous to the Stage 0–4 resolution-source build, and it is **out of scope** for S0. Until such a source exists and is audit-checked, P1 cannot compute `canonical_side_price` and the probe stays blocked on price input.

---

## S0 completion note

S0 is complete: the price-build source was read and the verdict is proven, not pending. The evidence file is `artifacts/named_binary_probe/price_input_s0_inspection.txt` (produced by `scripts/inspect_price_input_s0.py`, read-only). No further inspection is needed to establish the block; the next step (if authorized) is a **price-source build spec**, not more S0.

## Guardrails
No P1/P2 code or tests changed. No scoring, no Brier/log-loss/calibration/reliability/splits, no wallet/OrdersMatched/log_index/PnL, no gate change, `named_binary_probe_blocked` not flipped. This is an inspection/record artifact only.
