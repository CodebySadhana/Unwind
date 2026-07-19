# Unwind

**The undo layer for AI agent actions.** One command reverses an entire agent run.

> 🏆 Built for OpenAI Build Week with Codex + GPT-5.6 · **Developer Tools track**
> `git revert`, for what your agent did to production.

<!-- TODO: hero GIF here — the injection-then-revert moment, ≤15 seconds, autoplay -->
![demo](docs/assets/revert-demo.gif)

**Revert fidelity on UnwindBench: `XX.X%` across `NN` adversarial runs** (failures published below).

---

## The problem, in 20 seconds

AI agents can act now — update records, issue refunds, send emails. So everyone built
**locks**: permissions, approvals, identity for agents. But a lock can't help you once
the agent is inside and breaking things. Rollback rates across enterprise agent
deployments have hit ~74% — today that rollback is an engineer at 11pm reconstructing
what the agent touched from logs.

**Everyone built the lock. Unwind is the undo button.**

## Quickstart (30 seconds)

```bash
npx unwind demo        # seeded mock services + a scripted agent run
# ...watch the agent act, then:
npx unwind revert run_4a91
```

Or from source:

```bash
git clone https://github.com/<org>/unwind && cd unwind
pnpm install && pnpm dev
```

<!-- TODO: verify quickstart from a CLEAN machine before submission. This is the first thing judges do. -->

## How it works

Unwind is a proxy between your agent and real APIs. On every write:

1. **Shadow capture** — `GET` the record before the `PUT`. Store the diff.
2. **Compensation planning (GPT-5.6)** — the model reads the tool schema + payload and emits
   the exact inverse call(s), cached per endpoint so it runs once per action *type*.
3. **Tier gate** — every action is classified honestly:

| Tier | Meaning | Behaviour |
|---|---|---|
| **Reversible** | Before-state can be captured and restored | Auto-execute. Snapshot first. |
| **Compensable** | Can't be erased, can be neutralised (e.g. a Stripe refund) | Auto-execute with full logging. |
| **Irreversible** | Sent email, wire transfer, cascading delete | **Forced approval / hold queue. Never fake an undo.** |

4. **Revert** — `unwind revert <run-id>` replays compensations in reverse order and shows
   a diff of what changed back. Every action carries an append-only chain:
   `request → decision → call → outcome → compensation plan → revert status`.

<!-- TODO: architecture diagram (one image, 4 boxes) — docs/assets/architecture.png -->

## UnwindBench

We forked [AgentDojo](https://github.com/ethz-spylab/agentdojo) (NeurIPS 2024, the standard
prompt-injection benchmark for tool-calling agents) and asked the question the defence
literature doesn't: **not "did the bad call happen" but "can you undo it."**

| Suite | Runs | Injections landed | Reverted cleanly | Fidelity |
|---|---|---|---|---|
| Workspace | — | — | — | —% |
| Banking | — | — | — | —% |
| **Total** | **NN** | — | — | **XX.X%** |

**Published failures** (this table is the point — a system that claims 100% undo is lying):

| Run | Action | Why revert failed / was refused | Tier verdict |
|---|---|---|---|
| <!-- TODO --> | | | |

Reproduce: `pnpm bench`. Full methodology in [docs/UNWINDBENCH.md](docs/UNWINDBENCH.md).

## How Codex + GPT-5.6 built this

<!-- Judges are explicitly scored on this. Keep it above the fold of the bottom half. -->

- **Codex** wrote ~X% of the codebase across N sessions, steered by [AGENTS.md](AGENTS.md)
  and a project skill in `.codex/skills/unwind/`. Key sessions and decisions are logged in
  [CODEX_LOG.md](CODEX_LOG.md). `/feedback` session ID: `<!-- TODO -->`.
- **GPT-5.6** is load-bearing, not decorative: given an arbitrary API call, inferring its
  inverse (does this mutation cascade? which endpoint restores prior state?) is a genuine
  reasoning task. Plans are strict-JSON, validated, and **fail closed** — an unparseable
  plan means the action is treated as irreversible.
- Where the human intervened: <!-- TODO: 2–3 honest bullets — tier-gate policy, benchmark design, edge cases -->

## For judges

- **Test without rebuilding:** [docs/DEMO.md](docs/DEMO.md) — seeded sandbox, 5-minute script.
- **Demo video (<3 min):** <!-- TODO: YouTube URL -->
- **Why now:** the EU AI Act's high-risk obligations become enforceable **2 August 2026**;
  FINRA's 2026 report names autonomy, scope creep, and auditability as the novel agent risks.
  Unwind's ledger is that audit artifact as a byproduct of how the system works.

## Project structure

```
src/planner/    GPT-5.6 compensation planner (+ plan cache)
src/shadow/     before-state capture
src/gate/       tier classification + hold queue
src/ledger/     append-only chain + revert engine
bench/          UnwindBench (AgentDojo fork)
mocks/          4 seeded services: billing, CRM, tickets, email
```

## License

Apache-2.0 — see [LICENSE](LICENSE).
