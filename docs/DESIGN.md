# Design

Why the agent is built the way it is, and what each decision costs.

---

## 1. Many small processes, not one agent

An agent that plans, tests, verifies and reports in one long-lived session accumulates its own
reasoning as context — and then **it agrees with itself**. By the time it reaches the verification
step it has already spent thousands of tokens arguing that the defect is real.

So each stage is a separate process with its own prompt and its own tools. The verifier is told
the claim and the steps, and nothing else. It does not know what the executor concluded, how
confident it was, or how long it spent.

**What it costs.** State has to be serialised between stages, and every handoff is a place where
information can be lost. The compensation is that the run folder becomes a complete audit trail,
and a run that dies halfway still leaves everything it learned on disk.

---

## 2. The tool allowlist is a safety boundary, not a configuration detail

| Stage | Tools | Explicitly denied |
| --- | --- | --- |
| Browser sessions | navigate, click, type, snapshot, console, network | file writes, shell, source access |
| Planner | read-only source access | the browser |
| Root-cause investigator | read-only source access | the browser, file writes |

A browser session that can edit files can "fix" the bug it was asked to find and then report
success. That is not a hypothetical failure mode — it is the obvious thing for a helpful model to
do when it finds something broken.

Separately, a planner with browser access stops planning and starts testing, and you get a plan
written after the fact to match what it already saw.

---

## 3. The planner never sees selectors

Scenarios are written in user language: who is on what screen, what they do, what they should see.
No element ids, no CSS paths, no API endpoints.

Two reasons. A plan that names selectors rots the first time the markup changes. And — more
importantly — a human has to be able to read the plan and approve it before it runs against
anything shared. Nobody approves a wall of `[data-testid]`.

**What it costs.** The browser session has to find things itself, which is slower and occasionally
wrong. That is the correct trade: a session that misreads the screen produces one bad scenario,
while a brittle selector produces a suite that silently stops testing anything.

---

## 4. Every assertion needs a source

The plan's expected results come from the ticket or the diff. If the planner cannot source an
expectation, it is not allowed to assert it — the doubt goes into the plan's notes as an open
question instead.

This kills the most common failure of generated test plans: confident assertions about how a
feature "should" work, invented by a model that never read a requirement. Those produce defects
that are really disagreements about the spec, and they are the fastest way to lose a developer's
trust.

---

## 5. Parallelism, and what it breaks

Browser sessions run concurrently, sharded round-robin rather than in contiguous blocks so one
slow scenario does not decide the run's wall-clock time.

**What it costs.** The sessions share one application. A plan whose scenarios mutate the same
record will produce sessions that interfere with each other, and the resulting "defects" are the
agent tripping over itself.

This is not solved automatically — it is surfaced. A `--plan-only` mode prints the plan and stops,
so a human can read it before it touches anything shared. Automating this away would mean letting
the agent decide which data is safe to mutate, which is exactly the judgement it should not have.

---

## 6. An honest "I don't know" beats a plausible guess

The root-cause stage traces a confirmed symptom back through the source to a `file:line`. It is
explicitly permitted to return nothing, and told to do so when the evidence does not reach the
source.

One invented file path costs the report all of its credibility. Credibility is the only reason
anyone opens the second report, and the second report is the entire value of the system — the
first one gets read out of curiosity.

---

## 7. What is deliberately not automated

| Left to a human | Why |
| --- | --- |
| Which environment to run against | A silent retry elsewhere hides the failure worth seeing. |
| Whether a plan may mutate shared data | Requires knowing what other people are relying on right now. |
| Whether a `data_gap` means seed the data or change the plan | Depends on what the test is actually for. |
| Approving the report before it becomes tickets | The agent proposes; a person decides what the team sees. |

The agent is built to be **trusted on a narrow question** — did this change break something
observable — rather than to be autonomous over the whole workflow. Narrow and reliable is worth
more than broad and occasionally wrong, because the second one still needs a human reviewing
everything, which is the cost you were trying to remove.

---

## 8. Failure modes, ranked by cost

1. **Reporting a defect that is not real.** Costs a developer an afternoon. Mitigated by the
   verification gate and by naming the known false-positive classes explicitly in the prompts.
2. **Reporting "all clear" after a run that never reached the app.** Costs a release. Structurally
   prevented: blocked verdicts publish nothing.
3. **Missing a real defect.** Costs what it would have cost without the agent. Acceptable — this
   is a net addition to a QA process, not a replacement for one.

The ordering is the design. Everything above follows from putting false confidence above missed
coverage.
