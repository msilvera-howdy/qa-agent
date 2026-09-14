# Agentic System Map

How context, agents, tools and workstreams are structured, and how their outputs are
integrated. Prompt-level detail lives in [PROMPTS.md](PROMPTS.md); the reasoning behind
each choice lives in [DESIGN.md](DESIGN.md).

## Shape of the system

Not one agent with many tools. A pipeline of short-lived processes, each with its own
prompt, its own context, and its own tool allowlist.

```
session → context → plan → execute (xN) → VERIFY (xM) → root cause → curate → report
```

A single long-lived agent that tests and then checks its own work accumulates its own
reasoning as context and agrees with itself. Isolation is the mechanism that makes the
verification step mean anything.

## The agents

| Agent | Context it is given | Context it is denied | Tools | Output |
|---|---|---|---|---|
| **Session probe** | Saved browser session state | — | Browser | Live session, or `blocked_auth` |
| **Planner** | Ticket, acceptance criteria, linked PR diff, read-only source | The browser, the running app | Read-only source access | Scenarios in user language, each traced to an acceptance criterion |
| **Executor** (xN) | Its shard of the plan | The other executors' findings | Browser only — no file writes, no shell, no source | Candidate findings |
| **Verifier** (xM) | One claim: title, steps, observed, expected, start URL | Who reported it, how confident they were, how long they spent, every other candidate | Browser only, fresh session | `confirmed` / `not-reproducible` / `known-false-positive` + note |
| **Root-cause investigator** | Confirmed findings, read-only source | The browser, file writes | Read-only source access | `file:line`, or nothing |
| **Curator** | All candidates, all verdicts, all verifier notes | — | None | Published report |

## Deterministic controls

The parts that must not depend on an agent remembering them:

| Control | Enforced how | What it prevents |
|---|---|---|
| Tool allowlist per stage | Process-level, not prompt-level | A browser session that can write files "fixes" the bug it was asked to find, then reports success |
| Planner has no browser | Process-level | A planner that starts testing writes the plan after the fact to match what it saw |
| Verifier context isolation | Separate process, claim passed as data | A verifier that knows the original reasoning agrees with it |
| Verdict model | Curator, not the agent | "No defects found" after a run that never reached the app |
| Pessimistic verdict merge | Curator | A suite claiming coverage it did not have |
| `--plan-only` mode | Runner flag | An agent deciding which shared data is safe to mutate |

## Parallel work

**Where the parallelism is.** Two stages fan out:

- **Execution** — N browser sessions run concurrently against one application.
  Scenarios are sharded **round-robin rather than in contiguous blocks**, so one slow
  scenario does not decide the run's wall-clock time.
- **Verification** — each candidate finding fans out to its own fresh session. These are
  independent by construction: a verifier is given one claim and nothing else, so there
  is no ordering between them.

**How the work is coordinated.** The stages are serialised; the sessions inside a stage
are not. Each session writes its own output to the run folder and the next stage reads
the folder, so there is no shared mutable state between concurrent sessions.

**What parallelism costs, and how it is handled.** Concurrent sessions share one
application. A plan whose scenarios mutate the same record produces sessions that
interfere with each other, and the resulting "defects" are the agent tripping over
itself. This is **surfaced, not solved**: `--plan-only` prints the plan and stops so a
human can read it before it touches anything shared.

**How outputs are integrated.** The curator merges pessimistically — the run is
`completed` only if every parallel session completed. One blocked session means the
suite did not cover what it claims to have covered, and that changes what gets
published.

## Autonomous loops

There are two, and they are different things. The one the judges asked for is the second.

### 1. The loop the product runs

The verification loop inside a run, with no human instruction in the middle:

```
ACT      executor drives the app and produces a candidate finding
VERIFY   a fresh session, with none of the executor's context, is told to disprove it
OBSERVE  the verifier reproduces, or explains it away, or cannot reach the app
REACT    confirmed → root cause → report
         not-reproducible / known-false-positive → dropped to disk, never published
         blocked → the run's verdict changes and publishing stops
```

No human prompt occurs between those steps. A human sets the environment and approves
the plan before the loop, and approves the report after it.

### 2. The loop in how this was built

A recorded instance from a development session on 2026-09-05: the agent changed a test
file, ran typecheck plus the full suite, read a failure, named its specific cause,
applied a one-line fix, and re-ran the same verification to a pass — 24 seconds, with no
human turn anywhere in the window.

Full trace with timestamps, the failure output, the diagnosis and the diff:
**[AUTONOMOUS-LOOP.md](AUTONOMOUS-LOOP.md)**.

## Context engineering

The decisions about what each process is and is not told:

- **The verifier's ignorance is the feature.** It is given the claim and the steps and
  nothing else. Framing decides the outcome: "check whether this is real" confirms
  almost everything; "disprove this" does not.
- **Snapshot budget.** Every accessibility snapshot stays in context for the rest of a
  session. Sessions are told to scope by depth (3 for normal exploration, 5 for grids
  and forms), to snapshot after navigation rather than after every click, and to verify
  cheaply — wait-for-text, URL reads, console, network — none of which reload the tree.
  Budget: at most one snapshot per three or four actions. Without this a session ends at
  40% coverage with nothing to show.
- **Named failure modes, not abstract advice.** Each prompt names the specific way a run
  went wrong, in that failure's own vocabulary, and states its cost. Every line in the
  prompts was added after a run failed that way.
- **Constrained output shape** so the orchestrator can parse it and the model cannot
  hedge.

Full annotated prompts in [PROMPTS.md](PROMPTS.md).

## Evidence

- **Autonomous loop** — [AUTONOMOUS-LOOP.md](AUTONOMOUS-LOOP.md): a verbatim trace of a
  development-session loop that closed without human instruction, with the last human
  turn before it and the first one after it marked by index and timestamp.
- **Sample output** — [SAMPLE-REPORT.md](SAMPLE-REPORT.md) shows a run that produced
  seven candidates and published two, with the five rejections and the verifier's reason
  for each. The header accounting (`2 reported · 4 dropped as unverified · 1 merged as
  duplicate`) is printed on every report on purpose.
