# Agent Workspace Format (AWF)

A filesystem-backed workspace format for inspecting, debugging, and intervening in multi-agent workflow execution.

AWF is a lightweight convention and integration layer for multi-agent systems. It lets agents and workflow runtimes write structured artifacts such as events, plans, decisions, staged diffs, tool calls, questions, and outputs into a predictable workspace on disk.

The goal is simple: make agent work visible, inspectable, interruptible, and resumable without requiring a database, a new workflow engine, or a specific UI framework.

---

## 1. What AWF Solves

Modern multi-agent systems often run as a stream of hidden actions:

- agents call tools
- agents make decisions
- agents modify or propose files
- agents hand work to other agents
- agents block on missing information
- orchestration happens across parallel tasks

Without a shared workspace format, humans usually see only logs, final output, or a chat transcript. That is not enough for serious workflow editing, debugging, or governance.

AWF creates a durable, structured workspace for every run and every agent.

It gives a workflow editor a reliable source for:

- live timelines
- agent swimlanes
- staged diffs
- decision logs
- tool call history
- human question inboxes
- run manifests
- agent status
- generated artifacts

AWF should be treated as an **observer, persistence, and inspection layer** for an existing workflow runtime.

It is not a replacement for your workflow engine.

---

## 2. Design Principles

### Filesystem-first

AWF stores runtime state under `.agents/`. This makes it easy to inspect with normal tools, commit sample workspaces, archive runs, and debug without infrastructure.

### Append-only history

Each agent writes an append-only `events.jsonl` file. Every line is one event. Past event lines are never rewritten.

### Human inspectability

Plans, decisions, questions, staged diffs, and artifacts are stored as simple files. A developer or operator can inspect them without needing a custom database client.

### Safe by default

Agent file edits are staged as patch files. AWF does not apply diffs to the host repo by default.

### Framework-neutral

AWF does not assume Next.js, React, Svelte, Tauri, or any specific UI framework. It can be integrated into an existing workflow editor.

### Schema-first

AWF events are defined with Zod. The event schema is the source of truth. TypeScript types are inferred from schemas rather than duplicated by hand.

### Minimal infrastructure

AWF v0 does not require a database, ORM, authentication, multi-user collaboration, or replay engine.

---

## 3. Conceptual Model

AWF has three core concepts:

1. **Run**  
   A single workflow execution.

2. **Agent workspace**  
   A per-agent directory containing that agent's manifest, events, plan, decisions, questions, artifacts, scratch files, and staged diffs.

3. **Event log**  
   An append-only JSONL stream that records what happened.

A workflow run may involve one orchestrator and many child agents, or it may map to an existing graph of nodes, workers, tools, or tasks.

Recommended mapping:

| Existing workflow concept | AWF concept |
|---|---|
| Workflow execution | Run |
| Agent / node / task / worker | Agent workspace |
| Runtime lifecycle event | AWF event |
| Tool invocation | `tool.call` |
| Tool response | `tool.result` |
| Human prompt | `question.ask` |
| Human response | `question.answer` |
| Proposed file modification | `file.edit` + `diff.staged` |
| Generated output | `artifact.create` |
| Planning update | `plan.set` or `plan.update` |
| Reasoning checkpoint | `decision` or `note` |
| Pause/resume/cancel/edit | `intervention` |

---

## 4. Workspace Layout

AWF writes runtime data under `.agents/`.

```text
.agents/
  manifest.json
  shared/
  agents/
    <agentId>/
      .agent/
        manifest.json
        events.jsonl
        plan.md
        decisions.md
        questions/
          <questionId>.json
        artifacts/
        scratch/
        diffs/
          <NNNN>-<slug>.patch
```

### `.agents/manifest.json`

The root manifest describes the run-level state.

It should include:

```json
{
  "runId": "run_01J00000000000000000000000",
  "goal": "Implement checkout workflow",
  "status": "running",
  "createdAt": "2026-05-14T04:00:00.000Z",
  "updatedAt": "2026-05-14T04:01:00.000Z",
  "agents": [
    {
      "agentId": "orchestrator",
      "path": "agents/orchestrator/.agent",
      "status": "running"
    }
  ],
  "metadata": {}
}
```

Recommended statuses:

```text
running | success | failure | cancelled
```

### `.agents/shared/`

Read-mostly shared context for the run.

Examples:

- shared requirements
- global constraints
- user-provided context
- environment metadata
- workflow-level notes

For v0, this folder can remain minimal.

### `.agents/agents/<agentId>/.agent/manifest.json`

The agent manifest describes one participant in the workflow.

Example:

```json
{
  "runId": "run_01J00000000000000000000000",
  "agentId": "tests",
  "parentAgentId": "orchestrator",
  "goal": "Add workflow runtime tests",
  "role": "test-engineer",
  "status": "blocked",
  "createdAt": "2026-05-14T04:00:03.000Z",
  "updatedAt": "2026-05-14T04:00:10.000Z",
  "startedAt": "2026-05-14T04:00:03.000Z",
  "endedAt": null,
  "metadata": {}
}
```

Recommended statuses:

```text
idle | running | blocked | success | failure | cancelled
```

### `.agent/events.jsonl`

Append-only canonical event history for the agent.

Each line is one event:

```json
{"id":"evt_01J00000000000000000000000","ts":"2026-05-14T04:00:01.000Z","runId":"run_01J00000000000000000000000","agentId":"orchestrator","parentEventId":null,"type":"run.start","payload":{"goal":"Implement checkout workflow","model":"gpt-5.5-thinking"}}
```

Rules:

- one JSON object per line
- never rewrite old lines
- append using one buffered write per event line
- tolerate malformed lines in readers by reporting diagnostics instead of crashing

### `.agent/plan.md`

Current plan for the agent.

This file is human-readable and may be human-editable if the UI supports plan editing.

Example:

```md
# Plan

1. Inspect the workflow runtime.
2. Identify tool call boundaries.
3. Add observer hooks.
4. Record AWF events.
5. Add snapshot UI.
```

### `.agent/decisions.md`

Narrative decision log.

This is useful for humans who do not want to reconstruct decisions from raw event streams.

Example:

```md
## 2026-05-14T04:05:00.000Z — Use runtime observer instead of direct calls

The workflow engine already exposes node lifecycle hooks. AWF should attach as an observer instead of scattering writes through business logic.

Alternatives considered:
- write directly from every node
- introduce a new workflow engine
```

### `.agent/questions/<questionId>.json`

Pending or answered human-in-the-loop question.

Example pending question:

```json
{
  "questionId": "q_01J00000000000000000000000",
  "runId": "run_01J00000000000000000000000",
  "agentId": "tests",
  "status": "pending",
  "prompt": "Should I mock the agent runtime or use an in-memory workflow fixture?",
  "options": ["mock-runtime", "in-memory-fixture"],
  "blocking": true,
  "createdAt": "2026-05-14T04:10:00.000Z"
}
```

Example answered question:

```json
{
  "questionId": "q_01J00000000000000000000000",
  "runId": "run_01J00000000000000000000000",
  "agentId": "tests",
  "status": "answered",
  "prompt": "Should I mock the agent runtime or use an in-memory workflow fixture?",
  "options": ["mock-runtime", "in-memory-fixture"],
  "blocking": true,
  "createdAt": "2026-05-14T04:10:00.000Z",
  "answer": "in-memory-fixture",
  "answeredAt": "2026-05-14T04:11:00.000Z",
  "answeredBy": "human"
}
```

### `.agent/artifacts/`

Final or durable outputs produced by the agent.

Examples:

- generated documentation
- generated code files
- JSON analysis reports
- images or diagrams
- workflow summaries

Artifacts should be referenced by `artifact.create` events.

### `.agent/scratch/`

Temporary intermediate files.

Scratch files are safe to delete.

Examples:

- raw tool output
- intermediate parse results
- temporary notes
- partial generated data

### `.agent/diffs/`

Staged patch files.

Example:

```text
.agent/diffs/0001-add-awf-observer.patch
```

Rules:

- diffs are proposals only
- do not apply them automatically
- do not write directly to host project files as part of AWF runtime
- reference staged patches via `file.edit` and `diff.staged` events

---

## 5. Event Model

Every AWF event has a common envelope:

```ts
{
  id: string;
  ts: string;
  runId: string;
  agentId: string;
  parentEventId: string | null;
  type: string;
  payload: object;
}
```

### Event ID format

```text
evt_<26-character ULID>
```

Example:

```text
evt_01J00000000000000000000000
```

### Run ID format

```text
run_<26-character ULID>
```

Example:

```text
run_01J00000000000000000000000
```

### Timestamp format

Use ISO 8601 datetime strings with offset.

Example:

```text
2026-05-14T04:00:00.000Z
```

### Parent event ID

`parentEventId` links one event to a prior event when useful.

Examples:

- a `tool.result` can point to the corresponding `tool.call`
- a `question.answer` can point to the `question.ask`
- an `agent.spawn` can point to the event that caused the spawn

For v0, `parentEventId` may be `null` where correlation is not available.

---

## 6. Event Types

AWF v0 defines five event groups.

### Lifecycle events

| Type | Purpose |
|---|---|
| `run.start` | A workflow run started |
| `run.end` | A workflow run completed, failed, or was cancelled |
| `agent.spawn` | An agent created or delegated work to another agent |
| `agent.handoff` | One agent handed context to another agent |

Example `run.start`:

```json
{
  "id": "evt_01J00000000000000000000000",
  "ts": "2026-05-14T04:00:00.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "orchestrator",
  "parentEventId": null,
  "type": "run.start",
  "payload": {
    "goal": "Implement AWF integration",
    "model": "gpt-5.5-thinking"
  }
}
```

Example `agent.spawn`:

```json
{
  "id": "evt_01J00000000000000000000001",
  "ts": "2026-05-14T04:00:01.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "orchestrator",
  "parentEventId": null,
  "type": "agent.spawn",
  "payload": {
    "childAgentId": "tests",
    "goal": "Add workflow tests"
  }
}
```

### Reasoning events

| Type | Purpose |
|---|---|
| `plan.set` | Initial or full plan written |
| `plan.update` | Plan changed using a diff |
| `decision` | Agent made a notable decision |
| `note` | Agent wrote a lightweight observation |

Example `decision`:

```json
{
  "id": "evt_01J00000000000000000000002",
  "ts": "2026-05-14T04:02:00.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "runtime-integration",
  "parentEventId": null,
  "type": "decision",
  "payload": {
    "title": "Use workflow observer hooks",
    "rationale": "The current runtime already exposes lifecycle hooks, so AWF can be added without coupling it to business logic.",
    "alternatives": ["manual writes from every task", "replace runtime"]
  }
}
```

### Action events

| Type | Purpose |
|---|---|
| `tool.call` | Agent called a tool |
| `tool.result` | Tool returned success or failure |
| `file.write` | Agent wrote a file under artifacts or scratch |
| `file.edit` | Agent proposed an edit as a staged diff |
| `file.delete` | Agent proposed or recorded a delete operation |

Example `tool.call`:

```json
{
  "id": "evt_01J00000000000000000000003",
  "ts": "2026-05-14T04:03:00.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "runtime-integration",
  "parentEventId": null,
  "type": "tool.call",
  "payload": {
    "callId": "call_01J00000000000000000000000",
    "tool": "grep",
    "args": {
      "pattern": "WorkflowRuntime",
      "path": "src"
    }
  }
}
```

Example `tool.result`:

```json
{
  "id": "evt_01J00000000000000000000004",
  "ts": "2026-05-14T04:03:01.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "runtime-integration",
  "parentEventId": "evt_01J00000000000000000000003",
  "type": "tool.result",
  "payload": {
    "callId": "call_01J00000000000000000000000",
    "ok": true,
    "durationMs": 120,
    "output": ["src/runtime/workflow-runtime.ts"]
  }
}
```

### Output events

| Type | Purpose |
|---|---|
| `artifact.create` | Agent created a final or durable output |
| `diff.staged` | Agent staged a patch file |

Example `diff.staged`:

```json
{
  "id": "evt_01J00000000000000000000005",
  "ts": "2026-05-14T04:05:00.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "runtime-integration",
  "parentEventId": null,
  "type": "diff.staged",
  "payload": {
    "patchPath": "diffs/0001-add-awf-observer.patch",
    "summary": "Add AWF observer hook to workflow runtime",
    "filesChanged": 2
  }
}
```

### Human-in-the-loop events

| Type | Purpose |
|---|---|
| `question.ask` | Agent asked a human or policy layer a question |
| `question.answer` | Question was answered |
| `intervention` | Human/system paused, resumed, cancelled, or edited plan |

Example `question.ask`:

```json
{
  "id": "evt_01J00000000000000000000006",
  "ts": "2026-05-14T04:06:00.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "tests",
  "parentEventId": null,
  "type": "question.ask",
  "payload": {
    "questionId": "q_01J00000000000000000000000",
    "prompt": "Should I mock the agent runtime or use an in-memory workflow fixture?",
    "options": ["mock-runtime", "in-memory-fixture"],
    "blocking": true
  }
}
```

Example `question.answer`:

```json
{
  "id": "evt_01J00000000000000000000007",
  "ts": "2026-05-14T04:07:00.000Z",
  "runId": "run_01J00000000000000000000000",
  "agentId": "tests",
  "parentEventId": "evt_01J00000000000000000000006",
  "type": "question.answer",
  "payload": {
    "questionId": "q_01J00000000000000000000000",
    "answer": "in-memory-fixture",
    "answeredBy": "human"
  }
}
```

---

## 7. Writer API

The AWF writer is the runtime-facing API used to create workspaces, append events, write questions, stage diffs, and update manifests.

The exact API can be adapted to the host project, but the conceptual shape should look like this:

```ts
const awf = await AwfWorkspace.openOrCreate(".agents", {
  runId,
  goal: "Implement AWF integration",
});

const orchestrator = awf.agent("orchestrator");
await orchestrator.start("Coordinate implementation");

const tests = await orchestrator.spawn("tests", "Add workflow tests");

const call = tests.tool("grep", {
  pattern: "WorkflowRuntime",
  path: "src",
});

await call.finish({
  ok: true,
  output: ["src/runtime/workflow-runtime.ts"],
});

await tests.decide({
  title: "Use in-memory fixture",
  rationale: "It exercises the runtime behavior without mocking the observer boundary.",
  alternatives: ["mock runtime", "full external process"],
});

const answer = await tests.askQuestion(
  "Should I mock the agent runtime or use an in-memory workflow fixture?",
  {
    options: ["mock-runtime", "in-memory-fixture"],
    blocking: true,
  }
);
```

### Recommended writer capabilities

| Method | Behavior |
|---|---|
| `openOrCreate(root, options)` | Creates or opens `.agents/` |
| `agent(agentId)` | Returns an agent workspace handle |
| `start(goal)` | Updates manifest and emits lifecycle event |
| `spawn(childAgentId, goal)` | Creates child workspace and emits `agent.spawn` |
| `handoff(toAgentId, context)` | Emits `agent.handoff` |
| `setPlan(markdown)` | Writes `plan.md` and emits `plan.set` |
| `updatePlan(markdown)` | Writes `plan.md` and emits `plan.update` |
| `decide(input)` | Emits `decision` and appends `decisions.md` |
| `note(text)` | Emits `note` |
| `tool(name, args)` | Emits `tool.call`; returns tool call handle |
| `finish(result)` | Emits `tool.result` |
| `writeArtifact(path, content, kind)` | Writes artifact and emits `artifact.create` |
| `writeScratch(path, content)` | Writes scratch/intermediate file |
| `editFile(path, oldText, newText)` | Stages patch and emits `file.edit` + `diff.staged` |
| `askQuestion(prompt, options)` | Writes question file, emits `question.ask`, waits for answer if blocking |
| `end(status, summary)` | Emits `run.end` or marks agent completed |

---

## 8. Integration with an Existing Workflow Runtime

AWF should be integrated at the runtime boundary, preferably through existing hooks, middleware, plugins, observers, or event sinks.

Avoid scattering AWF writes throughout unrelated business logic.

### Preferred integration pattern

```text
Workflow Runtime
      │
      ▼
Runtime Observer / Event Sink
      │
      ▼
AWF Writer
      │
      ▼
.agents/ filesystem workspace
      │
      ▼
Workflow Editor UI Snapshot
```

### Runtime event mapping

| Runtime event | AWF write |
|---|---|
| workflow started | create run, emit `run.start` |
| workflow completed | emit `run.end`, update manifest |
| node/agent started | create agent workspace, update manifest |
| child task created | emit `agent.spawn` |
| task delegated | emit `agent.handoff` |
| tool started | emit `tool.call` |
| tool completed | emit `tool.result` |
| plan changed | write `plan.md`, emit `plan.set` or `plan.update` |
| decision made | emit `decision`, append `decisions.md` |
| loggable observation | emit `note` |
| proposed file change | stage diff, emit `file.edit` and `diff.staged` |
| output created | write artifact, emit `artifact.create` |
| human input required | write question, emit `question.ask`, block if required |
| human input answered | update question, emit `question.answer`, resume |
| user pause/resume/cancel | emit `intervention` |

---

## 9. Human-in-the-Loop Flow

AWF models human intervention through question files and events.

### Blocking question flow

```text
Agent needs input
  │
  ▼
Writer creates questions/<id>.json with status=pending
  │
  ▼
Writer emits question.ask
  │
  ▼
Agent status becomes blocked
  │
  ▼
UI shows pending question
  │
  ▼
Human submits answer
  │
  ▼
UI updates question JSON with answer/status=answered
  │
  ▼
Writer detects answer
  │
  ▼
Writer emits question.answer
  │
  ▼
Agent resumes
```

### UI write rules

The UI should only write:

1. answers to `questions/<id>.json`
2. human edits to `plan.md`, if supported

When answering a question, preserve existing fields and add or update:

```json
{
  "status": "answered",
  "answer": "in-memory-fixture",
  "answeredAt": "2026-05-14T04:07:00.000Z",
  "answeredBy": "human"
}
```

The writer should detect this update and emit a `question.answer` event.

---

## 10. Snapshot Reader

The snapshot reader converts `.agents/` into a UI-friendly structure.

It should return:

- root manifest
- agent manifests
- parsed events
- invalid event diagnostics
- plans
- decisions
- questions
- pending questions
- diffs
- artifacts metadata
- derived timeline
- derived swimlanes

Example snapshot shape:

```ts
interface AwfSnapshot {
  root: RunManifest | null;
  agents: AgentSnapshot[];
  timeline: Event[];
  swimlanes: Swimlane[];
  pendingQuestions: QuestionFile[];
  diagnostics: SnapshotDiagnostic[];
}

interface AgentSnapshot {
  agentId: string;
  manifest: AgentManifest | null;
  events: Event[];
  planMarkdown: string | null;
  decisionsMarkdown: string | null;
  questions: QuestionFile[];
  diffs: DiffSnapshot[];
  artifacts: ArtifactSnapshot[];
}
```

Reader behavior:

- if `.agents/` is missing, return an empty snapshot
- if an agent folder is incomplete, include diagnostics but continue
- if an event line is malformed, skip it and record a diagnostic
- sort global timeline by timestamp
- group swimlanes by `agentId`
- keep large patch/artifact content optional or truncated

---

## 11. Workflow Editor UI Requirements

The existing workflow editor should read AWF snapshots and display them in the current UI framework.

Do not build a separate viewer unless the project has no existing editor surface.

### Required panels

| Panel | Purpose |
|---|---|
| Run Summary | Show run ID, goal, status, timing |
| Agent Swimlanes | Show agents/nodes/tasks and status over time |
| Timeline | Show events sorted by timestamp |
| Decisions | Show decision log per agent |
| Diffs | Show staged patches |
| Questions | Show pending and answered questions |
| Artifacts | Show generated outputs, if useful |

### Timeline display

Recommended event highlighting:

| Event | UI treatment |
|---|---|
| `run.start` | run started marker |
| `agent.spawn` | child agent/node created |
| `tool.call` | tool started |
| `tool.result` | tool success/failure |
| `decision` | important reasoning checkpoint |
| `diff.staged` | proposed code/file change |
| `question.ask` | blocking/non-blocking question |
| `question.answer` | human/policy response |
| `run.end` | run completion marker |

### Live updates

Use the existing project mechanism:

- existing API route
- existing backend service
- existing Tauri command/event bridge
- existing WebSocket
- existing SSE
- filesystem watcher
- polling

For v0, polling every one second is acceptable.

---

## 12. Safety and Integrity Rules

### Do not write runtime edits to host files

`file.edit` records a proposed change. It should not modify the host repo.

The actual patch goes under:

```text
.agent/diffs/<NNNN>-<slug>.patch
```

### Prevent path traversal

Any writer method that accepts paths must reject unsafe paths such as:

```text
../outside-project
/absolute/path
.agent/../../secret
```

### Append events atomically

Use one append operation per JSONL line.

Recommended pattern:

```ts
await fs.appendFile(eventsPath, JSON.stringify(event) + "\n", "utf8");
```

### Write JSON safely

For manifests and question files, prefer temp-file-and-rename:

```text
manifest.json.tmp -> manifest.json
```

### Reader resilience

Readers should be tolerant. They should not crash just because one event line or question file is malformed.

---

## 13. Recommended Implementation Modules

The physical structure should adapt to the existing project.

### If the project is a monorepo

```text
packages/
  schema/
    src/
      events.ts
      ids.ts
      manifest.ts
      questions.ts
      index.ts
  writer/
    src/
      workspace.ts
      agent-workspace.ts
      snapshot.ts
      diff.ts
      paths.ts
      index.ts
```

### If the project is a single app

```text
src/
  awf/
    schema/
      events.ts
      ids.ts
      manifest.ts
      questions.ts
      index.ts
    writer/
      workspace.ts
      agent-workspace.ts
      snapshot.ts
      diff.ts
      paths.ts
      index.ts
```

### If the project has a runtime package

Prefer placing AWF writer integration near runtime infrastructure:

```text
src/runtime/observers/awf-observer.ts
```

or:

```text
packages/runtime/src/observers/awf-observer.ts
```

---

## 14. Schema Module

The canonical event schema should live in `events.ts`.

It defines:

- `EventBase`
- lifecycle event schemas
- reasoning event schemas
- action event schemas
- output event schemas
- human-in-the-loop event schemas
- `Event` discriminated union
- `Event` inferred type
- `EventType`
- `parseEventLine(line)`

Example parser:

```ts
export function parseEventLine(line: string): Event | null {
  try {
    const json = JSON.parse(line);
    const result = Event.safeParse(json);
    return result.success ? result.data : null;
  } catch {
    return null;
  }
}
```

Rules:

- new event types must be added to the relevant group
- new event types must be added to the discriminated union
- documentation must be updated when event types change
- do not define separate hand-written TypeScript event types

---

## 15. Manifest Schemas

Recommended `RunManifest` schema:

```ts
export const RunManifest = z.object({
  runId: z.string().regex(/^run_[0-9A-HJKMNP-TV-Z]{26}$/),
  goal: z.string(),
  status: z.enum(["running", "success", "failure", "cancelled"]),
  createdAt: z.string().datetime({ offset: true }),
  updatedAt: z.string().datetime({ offset: true }),
  agents: z.array(z.object({
    agentId: z.string().min(1),
    path: z.string(),
    status: z.string(),
  })).default([]),
  metadata: z.record(z.unknown()).default({}),
});

export type RunManifest = z.infer<typeof RunManifest>;
```

Recommended `AgentManifest` schema:

```ts
export const AgentManifest = z.object({
  runId: z.string().regex(/^run_[0-9A-HJKMNP-TV-Z]{26}$/),
  agentId: z.string().min(1),
  parentAgentId: z.string().nullable().optional(),
  goal: z.string().optional(),
  role: z.string().optional(),
  status: z.enum(["idle", "running", "blocked", "success", "failure", "cancelled"]),
  createdAt: z.string().datetime({ offset: true }),
  updatedAt: z.string().datetime({ offset: true }),
  startedAt: z.string().datetime({ offset: true }).nullable().optional(),
  endedAt: z.string().datetime({ offset: true }).nullable().optional(),
  metadata: z.record(z.unknown()).default({}),
});

export type AgentManifest = z.infer<typeof AgentManifest>;
```

---

## 16. Question Schema

Recommended `QuestionFile` schema:

```ts
export const QuestionFile = z.object({
  questionId: z.string().regex(/^q_[0-9A-HJKMNP-TV-Z]{26}$/),
  runId: z.string().regex(/^run_[0-9A-HJKMNP-TV-Z]{26}$/),
  agentId: z.string().min(1),
  status: z.enum(["pending", "answered"]),
  prompt: z.string(),
  options: z.array(z.string()).optional(),
  blocking: z.boolean().default(true),
  createdAt: z.string().datetime({ offset: true }),
  answer: z.string().optional(),
  answeredAt: z.string().datetime({ offset: true }).optional(),
  answeredBy: z.enum(["human", "policy"]).optional(),
});

export type QuestionFile = z.infer<typeof QuestionFile>;
```

---

## 17. ID Helpers

Recommended helpers:

```ts
export function eventId(): string;
export function runId(): string;
export function questionId(): string;
export function callId(): string;
export function safeAgentId(input: string): string;
```

Expected output formats:

```text
eventId     -> evt_<ULID>
runId       -> run_<ULID>
questionId  -> q_<ULID>
callId      -> call_<ULID>
safeAgentId -> lowercase-safe-agent-name
```

Use Crockford ULIDs or an equivalent sortable ID generator that satisfies the schema regex.

---

## 18. Diff Staging

`editFile(path, oldText, newText)` should:

1. compute added and removed line counts
2. create a unified diff
3. write the diff under `.agent/diffs/`
4. emit `file.edit`
5. emit `diff.staged`
6. return a reference to the staged patch

Example patch path:

```text
diffs/0001-add-awf-observer.patch
```

Example `file.edit` payload:

```json
{
  "path": "src/runtime/workflow-runtime.ts",
  "added": 24,
  "removed": 3,
  "diffRef": "diffs/0001-add-awf-observer.patch"
}
```

Example `diff.staged` payload:

```json
{
  "patchPath": "diffs/0001-add-awf-observer.patch",
  "summary": "Add AWF observer hook",
  "filesChanged": 1
}
```

---

## 19. Testing Strategy

Minimum tests:

### Schema tests

- every event type parses
- invalid event lines return `null`
- ID helpers match expected patterns
- manifests parse
- question files parse

### Writer tests

- creates `.agents/` structure
- creates per-agent `.agent/` structure
- appends valid JSONL events
- does not rewrite previous events
- stages diffs under `.agent/diffs/`
- does not write runtime edits to host files
- writes pending question file
- detects answered question file
- emits `question.answer`

### Snapshot tests

- missing `.agents/` returns empty snapshot
- malformed JSONL line becomes diagnostic
- timeline is sorted
- swimlanes are grouped by agent
- pending questions are derived correctly
- diffs are listed correctly

### Integration tests

If the existing workflow runtime has test harnesses, add one test proving:

- running a workflow emits AWF events
- a tool call becomes `tool.call` and `tool.result`
- a blocking human question can be answered and resumed

---

## 20. Versioning and Evolution

AWF v0 is intentionally small.

Out of scope for v0:

- event replay to rebuild filesystem state
- database persistence
- authentication
- multi-user collaboration
- real-time conflict resolution
- applying diffs to the host repo
- external agent framework integration if not already present
- long-term archival format

Design for future replay, but do not build it in v0.

Future versions may add:

- run history browser
- event replay
- patch application workflow
- approval policies
- multi-user intervention
- remote artifact storage
- distributed agent workspaces
- signed event logs
- richer graph/DAG metadata

---

## 21. Implementation Checklist

Use this checklist when adding AWF to a project.

### Runtime

- [ ] Identify workflow run start/end hooks
- [ ] Identify agent/node/task lifecycle hooks
- [ ] Identify tool call boundary
- [ ] Identify human input/blocking boundary
- [ ] Add AWF writer
- [ ] Emit lifecycle events
- [ ] Emit tool call/result events
- [ ] Emit decision/note events where available
- [ ] Stage proposed file edits
- [ ] Write artifacts under `.agent/artifacts/`
- [ ] Write scratch under `.agent/scratch/`
- [ ] Implement question ask/answer flow

### Filesystem

- [ ] Create `.agents/manifest.json`
- [ ] Create `.agents/shared/`
- [ ] Create `.agents/agents/<agentId>/.agent/`
- [ ] Create `events.jsonl`
- [ ] Create `plan.md`
- [ ] Create `decisions.md`
- [ ] Create `questions/`
- [ ] Create `artifacts/`
- [ ] Create `scratch/`
- [ ] Create `diffs/`

### UI

- [ ] Read AWF snapshot
- [ ] Show run summary
- [ ] Show agent swimlanes
- [ ] Show timeline
- [ ] Show decisions
- [ ] Show diffs
- [ ] Show pending questions
- [ ] Allow answering questions
- [ ] Refresh snapshot live or via polling

### Quality

- [ ] TypeScript typecheck passes
- [ ] Tests pass
- [ ] No `any` unless justified
- [ ] No runtime writes outside `.agents/`
- [ ] Event schema is canonical
- [ ] New event types documented

---

## 22. Recommended Codex Implementation Instruction

When asking a coding agent to implement AWF, use this framing:

```text
Implement AWF as an observer, persistence, and inspection layer for the existing multi-agent workflow runtime.

Do not replace the current workflow engine.
Do not replace the current UI framework.
Do not assume Next.js.
Do not create a separate demo app unless needed for smoke testing.

Integrate AWF into the existing run lifecycle, agent lifecycle, tool call boundary, staged diff flow, and human question flow.

Use the canonical Zod event schema.
Write runtime data only under .agents/.
Expose a snapshot that the existing workflow editor can render.
```

---

## 23. Glossary

### AWF

Agent Workspace Format. A filesystem convention and supporting tooling for inspectable multi-agent execution.

### Run

One workflow execution.

### Agent

Any workflow participant: agent, node, task, worker, persona, or tool-using unit.

### Workspace

The `.agents/` root plus per-agent `.agent/` folders.

### Event

A typed JSON object written as one line in `events.jsonl`.

### Snapshot

A read model built by scanning `.agents/` and deriving UI-friendly data.

### Staged diff

A proposed file edit stored as a patch file, not applied to the host repo.

### Question

A human-in-the-loop request written as JSON and surfaced in the workflow editor.

### Intervention

A human or system action such as pause, resume, cancel, or edit-plan.

---

## 24. Summary

AWF makes multi-agent workflows inspectable without forcing a new runtime or UI framework.

It gives agents a disciplined way to leave behind:

- what they did
- why they did it
- what tools they called
- what files they proposed changing
- what questions blocked them
- what the human answered
- what artifacts they produced

For a workflow editor, AWF becomes the bridge between autonomous execution and human oversight.

The result is a multi-agent system that is easier to debug, govern, trust, and improve.

