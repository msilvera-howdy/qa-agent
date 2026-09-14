# QA Agent

An autonomous QA agent for web applications. You give it a ticket id. It opens a real browser,
exercises the change the way a manual tester would, **verifies every defect before reporting it**,
and returns a report a developer can act on without asking a follow-up question.

Built on the Claude Code CLI driving Playwright through MCP.

> **About this repository.** This is a design overview — the architecture, the reasoning behind it,
> and the prompt design that makes it work. The running implementation is maintained privately.
> Everything described here is in daily use on a production QA workflow.

---

## The problem it actually solves

Most "AI testing" tools generate test scripts. That is the easy half.

The expensive half is what a QA engineer actually spends their day on: exercising a change in a
real browser and **deciding what is worth a developer's attention**.

That decision is where LLM-driven testing falls apart. Point a model at a browser and it will
report defects that are not there:

| What the model sees | What it reports | What it actually is |
| --- | --- | --- |
| `$89K` in a field | "the value is truncated" | a display abbreviation; focus shows `$89,432` |
| Submit clicked, nothing happened | "the button is dead" | inline validation it snapshotted too early |
| Two totals that disagree | "the numbers are wrong" | two different scopes, both correct |
| A login screen | "the page is blank" | an expired session |

A QA tool that cries wolf is **worse than no tool**, because every false finding costs a developer
an afternoon to disprove. After two of those, nobody reads the third report.

So the entire design is organised around one idea.

---

## The core idea: belief is not a finding

Everything a browser session reports is a **candidate**, not a defect.

Every candidate is handed to a **second, independent browser session** that knows nothing about
the first, is given only the claim and the steps, and is told to **disprove it**.

```
  executor session                        verifier session
  ────────────────                        ────────────────
  drives the app                          fresh browser, zero shared context
  believes it saw X       ──── X ────▶    "another tester claims X. Disprove it."
                                          · follows the steps exactly as written
                                          · retries once on the most charitable reading
                                          · actively tries to explain it away
                                                     │
                       confirmed ◀────────────────────┴────────────────▶ rejected
                            │                                              │
                     into the report                          kept on disk, never published
```

Only what survives the second session reaches a human. In practice this discards a meaningful
share of what the first session was certain about.

The reason it works is that the verifier has **no memory of the original session**. A single
long-lived agent that tests and then checks its own work simply agrees with itself.

---

## The second idea: "broken" and "I couldn't test it" are opposite answers

From inside a browser page, these look identical:

- the feature under test is genuinely broken
- the session expired
- the app never booted
- the required data does not exist in this environment
- the browser was never installed

They mean opposite things to the developer reading the report. So every run resolves to a
**verdict**, and the verdict decides whether anything gets published at all:

| Verdict | Meaning | What gets published |
| --- | --- | --- |
| `completed` | The agent exercised the app. | The findings. |
| `blocked_auth` | Never got past login. | **Nothing.** |
| `blocked_environment` | Reached the app, couldn't use it. | **Nothing.** |
| `data_gap` | The plan needed data this environment lacks. | Findings, flagged. |
| `timeout` | Ran out of time. | Findings, flagged as incomplete. |

Verdicts merge **pessimistically**: a run is `completed` only when every parallel session
completed. One blocked session means the suite did not cover what it claims to have covered.

Reporting "no defects found" after a run that never reached the app is the single most expensive
lie this system could tell, so it is structurally prevented rather than discouraged.

---

## The pipeline

```
   qa test ABC-123
         │
   ┌─────▼──────┐
   │  session   │  probe the saved browser session; re-authenticate if stale
   └─────┬──────┘  a stale session becomes `blocked_auth`, never a page full of phantom bugs
         │
   ┌─────▼──────┐
   │  context   │  ticket + acceptance criteria + linked pull request diff
   └─────┬──────┘  PR discovery degrades: explicit flag → link in the ticket → repo search
         │
   ┌─────▼──────┐
   │   plan     │  scenarios in user language, each traced to an acceptance criterion
   └─────┬──────┘  no selectors, no endpoints — a plan a human could execute and approve
         │
   ┌─────▼──────┐
   │  execute   │  N browser sessions in parallel, scenarios sharded round-robin
   └─────┬──────┘  browser tools only — a session that can write files can "fix" the bug
         │
   ┌─────▼──────┐
   │  VERIFY    │  every candidate → a fresh adversarial session told to disprove it
   └─────┬──────┘  ← the stage that makes the output trustworthy
         │
   ┌─────▼──────┐
   │ root cause │  trace the symptom back through the source to a real file:line
   └─────┬──────┘  "not found here" is an allowed answer; an invented path is not
         │
   ┌─────▼──────┐
   │  curate    │  blocked run → publish nothing · unconfirmed → drop · duplicates → merge
   └─────┬──────┘
         │
   ┌─────▼──────┐
   │  report    │  markdown, most severe first, stating what was dropped and why
   └────────────┘
```

Each stage is a **separate process** with its own prompt and its own tool allowlist. Browser
stages get browser tools and nothing else. Reasoning stages get read-only source access and no
browser. Nothing gets both.

---

## What it does

| Command | What it does |
| --- | --- |
| `qa test <TICKET-ID>` | Verify one ticket end to end. |
| `qa explore` | Exploratory session — hunt for defects with no script, same verification gate. |
| `qa report` | Re-render the report for the most recent run. |
| `qa debug` | Explain why a run ended the way it did, and what to do about it. |

Every run leaves a complete audit trail on disk: the ticket it worked from, the plan it chose,
**every candidate finding including the ones verification rejected**, the curated result, and the
verbatim transcript of every session.

That rejected-findings file matters. When the agent drops something it should have reported,
that is where you find out why.

---

## Read next

| Document | What is in it |
| --- | --- |
| [docs/DESIGN.md](docs/DESIGN.md) | Why it is built this way — the decisions and what they cost. |
| [docs/PROMPTS.md](docs/PROMPTS.md) | The prompt design. This is where most of the engineering actually lives. |
| [docs/SAMPLE-REPORT.md](docs/SAMPLE-REPORT.md) | What it hands back at the end of a run. |


---

## Running it

The implementation is not published, so this section is a specification of what the tool requires rather than instructions you can follow from this repository.

| Requirement | |
| --- | --- |
| Node.js | 24 or later |
| Package manager | npm |
| Build step | None. TypeScript sources run directly. |
| Agent CLI | Installed and authenticated separately. Spawned as a subprocess, one per pipeline stage. |
| Automation browser | Installed out-of-band from the pinned MCP package. |

Three gates must pass before anything ships:

| Gate | Command |
| --- | --- |
| Unit suite | npm test |
| Type gate, no emit | npm run typecheck |
| Documentation gate | npm run check:docs |

The engine is invoked from the target repository's directory, not its own, so there are deliberately no npm scripts for the agent itself.

**Configuration** is by environment variable: a model-provider credential, the base URL of the application under test, and either an identity-provider token or a bot account for authentication. Ticket mode additionally needs issue-tracker and code-host credentials. No values, defaults or examples appear anywhere in this repository.

**External services**: a model provider, a browser-automation MCP server, and the application under test. Ticket mode also needs an issue tracker and a code host. The engine requires no database, no message broker, no container runtime and no cloud account. All state is files on disk.

## Engineering documentation

| Document | What is in it |
| --- | --- |
| docs/SPEC.md | Objective, requirements, constraints, architecture, definition of done. |
| docs/SYSTEM.md | The agentic system map: agents, contexts, tool boundaries, parallel work. |
| docs/AI-DEV-LOG.md | How this was built: iterations, failures, corrections, human decisions. |
| docs/AUTONOMOUS-LOOP.md | A verbatim trace of a development loop that closed with no human instruction. |
| docs/TEST-HARNESS.md | What the gates cover, and one of them catching a real defect. |
