---
name: relay
description: Skill for autonomously driving the implementation phase of tasks whose design is already agreed (fable-orchestrator edition). Use when the main session model is fable; use relay-opus when it is Opus. Fires when the user signals intent to execute an agreed plan ("go ahead", "start implementing", etc.) or explicitly invokes /relay.
---

# relay

A skill for autonomously driving the implementation phase after the design has been settled.

When this skill fires, **the design and approach are assumed to be already agreed in the main session.** How far the design was refined before invocation is at the user's discretion; this skill takes that agreement as given and starts implementing. Do not redo the design here.

## Division of roles

The core of this skill is layer separation. **The core of relay is the main session (fable).**

- **Main session (fable) = design, orchestration, and audit layer**
  Holds the overall design, decomposes tasks, instructs subagents, and audits the deliverables that come back. As a rule, the main session does not touch implementation details directly. Keeping the main context free of grep dumps and trial-and-error logs is essential to preserving audit accuracy.

- **Subagents (relay-*) = implementation and investigation layer**
  Carry out individual work items, each in its own independent context window.

## Guard at invocation

relay-investigator / relay-reviewer are `model: inherit` and use the main session's model as-is. If invoked with a main model outside the fable / opus family (sonnet, haiku, etc.), investigation and review quality degrades **without warning**. Check the main model at invocation; if it is not in the fable / opus family, do not start work — warn the user and ask for instructions.

## Procedure

Execute the following 5 steps in order. Do not skip steps.

### Step 1: Build the task decomposition table

Before spawning even one subagent, always create the task decomposition table and output it as a message. The format is fixed:

| # | Task | Depends on | Agent | Completion criteria |
|---|---|---|---|---|
| 1 | (what to do) | none | relay-investigator | (mechanically checkable condition) |
| 2 | (what to do) | #1 | relay-implementer-std | (same) |

- **Depends on**: numbers of prerequisite tasks. Tasks with no dependencies are candidates for parallel execution
- **Agent**: one of the 7 relay-* agents in the routing table below
- **Completion criteria**: written so they can be verified mechanically at audit time — "test X passes", "file Y contains function Z". Vague criteria like "X is improved" are not acceptable

This table is the ledger for the whole execution phase. When tasks need to be added or split, update the table before acting.

### Step 2: Pick the agent and build the prompt

Do not design subagents from scratch. Choose from the relay-* agents defined in `~/.claude/agents/` according to the routing table below. Model, effort, tool permissions, and report format are fixed in each agent definition and enforced mechanically by the harness.

#### Routing table

| Agent | model / effort | Use for |
|---|---|---|
| relay-investigator | inherit (= main) / high | Investigating the codebase, specs, and root causes. Returns options and a recommendation (read-only) |
| relay-reviewer | inherit (= main) / high | Evaluating diffs for quality, consistency, and security (read-only) |
| relay-implementer | opus (4.8) / high | Changes spanning multiple files, implementations involving concurrency or boundary conditions, complex debugging |
| relay-implementer-std | sonnet / medium | Code generation and refactoring that follows existing patterns |
| relay-codex | sonnet / medium (implementation itself is GPT via codex exec) | Cross-vendor implementation of fully specified tasks. Streams the spec to the Codex CLI and independently verifies the result |
| relay-verifier | sonnet / medium | Running builds, tests, and lint, and returning a first-pass assessment (no fixing) |
| relay-mechanic | haiku / low | File exploration, simple substitutions, boilerplate generation (judgment-free work only) |

Language-level convention and style issues are resolved mechanically with linters / formatters, not review agents (run them as part of relay-verifier's verification). When domain-specific knowledge is needed, existing general-purpose agents (docs-lookup, architect, etc.) may be used as auxiliaries, but keep relay-* as the main path of the implementation phase and do not break the consistency of model selection.

#### Cross-vendor implementation lane (relay-codex)

Implementation tasks specified firmly enough to write out all 4 required prompt items, with no room for discretion, may be routed to relay-codex. Having GPT (Codex CLI) do the implementation catches problems a single vendor's model family would collectively miss. It presupposes that the completion criteria include an executable verification command; tasks with remaining discretion or design judgment go to the relay-implementer family. If relay-codex returns "failed (codex unavailable)", reroute the same spec to relay-implementer or relay-implementer-std.

#### Two-axis review split

For tasks whose spec is documented (clear completion criteria, or grill-me / PRD artifacts exist), split the review into a **spec axis** and a **convention axis** and spawn two relay-reviewers in parallel. The axis is specified in the spawn prompt; axis definitions and the smell baseline are defined on the relay-reviewer side. In the audit, keep the two reports side by side, per axis. Keep visible, separately, code that follows convention but deviates from spec and code that matches spec but violates convention — do not merge the axes into one ranking, because one set of findings will bury the other.

For minor changes, a single relay-reviewer with no axis specified (general review) is enough.

#### Required prompt items

Subagents cannot see the main conversation. The spawn prompt must state all 4 items below. Never spawn with even one item missing.

1. **Purpose**: which part of the whole this task covers (1–2 sentences)
2. **Scope**: an explicit list of files/directories that may be touched. State that changes outside it are forbidden
3. **Constraints**: the agreed design decisions relevant to this task, coding conventions, things not to do
4. **Completion criteria**: copy the completion criteria from the task decomposition table verbatim

#### Structured reports

The report format is baked into each agent definition, so there is no need to copy it into the prompt. Every relay-* agent is defined to return a report with the following structure, and the audit checks reports against it.

```
## Result: complete / partial / failed
## Changed files: (list of full paths; "none" for investigation/verification/review)
## Commands run and results: (commands used for verification and the gist of their output)
## Judgment calls: (on-the-spot decisions not in the instructions; "none" if none)
## Open issues / concerns: ("none" if none)
```

### Step 3: Execute

- Tasks with no dependencies: put multiple Agent calls in one message and run them in parallel
- Dependent tasks: start only after the prerequisite task **passes audit**. Receiving a report is not enough to start
- Do not run implementation-type subagents in parallel on the same set of files. Serialize, or rethink the task split

### Step 4: Audit

Each time a subagent reports back, run the following checklist in the main session. Do not move to the next task until every item passes.

1. **Existence check**: the reported changed files exist and contain the reported changes (Read the relevant parts; no need to read whole files)
2. **Completion criteria verification**: confirm the table's completion criteria with mechanical facts (command output). Where builds/tests are involved, have relay-verifier run them or run the commands directly in main. **Never trust a subagent's "it worked" without verification**
3. **Scope check**: no changes leaked outside "Scope" (check with the file list from git diff, etc.)
4. **Adjudicate judgment calls**: check the report's "Judgment calls" section against the agreed design. If they conflict, send it back

A task that fails audit is sent back to the same agent with the failure reasons stated. **If it still fails after 2 send-backs, treat the task's approach itself as flawed: stop work and report the situation to the user.**

### Step 5: Stop at stopping points

When you reach a point that needs the user's confirmation or testing, stop there and hand back to the user. Where to stop depends on how far the design was refined before invocation, but always stop at least at these points:

- All tasks in the decomposition table have passed audit (completion report)
- A judgment beyond the scope of the agreed design becomes necessary
- The same task has been sent back twice
- A destructive operation (deletion, history rewriting, external publication) becomes necessary

## Difficulty assessment guidance

- "Looks simple" and "is simple" are different. Assign tasks that require context understanding or judgment to relay-implementer-std or above even when they look trivial; use relay-mechanic only when you are confident the work is purely mechanical (no judgment)
- When unsure, round up one tier. The cost of a redo after a send-back exceeds one model tier's difference
- Effort is fixed in the agent definition and cannot be overridden at spawn. The adjustment lever is which agent you assign

## About nesting

relay-* agents are, as a rule, leaves (final execution units) without the Agent tool; parallelism is bundled by the main session as fan-out. The old dynamic nesting practice is abolished. If one task contains serial internal stages like "investigate, then implement", split them into separate tasks in the decomposition table and have main chain them in order.

The only exception is relay-investigator. Investigation is "sweep wide, integrate in one head" work: pushing the sweeping down to children keeps the integration context clean. It is therefore allowed to delegate exploration — and only exploration — to the read-only Explore agent (the discipline is written in the agent definition). Nesting of any write-capable work remains fully forbidden.

## Autonomous mode (unattended operation)

When invoked together with /goal, or when the user explicitly asks for unattended continuous execution, add the following discipline. User intervention is summoned mechanically by the RELAY-STATUS declaration; the user is not expected to watch intermediate results.

### Ledger persistence

In addition to printing it in the conversation, register the Step 1 task decomposition table in the harness task ledger with TaskCreate (1 task = 1 Task; express dependencies with addBlockedBy). The in-conversation table can be lost to context compaction, so the Task ledger is the source of truth for progress. Set in_progress when starting, completed when the audit passes.

### Status declaration

At the end of every turn, output the following single line. When you reach a Step 5 stopping point, declare the corresponding BLOCKED type and stop.

`RELAY-STATUS: RUNNING | DONE | BLOCKED(DESIGN | MANUAL-TEST | PERMISSION | STUCK)`

- **RUNNING**: incomplete tasks remain in the ledger and work can continue
- **DONE**: all tasks passed audit and the completion report is out
- **BLOCKED(DESIGN)**: a judgment beyond the agreed design is needed. Include the points to re-grill in the report
- **BLOCKED(MANUAL-TEST)**: the user's hands-on verification is needed. Include the verification steps in the report
- **BLOCKED(PERMISSION)**: stopped on a permission check
- **BLOCKED(STUCK)**: the same task has been sent back twice

This line is the mechanical signal the /goal evaluator uses to judge termination. Never keep declaring RUNNING where BLOCKED should be declared. Making design decisions on the user's behalf to escape the evaluator's pressure is a worse failure than stopping.

### Recall

When stopping at DONE / BLOCKED, notify via PushNotification: the type, plus one line stating what you need from the user.

### Pre-flight check

Before starting, confirm that the commands needed to verify the completion criteria (tests, builds, lint) can run without permission prompts. If any are not pre-approved, ask the user about them in one batch before starting. In unattended operation, waiting on a permission prompt mid-run is equivalent to being stopped.

### Recommended /goal condition (canonical)

```
/goal RELAY-STATUS: DONE or BLOCKED has been declared
```

Correctness of the deliverable is guaranteed by the audit (mechanical verification of completion criteria), not by the goal evaluator. There is no need to write the deliverable's state into the goal condition. DONE means the mechanical completion criteria are satisfied; final acceptance (is it what was intended?) remains with the user after DONE.

### Loop-back

- BLOCKED(DESIGN) → re-grill only the affected branch (update ADR / CONTEXT.md) → update the ledger → re-arm /goal and resume
- BLOCKED(MANUAL-TEST) → incorporate the user's verification results into the ledger → re-arm /goal and resume
- BLOCKED(STUCK) → rethink the task's approach: re-split it, or ask the user

Because BLOCKED also counts as goal achievement and auto-clears the goal, one cycle of resumption after intervention is defined by re-arming /goal.
