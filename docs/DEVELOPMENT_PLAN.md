# AI Orchestration Engine — Development Plan

**Status:** Draft v1
**Owner:** ideabosque
**Repo:** `silvaengine/ai_orchestration_engine`
**Related:** `silvaengine/ai_coordination_engine` (procedure_hub), `silvaengine/a2a_daemon_engine`

---

## 1. Executive Summary

The **AI Orchestration Engine** (AOE) is a new SilvaEngine module that re-implements the multi-step orchestration responsibilities currently provided by `ai_coordination_engine`'s `procedure_hub`, with two fundamental architectural changes:

1. **One step = one session.** Today `procedure_hub` runs *all steps of a task* inside a single, self-invoking listener loop (Lambda re-entry every ~10s, capped at 10 iterations). The new engine treats every step as its own first-class execution session — created, dispatched, completed, and audited independently.
2. **A2A is the inter-step transport.** Steps no longer hand off via shared in-memory DAG state read by the same orchestrator process. Instead, when step *N* completes, its output returns to AOE; AOE remains the DAG authority, composes the successor input, and dispatches step *N+1* through the **A2A daemon** as a new supervised StepSession.

AOE's role becomes **operation & monitoring**: it owns the procedure/task definition, decides which step session to spawn next based on the DAG, dispatches it via A2A, observes status events, handles failures/retries/user-in-the-loop, and emits observability data. It does *not* directly invoke LLMs; agents do that themselves through the AI Agent Core Engine.

This plan defines the architecture, domain model, step lifecycle, A2A integration, module layout, and a phased build-out — designed so each phase delivers something demoable end-to-end.

---

## 2. Background & Motivation

### 2.1 What `procedure_hub` does today

(Detailed reference: `../ai_coordination_engine/ai_coordination_engine/handlers/procedure_hub/`)

Domain entities:
- **Coordination** — blueprint listing the agents involved.
- **Task** — workflow definition: a set of `agent_actions` with predecessors, primary path flag, optional `user_in_the_loop`, and optional `action_function`.
- **Session** — one execution of a Task.
- **SessionAgent** — per-agent execution state inside a Session, with `in_degree` for DAG dependency tracking.
- **SessionRun** — immutable audit record of one agent invocation.

Execution flow:
1. `executeProcedureTaskSession` mutation creates a Session and (optionally) decomposes the task query into subtasks via an orchestrator agent.
2. `init_session_agents` + `init_in_degree` materialize the DAG inside the Session.
3. The async listener `async_execute_procedure_task_session` runs a polling loop:
   - Find SessionAgents with `in_degree == 0`.
   - For each, either call `execute_action_function` (local) or `execute_session_agent` → `invoke_ask_model` (GraphQL into AI Agent Core Engine).
   - On agent completion, decrement successors' `in_degree` and self-invoke the listener for the next iteration.
4. Loop terminates when all agents are `completed`, any `failed`, or `wait_for_user_input` blocks progress.

### 2.2 Pain points

- **One Lambda invocation = many steps.** A single listener invocation can drive multiple ready agents, poll their async results, and self-invoke. Failure modes, timeouts, and retries cross step boundaries.
- **Polling-based async.** `update_session_agent` polls the AI Agent Core async task in a tight loop with a 60s timeout inside the listener Lambda. That blocks Lambda time on idle waits.
- **Hidden coupling to GraphQL invocation.** Every agent invocation goes through `invoke_ask_model` directly into AI Agent Core. There is no agent-addressable transport between procedure_hub and the agent runtime, so agent identity collapses to "what GraphQL operation do we call". A2A as the dispatch channel restores agent identity as a first-class address while keeping the runtime (AI Agent Core) unchanged.
- **Hard-coded loop heuristics.** 10s sleep, 10-iteration cap, a single execution strategy.
- **Mixed responsibilities.** `procedure_hub` simultaneously: stores DAG state, dispatches work, polls results, handles user-in-the-loop, and decides terminal status. Hard to evolve any one of these in isolation.
- **Race conditions on SessionAgent state.** Concurrent listener iterations + `executeForUserInput` can update the same row without conditional writes.

### 2.3 Why a new engine and not a rewrite

- The new design changes the *unit of orchestration* (step → step-session) and the *transport* (in-Lambda dispatch → A2A messages). Those propagate through models, listener, and integration boundaries.
- `procedure_hub` is in production. A clean parallel implementation lets us migrate workloads task-by-task and keep a working fallback.
- `a2a_daemon_engine` is now stable (v0.2.0, native A2A SDK v0.3.0+). It is the right time to build on it.

---

## 3. Architectural Goals

| # | Goal | Implication |
|---|------|-------------|
| G1 | Each step is its own session, with its own lifecycle, retry surface, and audit record. | Persistent `StepSession` entity; no implicit "current iteration" state. |
| G2 | Inter-step communication uses A2A messages. | AOE never mutates a downstream step in-process; it dispatches via the A2A daemon. |
| G3 | AOE is the operator/monitor of a task; agents are the workers. | AOE does not invoke LLMs directly. It schedules and observes. |
| G4 | Step boundaries are crash-safe checkpoints. | A failure in step *N* never corrupts step *N+1*'s state. Restartable from any boundary. |
| G5 | DAG semantics are preserved (predecessors, branches, user-in-the-loop, action functions). | Feature parity with `procedure_hub` at the task level. |
| G6 | Multi-tenancy via `partition_key = "{endpoint_id}#{part_id}"` from day one. | Avoid the migration burden ai_coordination_engine is taking on. |
| G7 | Observability is first-class — every state transition is emitted as a structured event. | New monitoring hub queries / dashboards become possible without new instrumentation. |
| G8 | Coexistence with `procedure_hub` during migration. | Same `Coordination` and `Task` blueprints can be executed by either engine. |

---

## 4. System Architecture

### 4.1 Components and boundaries

```
+----------------------------+        +------------------------------+
|  Client (UI / API caller)  |  --->  |  AI Orchestration Engine     |
+----------------------------+        |  (this module — operator)    |
                                      |                              |
                                      |  - Procedure / Task defs     |
                                      |  - TaskRun supervisor state  |
                                      |  - StepSession lifecycle     |
                                      |  - DAG progression           |
                                      |  - Status / metrics / logs   |
                                      +---------------+--------------+
                                                      |
                              dispatches step                observes step
                              via A2A dispatch             status events
                                                      |
                                                      v
                                      +------------------------------+
                                      |   A2A Daemon Engine       |
                                      |  - Agent registry            |
                                      |  - Message routing           |
                                      |  - Task assignment           |
                                      |  - Delivery + retry          |
                                      +---------------+--------------+
                                                      |
                                   HTTP POST to AI Agent Core's
                                   A2A-registered endpoint(s)
                                                      |
                                                      v
                                      +------------------------------+
                                      |  AI Agent Core Engine        |
                                      |  (the agent runtime —        |
                                      |   A2A-registered as the      |
                                      |   endpoint for every         |
                                      |   orchestrated agent_uuid)   |
                                      |  - Receives A2A dispatch      |
                                      |  - Invokes the named agent   |
                                      |    (LLM, tools, thread mgmt) |
                                      |  - Emits result back via A2A |
                                      +------------------------------+
```

Key boundary rules:
- **Every agent is invoked through `ai_agent_core_engine`.** AOE does not invoke LLMs or call AI Agent Core directly via GraphQL. AOE addresses agents by `agent_uuid` through A2A; AI Agent Core is the A2A-registered endpoint that actually runs them.
- **A2A daemon does not know about Procedures or DAGs.** It only knows agents, tasks (single-agent), and messages. It is the transport, not the orchestrator.
- **AI Agent Core does not know the DAG.** It only knows "I received an A2A task/message for `agent_uuid=X` with payload P; run agent X and report the result". Predecessor context, retry attempt, and step identity are all carried in the payload AOE composes.

### 4.2 The lifecycle of one task at a glance

```
1. Client -> AOE: executeTask(procedure, task, input)
2. AOE: persist TaskRun(status=pending), materialize StepNode DAG
3. AOE: pick step(s) with in_degree=0
4. For each ready step:
   a. AOE: persist StepSession(status=dispatching)
   b. AOE -> A2A: dispatch step (A2A `message/send` or daemon `assign_task` wrapper) with `{ step_session_uuid, inputs, ctx, callback }`
   c. A2A: deliver to AI Agent Core (HTTP POST w/ retry) — AI Agent Core is the
      A2A-registered endpoint for the agent_uuid
   d. AI Agent Core: invoke the named agent (LLM call, tool use, thread mgmt),
      then post the result message back to AOE's A2A address
   e. A2A -> AOE callback handler: parse, persist StepSession(status=succeeded|failed|wait_for_user_input)
5. AOE: on step completion, decrement successors' in_degree
6. AOE: schedule next ready steps (back to step 3)
7. When DAG drained: TaskRun(status=succeeded|failed)
```

There is **no listener loop polling DynamoDB**. Progression is driven by inbound A2A messages (step results) and explicit user actions (user-in-the-loop input). A short-lived "tick" Lambda exists only as a safety net for stuck states (configurable, optional).

---

## 5. Domain Model

All entities use composite partition key `partition_key = "{endpoint_id}#{part_id}"`.

### 5.1 Procedure (blueprint)

Renamed from `Coordination` to clarify role. A Procedure is the static workflow description.

| Field | Type | Notes |
|---|---|---|
| `partition_key` | str | hash key |
| `procedure_uuid` | str | range key |
| `procedure_name` | str | |
| `procedure_description` | str | |
| `agents` | list[AgentRef] | `{agent_uuid, agent_name, agent_type}` |
| `created_at`, `updated_at`, `updated_by` | | |

`agent_type` ∈ `{worker, decomposer, planner}` — kept compatible with current coordination semantics.

### 5.2 Task (blueprint)

Same role as today's `Task`: per-procedure workflow definition with agent dependencies.

| Field | Type | Notes |
|---|---|---|
| `partition_key` | str | hash key |
| `task_uuid` | str | range key |
| `procedure_uuid` | str | denormalized |
| `task_name`, `task_description` | str | |
| `initial_query_template` | str | optional templating |
| `step_definitions` | list[StepDef] | DAG nodes (see below) |
| `created_at`, `updated_at`, `updated_by` | | |

**StepDef** (was `agent_action`):

```
{
  step_id: str,                # stable id within the Task
  agent_uuid: str,
  predecessors: [step_id],
  primary_path: bool,
  user_in_the_loop: bool,
  action_function: { module_name, function_name } | null,
  output_aggregator: { module_name, function_name } | null,
  retry_policy: { max_attempts, backoff_seconds } | null,
  timeout_seconds: int | null
}
```

The static DAG is declared on the Task. If a decomposer is used, the Task still owns the blueprint/template, while `TaskRun` stores the materialized runtime graph generated from the decomposer output.

### 5.3 TaskRun (top-level execution record)

Replaces `Session` *only at the task level*. It does **not** hold per-step state.

| Field | Type | Notes |
|---|---|---|
| `partition_key` | str | hash key |
| `task_run_uuid` | str | range key |
| `procedure_uuid`, `task_uuid` | str | denormalized |
| `user_id` | str | |
| `task_query` | str / json | initial input |
| `input_files` | list | optional |
| `status` | enum | `pending → dispatching → in_progress → succeeded / failed / cancelled` |
| `materialized_step_definitions` | list[StepDef] | runtime StepDefs, equal to Task blueprint unless a decomposer expands them |
| `dag_state` | json | `{step_id: {status, step_session_uuid, in_degree}}` materialized graph |
| `subtask_queries` | list | output of decomposer if used |
| `final_output` | json | aggregated output (terminal step's result by default) |
| `created_at`, `updated_at` | | |

`dag_state` is a small JSON snapshot updated by AOE as steps progress. It is the source of truth for "what runs next" and is updated atomically with conditional writes.

### 5.4 StepSession (per-step execution record — the new primitive)

This is the core architectural change. **Each step gets its own session.**

| Field | Type | Notes |
|---|---|---|
| `partition_key` | str | hash key |
| `step_session_uuid` | str | range key |
| `task_run_uuid` | str | denormalized — GSI for "all steps of a run" |
| `step_id` | str | identifier within the Task DAG |
| `agent_uuid` | str | |
| `a2a_task_id` | str | A2A task id returned by the daemon / protocol adapter |
| `a2a_protocol_version` | str | e.g. `0.3.0`; records the protocol shape used for this dispatch |
| `aoe_contract_version` | str | e.g. `1.0`; records the SilvaEngine orchestration payload shape |
| `inputs` | json | `{ query, predecessor_outputs[], user_input?, files? }` |
| `output` | json | result payload from worker |
| `status` | enum | see §7 |
| `attempt` | int | retry counter |
| `error` | json | `{ type, message, traceback }` if failed |
| `started_at`, `dispatched_at`, `completed_at` | timestamp | |
| `created_at`, `updated_at`, `updated_by` | | |

Indexes:
- GSI by `task_run_uuid` for "list all steps of this run".
- GSI by `status` for "stuck steps" sweeps.

### 5.5 StepRun (audit / replay record)

Immutable per-attempt record. One StepSession may have multiple StepRuns if retried.

| Field | Type | Notes |
|---|---|---|
| `partition_key` | str | hash key |
| `step_run_uuid` | str | range key |
| `step_session_uuid`, `task_run_uuid`, `agent_uuid` | str | |
| `attempt` | int | |
| `a2a_message_in_id`, `a2a_message_out_id` | str | links to A2A message records |
| `agent_thread_uuid`, `agent_async_task_uuid` | str | links to AI Agent Core (worker may set these) |
| `started_at`, `completed_at` | timestamp | |
| `result` | json | snapshot of output / error |

### 5.6 Why split TaskRun and StepSession

| | `procedure_hub` (today) | AOE (proposed) |
|---|---|---|
| What "Session" means | the whole task execution | only the task-level coordinator record |
| Per-agent unit | `SessionAgent` (state inside a Session) | `StepSession` (its own session) |
| Lifecycle | shared listener loop iterates over all SessionAgents | each step is independently dispatched / observed |
| Restart granularity | task-level | step-level |
| Audit | `SessionRun` (1:1 with agent invocation) | `StepRun` (1:N per StepSession with retries) |

---

## 6. Operational & Quality Requirements

### 6.1 Test Coverage Strategy

This section names the *layers* of testing the engine commits to. Quantitative coverage and load-test targets are deferred until Phase 1c gives us a baseline (see §6.3).

**Unit Tests**
- `step_dispatcher`: DAG traversal, in_degree logic, parallel dispatch decisions
- `step_callback`: state machine transitions, retry logic
- `retry_policy`: backoff calculations, attempt tracking
- All models: serialization, validation, edge cases

**Integration Tests** — every public GraphQL mutation/query
- Mutations: `executeTask`, `executeForUserInput`, `cancelTask`, `insertUpdate*`
- Queries: list/get operations with filters
- A2A round-trip against the local A2A daemon (mock for Phase 0.5; switched to the real daemon once available)

**End-to-End Tests** — gate every Phase exit
- Happy path: simple task, DAG task, action-function task
- Failure scenarios: retry exhaustion, timeout, A2A delivery failure
- User-in-the-loop: pause, resume, cancellation

**Contract Tests** — payload validation between AOE and AI Agent Core (§8.2 / §8.3)
- AOE → A2A payload schema validation
- A2A → AOE callback message validation
- CI runs contract tests on every PR that touches `handlers/` or AI Agent Core's A2A handler.

**Load tests:** scope, concurrency targets, and SLOs are TBD — see §6.3.

### 6.2 Rollback & Disaster Recovery Plan

**Phase rollback strategy**
- Each phase / sub-phase is feature-flagged using a consistent scheme: `aoe.phase{N}[{a|b|c}].enabled` (e.g. `aoe.phase1c.enabled`, `aoe.phase2.enabled`, `aoe.phase2_5.enabled`).
- Phase rollout failure → disable the flag, revert to prior-phase behavior at the routing layer.
- Database schema changes are additive until Phase 8. No destructive migrations during phased rollout.

**Cross-engine routing during migration** (see Phase 6 / §12)
- Per-task config flag `executeWith ∈ {procedure_hub, ai_orchestration_engine}` is the rollback knob: flip a task back to `procedure_hub` if AOE misbehaves.
- Routing decisions are made at task creation / dispatch time, not mid-flight. We do **not** runtime-fail-over an in-flight TaskRun across engines — the two engines have different state shapes and a partial-execution handoff is a correctness risk.

**Data recovery**
- DynamoDB point-in-time recovery (PITR) enabled on all AOE tables.
- `StepRun` immutability provides forensic trail for replay.
- Cross-region backup is a Phase 8 / production-hardening concern, scoped per deployment environment.

**Circuit breakers** (in-engine, not cross-engine)
- Outbound A2A dispatch timeout (configurable; baseline measured in Phase 1c).
- Callback ingestion: if validation fails for >X% of inbound messages in a sliding window, alert + pause the affected `agent_uuid` / tenant. Threshold TBD post-baseline.
- The exact thresholds are intentionally not committed in this draft; they are calibrated against Phase 1c numbers (see §6.3).

### 6.3 Latency SLOs & Performance Benchmarks

**Targets are TBD until Phase 1c baseline.** The lesson from `ai_coordination_engine`'s `PERFORMANCE_OPTIMIZATION_PLAN` is to measure first, then commit. This section names the metrics we will track; numbers go in once Phase 1c gives us comparable readings against `procedure_hub`.

| Metric | Target | Measurement |
|--------|--------|-------------|
| Task dispatch latency (AOE -> A2A dispatch ack) | TBD (Phase 1c) | CloudWatch / structured log |
| Step callback processing (A2A → AOE → DAG advance) | TBD (Phase 1c) | end-to-end span |
| DAG state update (conditional write on `TaskRun`) | TBD (Phase 1c) | DynamoDB metric |
| End-to-end single-step task | within agreed factor of `procedure_hub` baseline | A/B vs. `procedure_hub` |
| End-to-end multi-step DAG | should improve over `procedure_hub` (parallelism) | A/B vs. `procedure_hub` |
| Sustained throughput (tasks/min) | TBD (Phase 7 load test) | dedicated load harness |

**Phase 1c deliverable:** capture the procedure_hub baseline for all metrics above on at least 3 representative tasks, and propose target values in a follow-up amendment to this plan.

**Regression gates**
- **Interim gate (until amendment):** no phase exits with a measured >2× regression vs. the procedure_hub baseline on the same task. This is a placeholder so Phase 1c has a falsifiable exit criterion; the post-amendment factor may be tighter or differentiated by metric.
- After the amendment, this section will list per-metric thresholds, and other phases / risks should refer to those rather than restating "2×".

### 6.4 Concurrency control on DAG state

`TaskRun.dag_state` lives as a single JSON field on the TaskRun row (§5.3). High-contention scenarios — fan-in steps where multiple predecessors complete in parallel and race to decrement the same successor's `in_degree` — must not corrupt that JSON.

**Approach: optimistic versioning, single-row.**
- Add a `version` integer field to `TaskRun`.
- On every `dag_state` update, callback handler reads `(dag_state, version)`, computes the new state, then conditional-writes `version = old + 1` with `ConditionExpression: version = :old`.
- On version conflict: re-read and retry, bounded (e.g. up to 5 attempts with short backoff).

This is the only mechanism committed in this draft. It is consistent with §5.3's "atomically with conditional writes" and does not require multi-row transactions or a separate lock table.

**Future option (deferred):** if contention measurements in Phase 2 / Phase 7 show retry rates above an agreed threshold, the model could split `dag_state` into per-step rows and use `TransactWriteItems`. That is a model change, not a tuning knob — it requires its own design pass.

### 6.5 Data Retention & TTL Policies

**Concrete retention durations are TBD, pending product / legal sign-off.** This section names the constraints the design must satisfy:

- Each TTL-able entity (`TaskRun`, `StepSession`, `StepRun`) stores a numeric epoch attribute (e.g. `expire_at`) that DynamoDB TTL can act on directly. (DynamoDB TTL cannot evaluate expressions like "completed_at + 90d" — the durations must be pre-computed at write time.)
- `StepRun` is immutable and treated as the authoritative audit trail; its retention is independent of `TaskRun`.
- Long-term retention is implemented via S3 archival (and optionally Glacier) before TTL deletion, not by extending DynamoDB TTL.
- `StepSession` retention follows its parent `TaskRun`.

A separate retention amendment will set the durations once product and compliance review concludes.

### 6.6 Backpressure & queue management

The mechanism is committed; the tunables are not.

**Concern.** Fan-out DAGs can produce many simultaneously-ready steps. Without limits, AOE can flood the A2A daemon, the AI Agent Core, or its own DynamoDB throughput.

**Mechanisms** (specific limits and thresholds calibrated against Phase 1c / Phase 7 measurements):
1. **Per-TaskRun concurrency cap.** AOE bounds the number of in-flight `dispatching` / `dispatched` / `running` StepSessions per TaskRun. Excess ready steps remain `created` and are dispatched as in-flight ones complete.
2. **Per-tenant dispatch rate limit.** Configurable cap on A2A dispatch calls per `partition_key` per minute.
3. **Queue-depth alerts.** Structured event `task.backpressure` is emitted when a TaskRun's `created` queue exceeds a configured threshold; alerting only — no behavior change.
4. **Shedding at the front door.** When per-tenant ready-queue depth exceeds a hard cap, AOE rejects new `executeTask` mutations with a `429`-style error. This is a safety valve, not a load-balancing strategy.

All four limits are configurable per environment. Default values are deliberately not committed in this draft.

---

## 7. Step Lifecycle & State Machine

### 7.1 StepSession states

```
                  +----------+
                  | created  |   (row written, not yet dispatched)
                  +----+-----+
                       |
                       v
                  +----------+
                  |dispatching|  (AOE called A2A dispatch; awaiting ack)
                  +----+-----+
                       |
                       v
                  +----------+
                  |dispatched|   (A2A acked; payload accepted by worker)
                  +----+-----+
                       |
                       v
                  +----------+
                  | running  |   (worker reported start)
                  +----+-----+
              +--------+---------+--------+--------------+
              v                  v                       v
        +-----------+      +------------+        +---------------+
        | succeeded |      |   failed   |        |wait_for_user  |
        +-----------+      +-----+------+        +-------+-------+
                                 |                       |
                       (retry?)  v                       v   (user input received)
                            +---------+              +-----------+
                            |retrying |              | resuming  |
                            +----+----+              +-----+-----+
                                 |                         |
                                 +-----+    +--------------+
                                       v    v
                                    (back to dispatching)
```

State transition triggers:
- `created → dispatching`: AOE picks step from ready queue.
- `dispatching → dispatched`: A2A daemon returns success on dispatch.
- `dispatched → running`: worker emits `step.started` A2A event (optional; some workers may skip and go straight to result).
- `running → succeeded`: worker emits result message; AOE callback persists output.
- `running → failed`: worker emits failure message *or* AOE timeout sweep.
- `running → wait_for_user_input`: worker emits a "needs input" message. AOE exposes it via mutation; on user input, StepSession transitions to `resuming` and AOE re-dispatches with augmented input.
- `failed → retrying`: if retry policy permits.
- `cancelled` (from any non-terminal state): explicit user/system cancel.

### 7.2 Why per-step session matters

- Lambda time budget is per-step, not per-task. A 5-step task with 60s steps is no longer a 300s Lambda.
- Failure replay is per-step. Re-run StepSession N without touching N-1 or N+1.
- Concurrency is natural: ready steps with `in_degree==0` can be dispatched in parallel as separate sessions.
- User-in-the-loop becomes a normal step state, not a global session pause.

### 7.3 DAG progression

`TaskRun.dag_state` is the in-memory DAG, persisted in DynamoDB:

```json
{
  "step_a": { "status": "succeeded", "step_session_uuid": "...", "in_degree": 0 },
  "step_b": { "status": "running",   "step_session_uuid": "...", "in_degree": 0 },
  "step_c": { "status": "pending",   "step_session_uuid": null,  "in_degree": 1 }
}
```

When a StepSession terminates, the AOE callback handler:
1. Updates the StepSession row.
2. Reads the TaskRun, decrements successors' `in_degree`, marks step's own status.
3. Conditional-writes `dag_state` (DynamoDB conditional update on `version`; see §6.4).
4. For successors with `in_degree==0`, triggers `dispatch_step`.
5. If no in-flight steps remain and all succeeded, marks TaskRun `succeeded` (or `failed` if any non-recoverable failure).

The conditional write replaces the implicit "single listener at a time" pattern in `procedure_hub` and makes parallel callbacks safe.

### 7.4 Sequence Diagrams

#### Happy Path: Single Step Execution

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌──────────────┐
│  Client │     │   AOE   │     │   A2A   │     │ AI Agent Core│
└────┬────┘     └────┬────┘     └────┬────┘     └──────┬───────┘
     │               │               │                 │
     │ executeTask() │               │                 │
     │──────────────>│               │                 │
     │               │               │                 │
     │               │ Create TaskRun│                 │
     │               │ + StepSession │                 │
     │               │───────┐       │                 │
     │               │<──────┘       │                 │
     │               │               │                 │
     │               │ dispatch_step() │                 │
     │               │──────────────>│                 │
     │               │               │                 │
     │               │               │ HTTP POST       │
     │               │               │────────────────>│
     │               │               │                 │
     │               │               │    202 Accepted │
     │               │               │<────────────────│
     │               │               │                 │
     │               │  Ack (task_id)│                 │
     │               │<──────────────│                 │
     │               │               │                 │
     │ task_run_uuid │               │                 │
     │<──────────────│               │                 │
     │               │               │                 │
     │               │               │    step.result  │
     │               │               │<────────────────│
     │               │               │                 │
     │               │ callback POST │                 │
     │               │<──────────────│                 │
     │               │               │                 │
     │               │ Update StepSession
     │               │ + Advance DAG │                 │
     │               │───────┐       │                 │
     │               │<──────┘       │                 │
     │               │               │                 │
     │ subscription/ │               │                 │
     │ query for   │               │                 │
     │ status      │               │                 │
     │<───────────>│               │                 │
     │               │               │                 │
```

#### DAG Progression: Fan-Out Parallel Steps

```
┌─────────┐     ┌─────────┐     ┌─────────────┐
│   AOE   │     │   A2A   │     │   Agents    │
└────┬────┘     └────┬────┘     └──────┬──────┘
     │               │                 │
     │ Step A completes                │
     │───────┐       │                 │
     │<──────┘       │                 │
     │               │                 │
     │ Decrement in_degree             │
     │ for B, C, D   │                 │
     │ (all reach 0) │                 │
     │───────┐       │                 │
     │<──────┘       │                 │
     │               │                 │
     │ Parallel dispatch               │
     │               │                 │
     │ dispatch(B)│                 │
     │──────────────>│                 │
     │               │──┐              │
     │               │  │              │
     │ dispatch(C)│  │ dispatch B  │
     │──────────────>│  │─────────────>│
     │               │  │              │
     │               │  │ dispatch C  │
     │ dispatch(D)│  │─────────────>│
     │──────────────>│  │              │
     │               │<─┘ dispatch D   │
     │               │───────────────>│
     │               │                 │
     │               │                 │
     │               │  Results (async)│
     │               │<────────────────
     │               │                 │
     │ callbacks     │                 │
     │<──────────────│                 │
     │               │                 │
```

#### Failure and Retry Flow

```
┌─────────┐     ┌─────────┐     ┌──────────────┐
│   AOE   │     │   A2A   │     │ AI Agent Core│
└────┬────┘     └────┬────┘     └──────┬───────┘
     │               │                 │
     │ dispatch_step() │                 │
     │──────────────>│                 │
     │               │                 │
     │               │    LLM call     │
     │               │────────────────>│
     │               │                 │
     │               │   LLM timeout   │
     │               │<────────────────│
     │               │                 │
     │               │ step.failure    │
     │               │ (retryable)     │
     │<──────────────│                 │
     │               │                 │
     │ Check retry   │                 │
     │ policy (attempt 1 < max)        │
     │───────┐       │                 │
     │<──────┘       │                 │
     │               │                 │
     │ Re-dispatch   │                 │
     │ (attempt 2)   │                 │
     │──────────────>│                 │
     │               │                 │
```

---

## 8. A2A Integration Design

### 8.0 Server / transport recommendation

**Recommendation:** use `a2a_daemon_engine` as the SilvaEngine A2A gateway, keep `ai_agent_core_engine` as the worker A2A server endpoint, and expose AOE as the callback / push-notification receiver.

This keeps orchestration ownership in AOE while avoiding a direct AOE -> AI Agent Core GraphQL dependency. It also lets the daemon hide daemon-specific routing, retries, and registry details behind a stable adapter.

Protocol alignment rule:
- The public wire shape should track the official A2A protocol version used by the daemon, currently expected to be A2A `0.3.0+`.
- The daemon may expose a SilvaEngine convenience API named `assign_task`, but that API must be documented as a wrapper over A2A `message/send` plus `pushNotificationConfig`, not as the A2A protocol itself.
- AOE stores both `a2a_protocol_version` and `aoe_contract_version` on each StepSession / StepRun so protocol migrations are auditable.

### 8.1 Conceptual mapping

| Procedure_hub concept | AOE + A2A equivalent |
|---|---|
| Direct GraphQL `invoke_ask_model` from procedure_hub into AI Agent Core | AOE sends a daemon dispatch request; the daemon emits A2A `message/send` to AI Agent Core's A2A endpoint; AI Agent Core invokes the agent |
| `SessionAgent.in_degree` decrement after async result | A2A task completion / callback handler in AOE advances `dag_state` |
| Listener self-invocation | None — eventful, no polling loop |
| `SessionAgent.user_in_the_loop` | StepSession `wait_for_user_input` mapped to A2A task state `input-required` |
| Action function (in-process, no LLM) | StepSession with `agent_action.action_function` set runs synchronously inside AOE without A2A — special case |

### 8.1.1 State mapping

AOE may use engine-specific state names internally, but all transport-facing code must map them to/from A2A task states consistently.

| AOE state | A2A task state / event | Notes |
|---|---|---|
| `created` | `submitted` | StepSession exists; not yet accepted by the worker runtime |
| `dispatching` | outbound `message/send` pending | Transient AOE state while calling the daemon |
| `dispatched` | `submitted` or `working` | Daemon accepted the request |
| `running` | `working` | Worker has started |
| `wait_for_user_input` | `input-required` | Resume by sending a follow-up message against the same A2A task/context when supported |
| `succeeded` | `completed` | AOE persists output/artifacts and advances the DAG |
| `failed` | `failed` or `rejected` | `rejected` is for non-retryable validation/auth/capability failures |
| `cancelled` | `canceled` | AOE uses A2A `tasks/cancel` where available |

### 8.2 Outbound: AOE dispatches a step

AOE dispatches through `a2a_daemon_engine`. The preferred daemon implementation maps this request to the official A2A `message/send` method with a `Message` containing text/data/file parts and a `pushNotificationConfig` pointing back to AOE. If the daemon exposes `assign_task`, that method is treated as a SilvaEngine wrapper, not the raw A2A protocol.

The canonical AOE adapter call should look conceptually like this:

```python
a2a_dispatch.dispatch_step(
  target_agent_id=step.agent_uuid,
  a2a_protocol_version="0.3.0",
  aoe_contract_version="1.0",
  message={
    "role": "user",
    "parts": [
      {"kind": "text", "text": composed_query},
      {"kind": "data", "data": {
        "predecessor_outputs": [...],
        "files": [...],
        "thread_uuid": carried_thread
      }}
    ],
    "messageId": f"{step_session_uuid}:{step_session.attempt}"
  },
  metadata={
    "step_session_uuid": step_session_uuid,
    "task_run_uuid": task_run_uuid,
    "step_id": step_id,
    "agent_uuid": step.agent_uuid,
    "user_id": task_run.user_id,
    "attempt": step_session.attempt,
    "idempotency_key": f"{step_session_uuid}:{step_session.attempt}"
  },
  push_notification={
    "url": AOE_CALLBACK_URL,
    "token": callback_token,
    "authentication": {"schemes": ["Bearer"]}
  }
)
```

The older `assign_task` shape below is retained only as the daemon wrapper shape until `docs/A2A_CONTRACT.md` is updated and code confirms the daemon API.

```python
a2a.a2a(
  action="assign_task",
  endpoint_id=partition_endpoint,
  task_id=step_session_uuid,           # AOE owns this id
  task_type="orchestration_step",
  assigned_agent_id=step.agent_uuid,   # resolved by A2A to AI Agent Core's endpoint
  input_data={
    "step_session_uuid": step_session_uuid,
    "task_run_uuid": task_run_uuid,
    "step_id": step_id,
    "agent_uuid": step.agent_uuid,
    "user_id": task_run.user_id,
    "query": composed_query,           # subtask query + templated vars
    "predecessor_outputs": [...],      # outputs of upstream steps
    "files": [...],
    "thread_uuid": carried_thread,     # optional: continuity across steps
    "attempt": step_session.attempt,
    "callback": {
      "agent_id": "ai_orchestration_engine",
      "endpoint_url": AOE_CALLBACK_URL
    }
  }
)
```

AOE marks StepSession `dispatched` on success; on A2A delivery failure (retries exhausted), `failed`.

### 8.3 Inbound: AOE receives step results

AOE receives A2A task updates from the daemon callback / push-notification path. The callback handler first validates the A2A envelope, then maps the task state and metadata back to a StepSession.

Transport-facing handlers should prefer A2A task states (`working`, `completed`, `failed`, `rejected`, `input-required`, `canceled`). Custom `step.*` messages may be used only inside the daemon wrapper and must normalize to the state mapping in §8.1.1 before touching AOE state.

AOE registers itself as an A2A agent (`agent_id="ai_orchestration_engine"`). The A2A daemon delivers worker messages to AOE's HTTP callback. AOE's callback handler dispatches by message type:

| Message type | Handler |
|---|---|
| `step.started` | mark StepSession `running`, set `started_at` |
| `step.result` | persist `output`, mark `succeeded`, advance DAG |
| `step.failure` | persist `error`, mark `failed`, evaluate retry policy |
| `step.input_required` | mark `wait_for_user_input`, persist prompt; surface via query/subscription |
| `step.heartbeat` | (optional) update `updated_at` for liveness |

All inbound messages must include `step_session_uuid`; AOE rejects messages that don't correlate to a known StepSession.

### 8.4 Agent runtime: AI Agent Core as the A2A endpoint

AI Agent Core is the recommended worker-side A2A server. It should expose an A2A-compatible endpoint and Agent Card, while the daemon owns registry/routing and can fan out by `agent_uuid`.

There is exactly one agent runtime: `ai_agent_core_engine`. Every orchestrated agent is invoked by it. From A2A's perspective, AI Agent Core is registered as an A2A agent (or a small set of A2A agents — see §14.2) whose `endpoint_url` is an HTTP route on AI Agent Core that:

1. Accepts A2A `message/send` requests, either directly or via the daemon's `assign_task` wrapper.
2. Resolves the `agent_uuid` to its agent definition (prompt, tools, model, etc.) from AI Agent Core's own data.
3. Composes the LLM input from A2A message parts + AOE metadata (`query`, `predecessor_outputs`, `files`, optional `thread_uuid`).
4. Runs the agent (LLM call, tool use, async task tracking — all existing AI Agent Core machinery).
5. Posts task updates / final artifacts back to AOE through the daemon callback / push-notification path, correlating by `step_session_uuid`.

What this means in practice:
- **AOE never calls `invoke_ask_model` GraphQL.** That coupling — the most cited pain point of `procedure_hub` — is gone.
- **Agent identity (`agent_uuid`) is preserved in metadata and/or daemon registry routing.** Agents do not need their own processes; the runtime is shared.
- **AI Agent Core gains an A2A-facing handler.** That handler is a small adapter over its existing async-task model: same prompt selection, same model invocation, same thread management — just a new entry channel and a new result channel (A2A in/out).

Implementation note: in early phases AOE may register AI Agent Core under one A2A agent_id (e.g. `ai_agent_core`) that fans out internally by `payload.agent_uuid`. Per-agent A2A registration is a later optimization (§14.2).

### 8.5 Action functions (no-A2A path)

For steps whose StepDef has `action_function` set, AOE skips A2A and runs the function in-process synchronously. The StepSession still goes through the full state machine (`created → dispatching → running → succeeded`) but `dispatched_at == started_at` and `a2a_task_id` is null. This preserves audit symmetry. Action-function steps do not pass through AI Agent Core at all.

---

## 9. Operation & Monitoring Responsibilities

The "module is the operator/monitor" rule (per the task brief) defines AOE's positive responsibilities:

### 9.1 Operate
- Accept task execution requests (GraphQL mutation `executeTask`).
- Materialize the DAG into `TaskRun.dag_state`.
- Dispatch ready steps via A2A.
- React to inbound step results: advance DAG, dispatch next ready steps.
- Apply retry policy on failures.
- Coordinate user-in-the-loop: expose pending input requests, accept user input mutations, resume affected steps.
- Cancel a task: cancel in-flight A2A tasks, mark TaskRun + remaining StepSessions `cancelled`.

### 9.2 Monitor
- **Per-step metrics:** dispatch latency, run duration, retry count, failure rate by `agent_uuid` and `step_id`.
- **Per-task metrics:** end-to-end duration, DAG depth, success rate.
- **Stuck-state detection:** periodic sweep (cron / scheduled rule) over `StepSession` GSI by status, flagging `dispatched`/`running` rows older than `timeout_seconds` and emitting `step.timeout` events that follow the failure path.
- **Structured logs:** every state transition emits a JSON log line with `task_run_uuid`, `step_session_uuid`, `agent_uuid`, `from_state`, `to_state`, `reason`. Log shape is fixed.
- **Queries for ops:** GraphQL queries for `taskRunList(status, since)`, `stepSessionList(taskRunUuid)`, `stuckStepList(olderThanSeconds)`, `agentExecutionStats(agentUuid, window)`.

### 9.3 Non-responsibilities

To keep boundaries crisp, AOE is **not** responsible for:
- Invoking LLMs (workers do).
- Storing or rotating prompts/agent configs (AI Agent Core / agent definitions own that).
- Multi-agent message routing between *non-orchestrated* agents (A2A daemon does that).
- Long-running session memory / threads (AI Agent Core does that; AOE just stores the link in `StepRun`).

---

## 10. Module Layout

Following the standard SilvaEngine pattern:

```
ai_orchestration_engine/
├── ai_orchestration_engine/
│   ├── __init__.py
│   ├── __main__.py
│   ├── main.py                       # Engine class, GraphQL handler entry
│   ├── schema.py                     # Graphene Schema, root Query/Mutation
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── config.py                 # Settings, A2A endpoint, AI Core endpoint, timeouts
│   │   ├── task_run_hub/
│   │   │   ├── __init__.py
│   │   │   ├── task_run.py           # executeTask mutation, dag materialization
│   │   │   ├── step_dispatcher.py    # picks ready steps, calls A2A
│   │   │   ├── step_callback.py      # inbound A2A handler (step.result/failure/...)
│   │   │   ├── action_function.py    # in-process function steps
│   │   │   ├── user_in_the_loop.py   # input-required + executeForUserInput
│   │   │   └── retry_policy.py       # backoff/attempt logic
│   │   ├── monitor/
│   │   │   ├── __init__.py
│   │   │   ├── stuck_sweep.py        # periodic timeout detector
│   │   │   └── metrics.py            # event emission
│   │   └── a2a_client.py             # thin wrapper around A2A daemon RPC
│   ├── models/
│   │   ├── __init__.py
│   │   ├── procedure.py
│   │   ├── task.py
│   │   ├── task_run.py
│   │   ├── step_session.py
│   │   ├── step_run.py
│   │   ├── cache.py
│   │   └── utils.py
│   ├── types/                        # Graphene ObjectTypes
│   ├── queries/
│   ├── mutations/
│   ├── utils/
│   └── tests/
├── docs/
│   ├── DEVELOPMENT_PLAN.md           # this file
│   ├── A2A_CONTRACT.md             # A2A message contract
│   ├── SEQUENCE_DIAGRAMS.md          # Mermaid diagrams
│   ├── LOCAL_DEVELOPMENT.md          # Setup guide
│   └── MIGRATION_FROM_PROCEDURE_HUB.md  # (Phase 6)
├── pyproject.toml
├── README.md
└── LICENSE
```

Key naming choice: the top-level handler family is `task_run_hub` (mirrors the role `procedure_hub` plays today, but reframed around per-step sessions). The `monitor/` package is a sibling, not a sub-handler — it operates *across* TaskRuns.

---

## 11. Phased Delivery Plan

The plan is organized so that each phase ends with something runnable end-to-end. **Phase 1 ships a working single-step orchestration through A2A**; later phases add DAG, retries, user-in-the-loop, etc.

### Phase 0 — Foundation (1 week)

Goal: Engine scaffold runs and serves an empty GraphQL schema.

- Create `pyproject.toml`, package skeleton, `main.py` with `ai_orchestration_graphql` entry point matching SilvaEngine convention.
- `handlers/config.py` with settings: A2A daemon endpoint, AI Agent Core endpoint, default timeouts, `partition_key` assembly.
- Empty `schema.py` with placeholder Query/Mutation.
- Smoke test: schema introspection returns.
- Deployment: confirm it builds as a Lambda layer + handler in the same way as other engines.

Exit: `python -m ai_orchestration_engine` returns a healthy schema; CI green.

### Phase 0.5 — Local Development Environment (0.5 week)

Goal: Developers can run AOE locally with mocked dependencies.

- `docker-compose.yml`: LocalStack (DynamoDB), Redis (cache), mock A2A daemon (Python Flask/FastAPI wrapper)
- `Makefile` targets: `make dev-up`, `make dev-down`, `make test`
- Local config override: `config.local.yaml` with localhost endpoints
- Test fixtures: DynamoDB table creation, seed data loader
- Documentation: `docs/LOCAL_DEVELOPMENT.md`

Exit: `make dev-up && make test` passes all unit tests without AWS credentials.

### Phase 0.75 - A2A Daemon Compatibility Spike (0.5 week)

Goal: Confirm the exact A2A server/client surface before implementation depends on it.

- Verify whether `a2a_daemon_engine` exposes official A2A `message/send` / `tasks/*` directly or only a SilvaEngine wrapper such as `assign_task`.
- Produce an adapter decision record: direct A2A client, daemon wrapper client, or both behind `a2a_client.py`.
- Confirm AI Agent Core's required server surface: Agent Card, `message/send`, task state updates, artifact output, push notification callback.
- Update `docs/A2A_CONTRACT.md` with the confirmed mapping and version.

Exit: A minimal echo task can be submitted through the chosen daemon API and correlated back to a local AOE callback by `step_session_uuid`.

### Phase 1 — Single-step Orchestration (3 weeks, broken into sub-phases)

**Phase 1a: AOE Scaffold with Models (1 week)**

Goal: Complete data layer and GraphQL schema for single-step tasks.

- Models: `Procedure`, `Task` (StepDef list of size 1), `TaskRun`, `StepSession`, `StepRun`
- Mutations: `insertUpdateProcedure`, `insertUpdateTask`, `executeTask` (synchronous, mock dispatch)
- `executeTask` accepts optional `idempotency_key`; repeated calls with the same tenant/task/key return the existing TaskRun.
- Queries: `procedure`, `task`, `taskRun`, `stepSession`, `stepSessionListByTaskRun`
- In-memory mock dispatcher: StepSession created but no actual A2A call
- Unit tests on models and mutations per §6.1 (concrete coverage targets set after the Phase 1c baseline amendment)

Exit: Can create Procedure/Task/TaskRun/StepSession via GraphQL; `executeTask` returns `task_run_uuid` with mocked `StepSession.status=created`.

**Phase 1b: A2A Integration (1 week)**

Goal: AOE dispatches to A2A daemon and receives callbacks.

- `step_dispatcher`: Call the AOE A2A adapter. The adapter uses official A2A `message/send` when available and otherwise uses the daemon's `assign_task` wrapper.
- `step_callback`: HTTP endpoint, register with mock A2A daemon
- Handle A2A task states plus normalized wrapper events (`step.result` / `step.failure` only inside the wrapper)
- Update StepSession status: `created → dispatching → dispatched → running → succeeded/failed`
- Test fixture: Echo worker (A2A-registered) that returns input as output

Exit: Echo worker task completes end-to-end: AOE → A2A → Echo Worker → AOE callback → TaskRun `succeeded`.

**Phase 1c: AI Agent Core A2A Handler (1 week)**

Goal: Real agent execution through A2A → AI Agent Core.

- **AI Agent Core A2A handler** — small adapter on `ai_agent_core_engine`:
  - Register with the daemon / A2A registry as the worker-side A2A server endpoint
  - Accept A2A `message/send` payloads, or the daemon wrapper payload if Phase 0.75 confirms that path
  - Resolve `payload.agent_uuid` to agent definition
  - Run through existing `invoke_ask_model` code path
  - Post result/failure back to AOE via A2A
- Cross-repo coordination: PRs in both repos, integration testing
- Feature flag: `aoe.phase1c.enabled` (default: false)

Exit: Calling `executeTask` with real `agent_uuid` routes through A2A → AI Agent Core → LLM invocation → result via A2A → TaskRun `succeeded`. Baseline captured per §6.3, and performance passes the interim regression gate defined there.

### Phase 2 — DAG with predecessors and parallelism (1 week)

Goal: Multi-step DAG support.

- Generalize `step_dispatcher` to materialize `dag_state` and dispatch *all* steps with `in_degree==0` in parallel.
- `step_callback` updates `dag_state` with conditional writes; finds and dispatches newly-ready successors.
- Predecessor outputs threaded into successor inputs via AOE-side composition (StepSession stores predecessor list snapshot in `inputs`).
- Implement `StepDef.output_aggregator` for fan-in composition; default aggregator passes predecessor outputs as an ordered list.
- Tests: linear chain, fan-out, fan-in, diamond.

Exit: A 5-step diamond DAG with the echo worker completes in the right order with parallel sibling steps.

### Phase 2.5 — Decomposer Agent Integration (1 week)

Goal: Support dynamic task decomposition before execution.

- Add `decomposer_step` to `Task.step_definitions` with `agent_type=decomposer`
- Pre-execution phase: If Task has decomposer, run it first via A2A → AI Agent Core
- Decomposer output: JSON array of `subtask_queries` (saved to `TaskRun.subtask_queries`)
- Materialize runtime StepDefs into `TaskRun.materialized_step_definitions` (may use template substitution)
- Resume standard DAG execution with materialized steps
- Fallback: If decomposition fails, mark TaskRun `failed` with error details

Exit: A Task with decomposer produces dynamic StepDefs, executes them as a DAG, and aggregates results.

### Phase 3 — Failure handling and retries (0.5 week)

- `retry_policy.py`: per-step `max_attempts` and `backoff_seconds`.
- `step.failure` handling consults retry policy, possibly re-dispatches.
- TaskRun fails fast when a non-`primary_path` step fails after retries (preserves procedure_hub semantics).
- Idempotency: workers may safely receive duplicate dispatch calls (idempotency key = `step_session_uuid:attempt`).

Exit: A flaky worker that fails N times then succeeds drives a TaskRun to `succeeded`; persistent failure marks TaskRun `failed`.

### Phase 4 — Action functions (0.5 week)

- `action_function.py`: synchronous in-process execution path that bypasses A2A but maintains StepSession state machine.
- Same S3-backed dynamic loading as ai_coordination_engine, OR a registry of explicitly imported functions in config (decision in §14).

Exit: A Task mixing AI worker steps and action_function steps runs end-to-end.

### Phase 5 — User-in-the-loop (1 week)

- StepSession state `wait_for_user_input`, mapped from A2A task state `input-required` or the daemon wrapper's normalized input-required event.
- Mutation `executeForUserInput(step_session_uuid, user_input)`.
- Resume path: re-dispatch StepSession with augmented input as a *new attempt* (StepRun increments).
- Query for pending-input steps in a TaskRun.

Exit: A two-step task where step 1 asks for clarification and step 2 uses the user's answer completes successfully.

### Phase 6 — Coexistence and migration (1 week)

(Note: Reordered before Phase 7. The AI Agent Core A2A handler from Phase 1c is the only invocation path. This phase is strictly about migrating existing coordination workloads.)

- Migration tooling: copy a `Coordination` + `Task` from ai_coordination_engine into AOE as a `Procedure` + `Task`, mapping `agent_actions` → `step_definitions` (predecessors, primary_path, user_in_the_loop, action_function carry over 1:1).
- Feature flag per Task: `executeWith ∈ {procedure_hub, ai_orchestration_engine}` so callers (or a router layer) can route a task to either engine during migration.
- Parallel-run mode (off by default) for selected tasks: run the same task through both engines and diff outputs into a comparison report.
- Compatibility shim for existing callers of `executeProcedureTaskSession` GraphQL (optional): translate to `executeTask` against AOE when the per-task flag is set, so client code is unaffected.
- Migration mapping for `StepDef.output_aggregator`: preserve existing procedure_hub fan-in behavior with registered aggregators and include differences in the parallel-run report.

Exit: An existing coordination Task runs through AOE producing output equivalent to `procedure_hub` on the same input, with diffs reported and reviewed for at least 3 representative tasks.

### Phase 7 — Monitoring & observability (1 week)

- `monitor/stuck_sweep.py`: scheduled rule (CloudWatch event) that scans StepSession GSI by status and emits `step.timeout` failures.
- Structured event emission on every state transition (CloudWatch Logs Insights query examples committed).
- Queries: `taskRunList`, `stuckStepList`, `agentExecutionStats`.
- Optional: GraphQL subscription for live TaskRun progress.
- **Monitoring of migrated tasks**: Compare metrics between engines, validate SLOs hold

Exit: Ops can answer "what's stuck", "what's slow", "which agent is failing" from GraphQL; migrated tasks show no metric regressions.

### Phase 8 — Production hardening (open-ended)

(Optimistic versioning on `TaskRun` lands earlier — see §6.4 / Phase 2 — and is not deferred to this phase.)

- Cache layer (HybridCacheEngine) for hot reads of `Procedure`/`Task`.
- DataLoader batch loaders for nested GraphQL fields.
- Authz hooks (JWT propagation matching knowledge_graph_engine pattern).
- Complete parallel-run diff harness with automated comparison reports.

---

## 12. Coexistence with `procedure_hub`

Both engines target the same multi-tenant DynamoDB cluster but use **separate tables** (`aoe-procedures`, `aoe-tasks`, `aoe-task-runs`, `aoe-step-sessions`, `aoe-step-runs`). They do not share rows. Migration is data copy + re-execution, not in-place schema migration.

Routing during migration:
- New tasks may be created in either engine.
- A "router" thin layer (in AOE or in the calling app) decides which engine executes a given task based on a per-task config flag.
- Both engines emit metrics into the same observability backend, tagged with engine name.

Decommissioning of `procedure_hub` is out of scope for this plan. It happens after AOE owns >80% of traffic with no regressions for ≥30 days.

---

## 13. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| A2A daemon becomes a SPOF for orchestration. | Rely on A2A's existing retry + persistent task store; AOE retries on dispatch failure; stuck-sweep catches lost messages. |
| Per-step Lambda overhead exceeds in-process listener overhead for short tasks. | Action-function path stays in-process. Measure end-to-end latency in Phase 1c; revisit if it breaches the regression gate in §6.3. |
| AI Agent Core's A2A handler diverges from the documented payload contract. | Single owner for the handler in `ai_agent_core_engine`; payload schema documented in `docs/A2A_CONTRACT.md`; AOE callback validates inbound messages and rejects mismatched shapes with a structured error. Contract test in CI exercises the round trip on every change. |
| Race between callback handlers for sibling steps writing the same `dag_state`. | Conditional writes with optimistic version on `TaskRun`. Retry on conflict (bounded). |
| Migration mistakes silently change task behavior. | Phase 6 parallel-run mode + diff harness on selected tasks before cutover. |
| `partition_key` adoption gets out of sync with ai_coordination_engine's migration. | AOE adopts it from day one; AOE never reads coordination tables, so divergence is contained. |

---

## 14. Open Questions / Decisions to Make

These do not block Phase 0/1 but should be resolved before the corresponding phase ends.

1. **Action function loading** (Phase 4). S3-dynamic-import like ai_coordination_engine, or registry of pre-imported functions in config? S3 dynamic is more flexible; registry is safer and faster. **Recommendation: registry.**
2. **AI Agent Core's A2A registration granularity** (Phase 1, see §8.4). Three options: (a) one A2A agent_id `ai_agent_core` that fans out internally by `payload.agent_uuid`; (b) one A2A agent_id per orchestrated `agent_uuid`, all pointing to the same AI Agent Core endpoint; (c) per-tenant variants of either. Option (a) is the smallest change; option (b) makes A2A's `find_agent` and capability-matching usable. **Recommendation: start with (a) in Phase 1; revisit before Phase 6 (Migration) once we know whether capability-based dispatch is needed.**
3. **AOE's A2A identity** (Phase 1). Single agent `ai_orchestration_engine` per environment, or per-tenant agents (`ai_orchestration_engine#{partition_key}`)? Per-tenant simplifies multi-tenant routing on A2A side but inflates the registry. **Recommendation: single global agent; tenant identified by message payload.**
4. ~~Decomposer agent~~ **RESOLVED** - Implemented as Phase 2.5
5. **Cancellation propagation** (Phase 3 or later). When a TaskRun is cancelled, do we wait for in-flight steps to finish, or send A2A `task.cancel` to each? **Recommendation: send `task.cancel` to A2A (which forwards to AI Agent Core) and mark StepSessions `cancelled`; AI Agent Core should treat cancel as best-effort and abort the underlying async task where possible.**
6. ~~Final output computation~~ **RESOLVED** - Implemented as part of Phase 6 (migration) with aggregator interface
7. ~~Idempotency of `executeTask`~~ **RESOLVED** - Implement in Phase 1a with a caller-provided `idempotency_key`.
8. **Subscription transport.** GraphQL subscriptions over WebSocket like knowledge_graph_engine, or polling queries only? **Recommendation: polling queries first; WebSocket only if a UI needs it.**

---

## 15. Out of Scope

- Procedure/task authoring UI.
- Intelligent / LLM-based step routing (selecting agents dynamically by capability). A2A daemon already supports `find_agent`; AOE could integrate later, but every phase in this plan uses explicit `agent_uuid` per StepDef.
- Cross-task scheduling / priority queues / quotas.
- Cost tracking per agent / per task.
- Multi-region deployment.

---

## 16. Glossary

- **Procedure**: workflow blueprint listing the agents involved.
- **Task**: workflow definition keyed to a procedure; declares the DAG.
- **TaskRun**: one execution instance of a Task.
- **StepDef**: declaration of one node in the Task DAG.
- **StepSession**: per-step execution session — the unit of orchestration in AOE.
- **StepRun**: per-attempt audit record under a StepSession.
- **AOE**: AI Orchestration Engine (this module).
- **A2A daemon**: a2a_daemon_engine, agent-to-agent message router.
- **Agent runtime**: `ai_agent_core_engine`, the single A2A-registered service that actually invokes any orchestrated agent (LLM call, tool use, thread management). Reached only via A2A from AOE.
