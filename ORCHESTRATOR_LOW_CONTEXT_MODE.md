# ORCHESTRATOR LOW-CONTEXT MODE

*A reusable operating protocol for review/decision tasks that must run without a full
repo scan. Pin this in the Claude Project Files panel. This file defines **how to operate**
in low-context mode; it does not grant authorization and does not override the source-of-truth
files.*

---

## Purpose

Some tasks are narrow reviews or go/no-go decisions that do **not** need the whole repo loaded.
This protocol keeps those tasks cheap and consistent: read the minimum, decide with a typed
verdict, stop. It exists so the discipline is written down (Rule 0) rather than re-derived from
chat each time.

## Precedence (read this first)

- **Source of truth is unchanged.** `GUARDRAILS.md`, `PROJECT_STATE.md`, `DECISION_LOG.md`,
  `CLOSED_FINDINGS.md`, `ARTIFACT_INDEX.md`, `DATA_CONTRACTS_*.md`, `PRICE_INPUT_CONTRACT_*.md`,
  and the active specs remain authoritative. This file never overrides them.
- **"Canonical repo" is a read-provenance statement, not a Rule-0 override.** Where a task says
  the GitHub repo (`rigolugo/pm_research`) is canonical, that means *that is where the pinned
  files are read/versioned from* — it does **not** change Rule 0 (pinned files are the only
  source of truth) or the delivery workflow (files are delivered for the user to copy into
  `C:\b1\pm_research`; real-data runs happen on the user's Windows box).
- **This file authorizes nothing.** No implementation, no data run, no network call, no probe,
  no scoring, no backfill, no P1/P2/P3 continuation, no gate change, no
  `named_binary_probe_blocked` flip. Authorization for any of those must be explicit, in-chat,
  and task-specific.

## Standing prohibitions

Do not restate them here (restating invites drift). They live in `GUARDRAILS.md` and the
"What is NOT authorized (standing)" section of `START_HERE.md`; both apply in full in
low-context mode.

---

## Minimal read policy

Read only the minimum files needed for the task:

1. `START_HERE.md`
2. `project_context/START_HERE.md` (if present as a separate onboarding file)
3. The specific file(s) under review.
4. Any contract / spec / handoff **directly referenced** by those file(s).
5. **Only if** the task touches authorization, implementation, data runs, network calls, probe
   execution, scoring, backfill, or P1/P2/P3 status, additionally read:
   - `GUARDRAILS.md`
   - `PROJECT_STATE.md`
   - the governing active spec for the task.

Do **not** clone/grep/scan the whole repo, inspect data, or run code unless the specific task
explicitly authorizes tests. Do **not** implement unless the specific task explicitly authorizes
implementation.

## Bootstrap pointer (point-in-time — verify, do not assume)

Repo bootstrap validation, when it has been performed, is recorded in its handoff
(e.g. `project_context/HANDOFF_orchestrator_github_bootstrap_validation.md`). If that handoff
records a passing bootstrap and nothing since has changed the repo/read provenance, do **not**
redo the full bootstrap. Treat "bootstrap is done" as a claim to confirm against that handoff,
not a standing fact baked into this protocol — if the handoff is absent or its state is unclear,
that is a `NEEDS VERIFICATION`, not an assumption.

---

## Decision protocol

Return a concise decision only — one of:

- **APPROVE** — the file/change is correct and within scope; proceed.
- **BLOCK** — a concrete defect or scope/guardrail violation; must be fixed before proceeding.
- **DEFER** — decision depends on something not yet available; name it.
- **ACCEPT FINDING** — record a result/finding as settled (typically a clean negative).
- **NEEDS VERIFICATION** — cannot conclude without checking an authoritative source; name the
  exact check.

Then: a short **why**, and the **next concrete action**. No long excerpts, no broad repo audit.
Stop after the decision.

> Discipline reminder (project rule): verify against an authoritative source before concluding;
> never conclude from one row or an all-one-role/all-one-direction output. If a claim in the task
> prompt asserts a fact you have not read this turn, prefer `NEEDS VERIFICATION` over trusting it.

---

## Closing

Even a low-context review closes with a brief Claude-to-Orchestrator note when it changes a file
or a decision state (what changed, the verdict, the next action). A pure read-only verdict that
changes nothing needs only the decision block above.
