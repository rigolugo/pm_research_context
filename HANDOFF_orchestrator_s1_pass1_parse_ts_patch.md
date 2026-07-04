# HANDOFF — Claude → Copilot: S1 Pass 1 parse_ts datetime patch

**Task:** Patch the single S1 Pass-1 bug where the user-run failed before coverage execution
with `ValueError: unparseable timestamp: Timestamp('2026-05-08 04:27:06+0000', tz='UTC')`.
Timestamp-only fix. No behavior change beyond timestamp parsing.

## Root cause

`parse_ts` handled `int`/`float`/`str` forms only. `Store.load_trades()` returns `traded_at`
as **pandas `Timestamp`** objects; a tz-aware Timestamp stringifies to
`2026-05-08 04:27:06+00:00` (offset, not ` UTC`), which matched none of the three `strptime`
formats, so `parse_ts` raised. The failure was in `run_pass1_coverage → loader.load_trades_index
→ tvals = [parse_ts(v) for v in grp["traded_at"].tolist()]`.

## Fix (narrow, timestamp-only)

Added one branch to `parse_ts`, after the `bool`/`int`/`float` guards and **before** string
parsing:

```python
if isinstance(value, datetime):
    dt = value if value.tzinfo is not None else value.replace(tzinfo=timezone.utc)
    return dt.timestamp()
```

- **pandas `Timestamp` is a `datetime` subclass**, so this single branch covers both pandas
  Timestamps and Python `datetime`, tz-aware and tz-naive.
- **tz-aware** → epoch directly. **tz-naive** → treated as **UTC** (project convention), so the
  result is deterministic regardless of the runner's local timezone (no `.timestamp()`-on-naive
  local-tz drift).
- **String and epoch forms are unchanged** (they never reach the new branch).
- **Token-id / integer-identity handling is untouched** — token ids never pass through
  `parse_ts`; they go through `canonical_int`, which still fails loud on scientific notation.
  Verified in the guardrail re-audit.

No other function changed. No gate change; `named_binary_probe_blocked` not assigned/flipped.
No network, no data run, no Pass 2 / S2 / P1–P3 / probe / scoring / backfill / wallet work.

## Tests added (pure / mocked, no network)

In `tests/test_price_source_s1_coverage.py`:
1. `test_parse_ts_accepts_pandas_timestamp_utc` — the exact failing input →
   `1778214426.0`.
2. `test_parse_ts_accepts_python_datetime_with_tzinfo` — tz-aware datetime.
3. `test_parse_ts_naive_datetime_treated_as_utc` and
   `test_parse_ts_naive_pandas_timestamp_treated_as_utc` — naive forms coerced to UTC, agree
   with the tz-aware value.
4. `test_store_loader_load_trades_index_handles_pandas_timestamps` — injects a fake
   `pm_research.data.store.Store` returning an in-memory dataframe whose `traded_at` values are
   pandas `Timestamp`s (incl. the previously-failing value) plus 78-digit string-safe token ids;
   exercises the real `load_trades_index` path and asserts per-condition `min()` epoch + intact
   string-safe token pairs.
5. `test_parse_ts_string_and_epoch_forms_preserved` and `test_parse_ts_still_fails_loud_on_garbage`
   — regression guards that the patch neither loosened parsing nor broke fail-loud behavior.

## Test result

```
python -m pytest tests\test_price_source_s1_coverage.py -q
92 passed
```

(85 existing + 7 new.) Guardrail re-audit on the patched script: no
`yes_price` / `1 - yes_price` / `1 - price` / `1 - p` synthesis, no `save_prices`, no
`backfill_prices_from_clob`, `named_binary_probe_blocked` never assigned.

## Files delivered

1. `scripts/price_source_s1_coverage.py` — patched `parse_ts`.
2. `tests/test_price_source_s1_coverage.py` — 7 new tests.
3. this handoff note.

## Next concrete action

Copy the two files into `C:\b1\pm_research`, re-run the S1 Pass-1 tests locally, then re-attempt
the previously-failing user-run step. The S1 Pass-1 **network run itself remains unauthorized**
(separate explicit authorization required; user-run on the Windows box only).
