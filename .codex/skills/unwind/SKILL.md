---
name: unwind
description: Use when writing, reviewing, or testing anything in the Unwind codebase — compensation planners, shadow capture, tier gates, ledger/revert logic, or UnwindBench runs. Encodes the compensation-plan patterns, tier rules, and round-trip testing conventions this repo requires.
---

# Unwind engineering skill

Unwind is a saga orchestrator where GPT-5.6 writes the compensating transactions.
This skill encodes how to build inside it correctly. AGENTS.md holds scope and
architecture; this file holds the *patterns*.

## Compensation plan format (strict — validate before use)

Every planner output MUST parse against this shape. Anything else → fail closed
(treat action as irreversible, route to hold queue):

```json
{
  "tier": "reversible | compensable | irreversible",
  "compensation_calls": [
    { "method": "PUT", "endpoint": "/crm/contacts/{id}", "body_from": "snapshot.before" }
  ],
  "preconditions": ["record still exists", "no newer write to same field"],
  "cascades": ["webhooks fired on update — cannot be recalled"],
  "rationale": "one sentence"
}
```

Rules:
- `compensation_calls` is empty **iff** tier is `irreversible`. Never emit a "best effort" call for tier three.
- Restore from `snapshot.before`, never from a model-reconstructed value.
- If the planner is uncertain between two tiers, pick the **more restrictive** one.

## Canonical worked examples (use these shapes as anchors)

1. **Reversible** — CRM field update: compensation = `PUT` the `snapshot.before` payload back.
2. **Compensable** — Stripe charge: compensation = `POST /refunds` with an **idempotency key**
   derived from the ledger action ID. The charge is neutralised, not erased — say so in `cascades`.
3. **Irreversible** — send email: no compensation call. Tier gate must route to hold queue.
   A "send apology email" is NOT a compensation; never generate one.

## Revert engine rules

- Replay compensations in **reverse chronological order** of the original run.
- Check `preconditions` before each compensation; on violation: pause, report which
  action can't cleanly revert, continue with the rest only if independent (no shared resource).
- Failure **during** compensation is the hardest edge case (see Temporal's saga docs):
  record partial-revert state in the ledger; a revert must itself be resumable.
- Every revert writes new ledger entries. Never mutate the original chain.

## Testing conventions

Every compensation recipe ships with a **round-trip test**:

```
state0 = snapshot(service)
apply(action)
revert(run)
assert deepEqual(snapshot(service), state0)   // reversible tier
```

For compensable tier, assert the *neutralised* invariant instead (e.g. net balance
restored, refund object exists). For irreversible, assert the action landed in the
hold queue and was NOT executed. A recipe without its test is not done.

## Bench conventions

- `pnpm bench` runs the AgentDojo-forked suites and regenerates the tables in
  `docs/UNWINDBENCH.md`, including the **failure table**. Never hand-edit result numbers.
- Report fidelity per tier and per suite. Failures are published, not filtered.

## Style reminders

- Saga vocabulary everywhere: compensation, unwinding, forward recovery.
- CLI verbs only: `unwind run | revert | ledger | bench`.
- Ledger rows are append-only; no update/delete code paths may exist.
