# HANDOFF — Claude → Orchestrator: S1 Price-Source Coverage, Pass 1 IMPLEMENTATION (v3, observed-shape capture)

**Task:** Implement **S1 Pass 1 coverage code + tests only**, per
`project_context/SPEC_price_source_s1_coverage.md` (ACCEPTED / coverage-only) and the
explicit in-chat authorization. v3 addresses the review BLOCK on the endpoint-shape ledger:
`price_source_s1_endpoint_shape.md` now captures the **observed** request/response shape from
the Pass-1 run (previously it rendered an always-empty list), and the coverage JSON now
reflects final `wrote_artifacts` / `artifact_dir` state.

**Status:** Implementation + pure-logic/mocked tests complete and passing in Claude's sandbox
(**85 passed**). **No network/data run was performed. No Polymarket endpoint was called.**
Files are delivered for the user to copy into `C:\b1\pm_research`; any real fetch is a later,
separately-authorized, **user-run** step on the Windows/Miniconda box.

---

## 0. v3 patch (this review cycle)

Two spec-critical defects fixed:

1. **Observed-shape capture.** `price_source_s1_endpoint_shape.md` previously rendered an
   always-empty observation list. Now the client records structural request/response facts
   into an in-memory `shape_sink` on every call, and `_build_shape_observations` distills the
   **first** single-token GET and **first** batch POST into a structure-only record that the
   renderer writes:
   - **GET:** path, params used (`market`, `startTs`, `endTs`, `interval`, `fidelity` when
     present), HTTP status, top-level response keys, detected series key (`history`/`prices`),
     first-point **keys**, point **count**, parse/Level-A status.
   - **Batch (`--allow-batch`):** path, payload keys, HTTP status, top-level response keys,
     `histories` key presence, and the batch-vs-single equivalence result (in the JSON).
   - **Deviations:** if a real response cannot be parsed to the documented shape, the observed
     deviation is recorded and the shape MD is best-effort written **before** returning
     `STOP_ENDPOINT_SHAPE_UNRECOGNIZED` (so S2 review can see the offending shape).
   - **No price series persisted.** The ledger stores keys/counts/statuses only — never point
     values. (Verified: injecting `p=0.4242` into a body yields a shape record and MD that
     contain the *key* `p` and a point count, but not the value.) Token ids may appear (already
     in the coverage ledger).
2. **JSON final-state.** `result["wrote_artifacts"] = True` and `result["artifact_dir"]` are
   now set **before** `_write_all_artifacts` writes `price_source_s1_coverage.json`, so the
   persisted JSON reflects final state (test-verified).

New/updated symbols: `observe_response_shape(...)` (pure structural extractor),
`PricesHistoryClient.shape_sink` + `_capture(...)`, `_build_shape_observations(...)`,
and a structure-only `_render_endpoint_shape_md(...)`. All heavy I/O stays behind the injected
abstractions; capture is exercised entirely by mocks.

---

## 1. What changed vs v1 (the runnable orchestration)

v1 shipped the pure-logic building blocks but its authorized CLI path only printed
"network path armed" and returned. v2 adds the **orchestration that composes those blocks
into an end-to-end runnable Pass-1 coverage run**, still hard-gated behind the two explicit
network flags, plus the loader/writer abstractions and the artifact emitters.

New in `scripts/price_source_s1_coverage.py`:
- `RunConfig` — run parameters incl. `network_authorized` (gates **all** fetch **and all**
  artifact writes).
- `StoreLoader` — real loader: `p0_preflight.json`, `named_binary_classification_contract.json`,
  `named_binary_resolution_source_rows.parquet` (read `dtype=str`), and trades via
  `Store(root).load_trades()` → per-condition `(condition_id, token_id, outcome_index)` tuples
  + `first_trade_ts = min(traded_at)`. All heavy imports (pandas / `Store`) are **lazy**.
- `ArtifactWriter` — real writer under `artifacts/named_binary_probe/price_source_s1/`;
  **refuses any `prices/` path**; JSON/MD/CSV emitters.
- `run_pass1_coverage(cfg, loader, client, writer)` — the runnable orchestration (steps 1–11
  below), depending **only on the three injected abstractions** so tests drive the whole run
  with mocks and never touch network/disk/parquet.
- `nearest_gap_seconds`, `level_c_validation`, `_pass1_verdict`, and the artifact renderers.
- `main()` now **builds the real loader/client/writer and runs the orchestration** on the
  authorized branch; the unauthorized branch returns `STOP_NOT_AUTHORIZED` and writes nothing.

---

## 2. End-to-end run steps (all Pass 1; each a guardrail checkpoint)

`run_pass1_coverage` performs, in order:

1. **Hard gate.** If `cfg.network_authorized` is False → return `STOP_NOT_AUTHORIZED`,
   **fetch nothing, write nothing**.
2. **Load + validate P0** (`load_p0_preflight` → `validate_p0`): require `p0_state == P0_CLEAR`
   and `nb_contract_version == nb-contract-2026-06-28.1` on the expected pin, contract stamp,
   and resolution-source stamp; adopt the **authoritative** `final_p0_eligible` from the
   artifact (accepted 39,693).
3. **Load contract conditions** (document-level version pin re-asserted).
4. **Load resolution-source rows** (`status == RESOLVED_SINGLE_WINNER`, read string-safe).
5. **Build universe** (`build_universe`): contract (`nb_eligible == True`, `nb_subclass ∈
   {UP_DOWN, OVER_UNDER, NAMED_OTHER}`) **inner-join on `condition_id`** with resolved rows;
   **subclass cross-check** hard-stops on mismatch; `YES_NO`/`UNUSABLE` excluded.
6. **Load trades index** for the universe cids; **enumerate token pairs from trades**
   distinct `(condition_id, token_id, outcome_index)` (`resolve_token_pairs`): exactly two
   stable string-safe side tokens; precision loss fails loud; large unresolved fraction →
   `STOP_TOKEN_ENUMERATION_UNRELIABLE`. **`resolved_winning_token_id` is never a pair source.**
7. **Build the deterministic Pass-1 stratified sample only** (`build_pass1_sample`), bounded by
   `--sample-size` (default 300); structural assertion that the sample is a strict subset and
   never the full universe.
8. **Fetch both side tokens for sampled conditions only** via the injected client
   (`GET /prices-history`, `market=<token_id>`, `startTs`/`endTs`/`interval`/`fidelity`).
   Batch (`POST /batch-prices-history`) is used **only if `--allow-batch`** and is
   equivalence-checked point-for-point against single-token (single is authoritative).
9. **Classify Level A** (per-token reachability status) and **Level B** (condition-level
   decision-window: `ts ≥ first_trade_ts + 3600` and strictly `< resolved_at`). `resolved_at`
   is used **only** as the window upper bound (leakage-safe). One-side coverage is not usable.
10. **Level C validation** on a spread sample **before trusting any all-one aggregate**; an
    all-one Level-A pool that fails the trust checks → `STOP_VALIDATION_REQUIRED`.
11. **Write the five coverage-only artifacts** and reconcile counts to the P0 universe.

**Verdict** (`_pass1_verdict`): `S1_SOURCE_VIABLE` (all three subclasses clear the 0.95
both-sides Level-B bar), `S1_SOURCE_PARTIAL`, `S1_SOURCE_NOT_VIABLE`, or
`STOP_VALIDATION_REQUIRED`. **Pass 1 success never auto-authorizes Pass 2.**

### Output artifacts (only when authorized + user-run), under
`artifacts/named_binary_probe/price_source_s1/`:
- `price_source_s1_coverage.json` — full machine-readable result.
- `price_source_s1_coverage.md` — narrative + honest framing.
- `price_source_s1_coverage_by_condition.csv` — coverage ledger: `condition_id, subclass,
  side_0_token, side_1_token, level_a_side_0, level_a_side_1, condition_reachability,
  level_b_class, nearest_gap_side_0_seconds, nearest_gap_side_1_seconds`. **No price values**
  (schema forbids any `p`/`price`/`*_price`/`series` column; asserted at import and in tests).
- `price_source_s1_endpoint_shape.md` — documented vs observed shape; records that `p` is a
  source-defined CLOB price with no complementary-side conversion.
- `price_source_s1_excluded.csv` — `TOKEN_PAIR_UNRESOLVED` / `NO_TRADE_ANCHOR`, reconciling.

**No price series is persisted anywhere; nothing is written under `prices/`.**

---

## 3. Tests run and results

Environment: Claude sandbox, Python 3.12, `pytest 9.1.1`. Pure logic; **no network, no disk
(except two `tmp_path` writer tests), no parquet** — the run is driven through mocked
`FakeLoader` / `FakeClient` / `FakeWriter`.

```
python3 -m pytest tests/test_price_source_s1_coverage.py -q
85 passed
```

New in v3 (endpoint-shape capture + JSON final-state), all mocked/no-network:
- **Authorized mocked run writes `endpoint_shape.md` with observed GET facts** (path, params
  used, HTTP status, top-level keys, series key, first-point keys, point count, Level-A status).
- **`--allow-batch` writes batch observed-shape facts** (path, payload keys, HTTP status,
  top-level keys, `histories` presence) **and the batch-vs-single equivalence result**.
- **A mocked endpoint-shape deviation is surfaced** — batch body missing `histories` →
  `STOP_ENDPOINT_SHAPE_UNRECOGNIZED`, deviation recorded in the result **and** written into the
  shape MD (`### Deviation` section), while the coverage JSON is **not** written on a stop.
- **Coverage JSON reflects final `wrote_artifacts` / `artifact_dir`** (parsed from the written
  bytes, not just the returned dict).
- **Real client populates `shape_sink`** via an injected transport (no network), and
  `_build_shape_observations` reads it correctly.
- **No price series in the shape ledger** — structural keys/counts only.

Plus the v2 orchestration tests and v1 pure-logic tests (unchanged):
- **No artifacts without network auth** — unauthorized run returns `STOP_NOT_AUTHORIZED`,
  writes nothing, fetches nothing.
- **Authorized mocked run writes all five expected artifacts** to the S1 directory (exact
  filename set asserted against `S1_ARTIFACT_FILENAMES`).
- **Run calls injected abstractions only** — loader/client/writer doubles; batch not called
  when disabled.
- **Samples Pass 1, never the full universe** — 360-condition fixture, `sample_size=30`:
  actual ≤ 30, strict subset, `fetched_token_count == 2 × sample_size_actual`,
  `sampled_only is True`.
- **Fetches both side tokens per condition.**
- **Pass 2 unavailable** from the orchestration result (`pass2_available == False`) and
  `run_pass2()` raises `Pass2NotAvailable`.
- **Stop propagation** — P0-not-clear and stale-contract halt the run and write nothing.
- **by-condition CSV has no price values** (header equals the coverage schema).
- **Clean both-sides fixture → `S1_SOURCE_VIABLE`; all-empty fixture → `S1_SOURCE_NOT_VIABLE`**
  but still writes the coverage artifacts (falsify cheaply, report honestly).
- **Batch only when enabled**; **disabled by default**.
- **`main()` default is unauthorized (rc=2)**; `build_run_config` reflects the auth flags.
- **Real `ArtifactWriter` refuses a `prices/` path** and writes under the S1 subpath.

> Discipline note (HANDOFF P1_REVIEW): these fixtures are **injected shapes**. Before any
> aggregate is believed, they must be reconciled against the **verified real**
> `/prices-history` response captured in `price_source_s1_endpoint_shape.md` during the first
> authorized user-run. The code fails loud (`STOP_ENDPOINT_SHAPE_UNRECOGNIZED`) if a real body
> deviates, rather than coercing.

---

## 4. Explicit non-performance statements

- **No network / data run by Claude.** Claude did not call `clob.polymarket.com` or any
  Polymarket endpoint. The real-`requests` path is a lazy import reached only on the user's
  box when both CLI flags are set; Claude never executes the authorized branch.
- **No Pass 2, no S2, no P1/P2/P3, no probe execution, no scoring** (no Brier / log-loss /
  calibration / reliability / splits), **no backfill**, **no writes into `prices/`**, **no
  wallet / OrdersMatched / log_index / PnL**, **no audit-gate modification**.
- **`named_binary_probe_blocked` is only observed/echoed from P0, never assigned or flipped**
  (executable-code audit confirms the identifier is never assigned).
- **No `yes_price` / `1 - yes_price` / `1 - price` / `1 - p` side synthesis** anywhere in
  executable code; `p` is carried verbatim as a source-defined price.
- The named-binary contract, orientation-by-identity, payout-vector winners, and the
  `first_price_after_warmup` policy are **reused, never redefined**.

---

## 5. Exact local Windows commands (later — only if separately authorized)

Copy into the repo:

```
C:\b1\pm_research\scripts\price_source_s1_coverage.py
C:\b1\pm_research\tests\test_price_source_s1_coverage.py
C:\b1\pm_research\project_context\HANDOFF_orchestrator_price_source_s1_pass1_IMPLEMENTATION.md
```

Run the pure-logic/mocked tests (no network; safe any time):

```powershell
$env:PYTHONPATH="C:\b1\pm_research"
cd C:\b1\pm_research
python -m pytest tests\test_price_source_s1_coverage.py -q
```

Dry-run the CLI (prints `STOP_NOT_AUTHORIZED`; performs **no** fetch, writes **nothing**):

```powershell
python scripts\price_source_s1_coverage.py --root C:\b1\data --artifacts-dir artifacts
```

**The Pass-1 network run itself is NOT authorized by this handoff.** It requires a separate,
explicit, in-chat user authorization naming the S1 Pass-1 coverage run, and even then it is
**user-run** (Claude does not fetch Polymarket data; no network-allowlist change is implied).
The armed invocation shape (both flags required):

```powershell
python scripts\price_source_s1_coverage.py `
  --root C:\b1\data --artifacts-dir artifacts `
  --sample-size 300 --seed 20260628 `
  --i-authorize-s1-pass1-network-run `
  --confirm-external-host clob.polymarket.com
```

Add `--allow-batch` only to also exercise `POST /batch-prices-history` (kept honest by the
per-token equivalence check; single-token remains authoritative).

---

## 6. Guardrail restatement

This task wrote code and tests **only**. It ran no data, fetched nothing, built no price
artifact, persisted no price series, computed no price/target/score, continued no P1/P2/P3,
touched no wallets/OrdersMatched/log_index/PnL, modified no audit gate, and did not flip
`named_binary_probe_blocked` (stays `true`). Pass 2 is hard-stopped and requires a separate
explicit go/no-go. S1 changes **no** project status by itself: the probe stays unauthorized,
P1 stays blocked on the price input, and whether to run S1 Pass 1 — let alone Pass 2 or an S2
build — is a fresh, user-authorized decision informed by (not granted by) this implementation.
