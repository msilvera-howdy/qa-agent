# Engineering Spec

## Objective

Give a QA engineer a command that takes a ticket id, exercises the change in a real
browser the way a manual tester would, and returns a report a developer can act on
without asking a follow-up question.

The target is not coverage. It is trust: a report that is worth opening a second time.

## The constraint everything follows from

Point a model at a browser and it reports defects that are not there — a display
abbreviation read as truncation, inline validation read as a dead button, two
correctly-scoped totals read as a mismatch, an expired session read as a blank page.

A QA tool that cries wolf is worse than no tool, because every false finding costs a
developer an afternoon to disprove. After two of those, nobody reads the third report.

So the system is specified around suppressing false confidence, accepting a higher
miss rate as the price.

## Requirements

### Functional

| Command | Behaviour |
|---|---|
| `qa test <TICKET-ID>` | Verify one ticket end to end: context → plan → execute → verify → root cause → curate → report. |
| `qa explore` | Exploratory session with no script, through the same verification gate. |
| `qa report` | Re-render the report for the most recent run. |
| `qa debug` | Explain why a run ended the way it did, and what to do about it. |

### Verification gate

- Everything a browser session reports is a **candidate**, not a defect.
- Every candidate is handed to a second, independent session that knows nothing about
  the first, is given only the claim and the steps, and is told to disprove it.
- Only candidates that survive reach a human. Rejected candidates are kept on disk,
  never published.

### Verdicts

Every run resolves to a verdict, and the verdict decides whether anything is published
at all:

| Verdict | Meaning | Published |
|---|---|---|
| `completed` | The agent exercised the app. | The findings. |
| `blocked_auth` | Never got past login. | Nothing. |
| `blocked_environment` | Reached the app, could not use it. | Nothing. |
| `data_gap` | The plan needed data this environment lacks. | Findings, flagged. |
| `timeout` | Ran out of time. | Findings, flagged incomplete. |

Verdicts merge pessimistically: a run is `completed` only when every parallel session
completed.

### Audit trail

Every run leaves on disk: the ticket it worked from, the plan it chose, every candidate
finding including the ones verification rejected, the curated result, and the verbatim
transcript of every session.

## Constraints

- **Tool allowlists are a safety boundary, not configuration.** Browser stages get
  browser tools and nothing else. Reasoning stages get read-only source access and no
  browser. Nothing gets both.
- **The planner never sees selectors.** Scenarios are written in user language so a
  human can read and approve a plan before it runs against anything shared.
- **Every assertion needs a source.** Expected results come from the ticket or the diff;
  an expectation the planner cannot source becomes an open question in the plan's notes,
  not an assertion.
- **No business-correctness judgements.** Test environments run on seeded data, so an
  implausible number is not a defect. Reactive behaviour is.
- **An honest "I don't know" beats a plausible guess.** The root-cause stage may return
  nothing; an invented file path is not an allowed answer.

## Architecture

```
qa test ABC-123
  session  → probe the saved browser session; re-authenticate if stale
  context  → ticket + acceptance criteria + linked pull request diff
  plan     → scenarios in user language, each traced to an acceptance criterion
  execute  → N browser sessions in parallel, scenarios sharded round-robin
  VERIFY   → every candidate → a fresh adversarial session told to disprove it
  root cause → trace the symptom back through the source to a real file:line
  curate   → blocked run → publish nothing · unconfirmed → drop · duplicates → merge
  report   → markdown, most severe first, stating what was dropped and why
```

Each stage is a separate process with its own prompt and its own tool allowlist.
See [SYSTEM.md](SYSTEM.md) for the agent and orchestration detail.

## Major technical decisions

Recorded with their costs in [DESIGN.md](DESIGN.md). In short:

1. Many small processes, not one long-lived agent — a single agent that checks its own
   work agrees with itself. Cost: state must be serialised between stages.
2. Tool allowlist as a boundary — a browser session that can write files can "fix" the
   bug it was asked to find and report success.
3. Selector-free plans — a plan naming selectors rots on the first markup change, and
   nobody approves a wall of `[data-testid]`.
4. Sourced assertions only — kills confident assertions invented by a model that never
   read a requirement.
5. Parallel sessions, sharded round-robin — interference between sessions is surfaced
   for a human rather than solved automatically.
6. Root cause may return nothing — one invented path costs the report its credibility.

## What is deliberately left to a human

| Left to a human | Why |
|---|---|
| Which environment to run against | A silent retry elsewhere hides the failure worth seeing. |
| Whether a plan may mutate shared data | Requires knowing what other people rely on right now. |
| Whether a `data_gap` means seed the data or change the plan | Depends on what the test is for. |
| Approving the report before it becomes tickets | The agent proposes; a person decides what the team sees. |

## Definition of done

A run is correct when:

- No finding reaches the report that a second, independent session did not confirm.
- A run that never reached the app publishes nothing — reporting "no defects found"
  after a blocked run is structurally prevented, not discouraged.
- Every reported finding carries steps, observed, expected, and what settled the
  verification.
- The report states what was dropped and why, with the counts on the header line.
- The run folder holds the full trail: plan, all candidates, rejections, transcripts.

## Failure modes, ranked by cost

1. **Reporting a defect that is not real.** Costs a developer an afternoon. Mitigated by
   the verification gate and by naming the known false-positive classes in the prompts.
2. **Reporting "all clear" after a run that never reached the app.** Costs a release.
   Structurally prevented: blocked verdicts publish nothing.
3. **Missing a real defect.** Costs what it would have cost without the agent.
   Acceptable — this is a net addition to a QA process, not a replacement for one.

The ordering is the design.
