# Donna OS — Architecture

Donna OS is a **multi-agent orchestration system** for Claude Code. You ask one agent — *Donna* — and she routes the work to a team of specialist agents, runs them in parallel across iterative rounds, checks their claims against the evidence, and returns a single synthesized answer.

This document describes the **full system** as it was built and run in production. This repository ships the part that works standalone today — **the interactive agent team** (the `.claude/agents/` role definitions). The **autonomous runtime**, the **observability data model**, and the **live console** described below are the fuller internal system; they're documented here because they're the real architecture, and they're on the [roadmap](../README.md#roadmap) for open-sourcing.

---

## 1. The layered model

```
  You ──► DONNA ──► employees (specialists) ──► skills ──► raw tools
          router      each with its own            reusable    Read, Grep,
          + synth     brain, memory, process       procedures  Bash, WebFetch
```

- **Donna** is the only agent that can dispatch others. She routes, orchestrates, and synthesizes — she never does the specialist work herself.
- **Employees** are 16 specialist agents (code review, QA, security, architecture, research, debugging, …), each a self-contained role definition with its own memory.
- **Skills** are reusable procedures an employee reaches for (some official, some third-party). Donna dispatches *employees*; the employee picks its own skills. This separation keeps routing simple and lets each role evolve independently.

---

## 2. The runtime loop

A single query flows through five phases. The engine runs each specialist as its **own operating-system process** (`claude -p`), so a round genuinely fans out in parallel.

```
  DECIDE ──► [ ROUND: dispatch specialists in parallel ] ──► EVALUATE ──┐
    │                                                          │        │
    │                          ┌───────────── loop (≤3 rounds) ┘        │
    │                          ▼                                        │
    │                    conflict? deeper question? vague finding?      │
    │                          │ yes → re-dispatch next round           │
    │                          │ no  ─────────────────────────────────► SYNTHESIZE ──► one answer
    └── 0 specialists ──────────────────────────────────────────────────►  (direct answer)
```

1. **Decide.** Donna reads the query and returns a plan: which specialists to involve and what to ask each. (For substantive questions this is typically 2–4 specialists; simple questions skip straight to a direct answer.)
2. **Dispatch (parallel).** Every chosen specialist runs concurrently as its own process. Each one reads its own role definition, pulls its recent memory, does the work, and writes its findings back.
3. **Evaluate.** Between rounds, an evaluation step asks: *do any two specialists contradict each other? did something surface a deeper question? was a finding too vague?* If so, it re-dispatches — pulling the previous round's findings forward so the next round builds on them.
4. **Loop.** Up to a bounded number of rounds (default 3), so the system converges rather than spinning.
5. **Synthesize.** A final step folds every round's findings into one answer — attributed to the specialists, with an explicit confidence level and a clear "here's your call."

---

## 3. The verification layer (anti-confabulation)

The single most important reliability idea in the system is the **verifier** — a dedicated agent whose only job is to catch fabrication before it reaches you.

When Donna is unsure, or the stakes are high (a security finding, a launch-readiness call), she spawns the verifier, which:

- **Re-runs an independent check** rather than trusting the specialist's word (e.g. runs its own code-review pass and diffs the two).
- **Reads the tool-call trace.** If a specialist *says* "I reviewed 8 files" but the trace shows 3 reads, that's flagged. If it *says* "tested on mobile" but only desktop calls appear, that's flagged.
- **Returns a per-claim verdict** — `VERIFIED` / `UNVERIFIED` / `FABRICATED` — plus a recommendation: accept, re-run with stricter instructions, or reject.

This is the "immune system" of the design. Multi-agent systems amplify hallucination — one agent's confident guess becomes another's input. The verifier is the counter-pressure.

---

## 4. Observability — every run captured end-to-end

A full run is recorded across a clean parent→child data model, so any run can be replayed, drilled into, or audited after the fact:

```
  session            ← the query + the final synthesized answer + rollups (rounds, specialists, tokens)
    ├─ specialist_call   ← one row per specialist, per round: input, output, tools used, tokens, timing
    │    └─ event         ← the LIVE stream of that agent's thinking + tool calls, as it happens
    ├─ memory_log        ← durable learnings written during the run
    └─ report            ← a self-contained, shareable snapshot of the whole session
```

Because every specialist's live event stream is persisted, the system isn't a black box — you can watch each agent think in real time, and you can go back later and see exactly *why* an answer came out the way it did.

---

## 5. The console

A live chat interface renders a run as it happens:
- Each query gets a response card that moves through *"choosing specialists → working (2/4 done) → writing your answer,"* with a live timer.
- Each specialist appears as a **chip** — round badge (R1/R2), status, role, a one-line preview, and its skill pills.
- Clicking any chip opens a **drill-in drawer** that streams that agent's thinking, tool calls, inputs, and final report in real time.
- A picker lets you either let Donna smart-route, or hand-pick specialists from a two-tier roster.

*(The AI runs only in the local Claude Code process — the hosted page deliberately redirects you to run it locally, so no model keys ever live server-side.)*

---

## 6. Reliability engineering

This is the part that separates "a demo" from "something that ran in production." The orchestrator is defensively built:

| Concern | Mechanism |
|---|---|
| **Process crashes mid-run** | On restart, the run is reconstructed from the database and **synthesized from partial work** rather than re-run from scratch. |
| **The watcher itself dies** | A supervisor restarts it with **exponential backoff** (5s → 15s → 30s → 60s), resetting once it's been healthy for a while. |
| **A timed-out agent leaves a zombie process** | Timeouts trigger a **process-tree kill**, fixing a real bug where a killed shell left the underlying agent alive to overwrite results. |
| **Rate limits** | Detected and retried with backoff (15s → 45s → 90s). |
| **Runaway cost** | Hard caps on rounds, specialists per round/session, wall-clock duration, and tokens — checked before *and* after each round. |
| **Two teammates trigger the same run** | **Atomic job-claiming** — exactly one worker owns a session; stale claims are reclaimable. |
| **A finished run gets clobbered** | Terminal state transitions are guarded — a late second worker can't overwrite a completed session. |
| **A missing dependency** | **Fail-open** everywhere: an unreadable skills directory, a repo it couldn't clone, a logging endpoint that's down — each degrades gracefully instead of failing the run. |

---

## 7. Design notes & honest limitations

Good architecture means knowing your system's edges. A few, stated plainly:

- **Conflict handling is *triggered*, not *adjudicated*.** The engine detects contradictions and re-dispatches to reconcile them, and the verifier catches fabrication — but there is no formal tie-break for two *honest-but-opposed* conclusions. When both specialists genuinely did the work and still disagree, the system surfaces the disagreement (with a confidence level) rather than mechanically deciding a winner. Formalizing that adjudication is the clearest next step.
- **The judgment is LLM-driven; the machinery is code.** The loop, the rounds, the re-dispatch plumbing, the caps, and the reliability scaffolding are deterministic code. The *decisions* — who to call, is this a contradiction, is another round worth it — are delegated to model steps. That's a deliberate trade-off: flexible routing, at the cost of non-determinism, bounded by hard caps and defaulting to "stop and answer" whenever a model step returns something unparseable.
- **Skills are external.** Some specialists reach for third-party skill packs; without them installed, those specialists fall back to their built-in reasoning. See [Skills & credits](../README.md#skills--credits).

---

*This architecture was built and run in production as the collaboration brain for a small human + AI team. The interactive framework is open here; the runtime, data model, and console are documented as the fuller system.*
