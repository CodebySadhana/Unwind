# AGENTS.md — Unwind

> **Read this before touching any code.** This file is the source of truth for conventions,
> architecture decisions, and scope. When in doubt, follow this file over inference.

## What this project is

**Unwind** is the undo layer for AI agent actions. It sits as a proxy between an agent and
real APIs. On every write it captures before-state, has GPT-5.6 plan the inverse call
(a "compensation plan"), assigns a blast-radius tier, and can reverse an entire agent run
in one command: `unwind revert <run-id>`.

Mental model: **`git revert`, for what your agent did to production.**
Formal model: a **saga orchestrator** where an LLM writes the compensating transactions
(Garcia-Molina & Salem, 1987). Use saga vocabulary in code and docs: *compensation*,
*compensating action*, *unwinding*, *forward recovery*.

## Hard scope (48-hour build — do NOT expand)

- **4 mock services** (Stripe-like billing, CRM, ticketing, email) — no more.
- **~15 tool endpoints total** across those services.
- Benchmark harness forked from **AgentDojo** (`ethz-spylab/agentdojo-core`).
- One CLI, one small web ledger view. No auth, no multi-tenant, no persistence beyond SQLite/JSON.

If a task seems to require a 5th service or a new subsystem: stop and flag it instead of building.

## Architecture (4 components — keep boundaries clean)

1. **Planner** (`src/planner/`) — GPT-5.6 call. Input: tool schema + payload. Output:
   strict JSON `{ compensation_calls: [...], tier: "reversible" | "compensable" | "irreversible", rationale }`.
   Plans are **cached per (endpoint, schema-hash)** — the model runs per action *type*, not per action.
2. **Shadow capture** (`src/shadow/`) — GET-before-PUT on every write. Store `{before, after, diff}`.
3. **Tier gate** (`src/gate/`) — reversible → auto-execute (snapshot first);
   compensable → auto-execute + full log; irreversible → **hold queue / forced approval. Never fake an undo.**
4. **Ledger + revert** (`src/ledger/`) — append-only chain per action:
   `request → policy decision → tool call → outcome → compensation plan → revert status`.
   `unwind revert <run-id>` replays compensations in **reverse order** and prints a diff.

## Non-negotiable product rules

- **Honesty over coverage.** If an action cannot be undone, say so loudly. A fake undo is a bug of the highest severity.
- Irreversible actions NEVER auto-execute. No exceptions, including in demos.
- The ledger is append-only. No update/delete paths on ledger rows, ever.
- Every endpoint gets a published reversibility score, including the ones scored "no".

## Model usage

- **GPT-5.6** — compensation planning only (genuine inference: what is the inverse of this call?).
- Cheaper/faster model or pure rules — tier classification fast-path, log summarization.
- All planner calls: temperature low, strict JSON schema output, validated before use.
  A plan that fails validation → action is treated as **irreversible** (fail closed).

## Conventions

- Language: **TypeScript** (Node 20+). Package manager: `pnpm`.
- CLI verbs: `unwind run`, `unwind revert <run-id>`, `unwind ledger <run-id>`, `unwind bench`.
- Run IDs: `run_` + 4 hex chars (e.g. `run_4a91`).
- Names: repo `unwind`, MCP server `unwind-mcp`, benchmark **UnwindBench**. Do not rename anything.
- Tests: `pnpm test` (vitest). Every compensation recipe gets a round-trip test:
  apply action → revert → assert state equals snapshot.
- Commits: conventional commits (`feat:`, `fix:`, `bench:`, `docs:`).

## Commands

```bash
pnpm install
pnpm dev              # start proxy + mock services
pnpm test             # unit + round-trip compensation tests
pnpm bench            # run UnwindBench (AgentDojo fork) and write docs/UNWINDBENCH.md tables
```

## Definition of done (for any task)

1. Round-trip test passes for any new/changed compensation recipe.
2. Ledger entry chain is complete for the new action type.
3. If the action is irreversible, verify it lands in the hold queue and NOT auto-execution.
4. `README.md` quickstart still works from a clean clone.

## The only metric that matters

**Revert fidelity rate on UnwindBench.** Every architectural decision is judged against
whether it moves that number or protects its honesty.
