# AI development log

How this agent was built, using agentic tooling, between **2026-04-23 and 2026-09-09**
(61 commits in the engine repository).

This is not a prompt dump. The sections below are the iterations that changed the design,
selected for having hard evidence behind them — a commit, a failing test, or an error in a
session log — over ones that were only conversation.

**On dates.** This tool predates the competition window, which opened 2026-08-31. That is
stated here rather than worked around, because every claim below is timestamped and the
dates are the point. Of the five iterations documented, **1 and 3 fall six days before the
window (2026-08-25); 2, 4 and 5 fall inside it (2026-09-05), as does the autonomous loop
and the documentation gate's first live catch (2026-09-09).** What was built during the
window is the documentation gate, the credential-resolution fix, the watchdog correction
and the decision recorded in iteration 5. Judges should read the dates as given.

**Citation format.** `commit <hash>` refers to the engine repository's history. `[idx N]`
refers to an event index in a Claude Code session log under `~/.claude/projects/…`, with its
UTC timestamp. Commit subjects are paraphrased rather than quoted, because they carry internal
issue keys (see the redaction log).

---

## How this was built

### The development environment

Work happened in **interactive Claude Code CLI sessions**, with the human operator directing
scope and the agent implementing, verifying and correcting inside a turn. 59 session logs exist
across the workspace; the engine's own development concentrates in one long-running session
spanning 2026-08-13 → 2026-09-09.

### Agentic components actually in use

| Component | Used? | Detail |
| --- | --- | --- |
| Claude Code CLI (interactive) | **Yes** | The development surface. |
| Headless CLI processes (`claude -p`) | **Yes** | The *product* spawns one per pipeline stage. |
| MCP server (browser automation) | **Yes** | Pinned version, spawned per process. |
| Skill + plugin packaging | **Yes** | The engine ships as a plugin; a skill dispatches to it. |
| Automated code review (skill) | **Yes** | Run before opening pull requests. |
| Subagents / fan-out | **No evidence** | In *development*. The product itself fans out — see below. |
| Hooks (any kind) | **No** | Verified absent — see below. |
| CI pipeline | **No** | Verified absent — see below. |

> **Where the orchestration is.** The rows above describe the *development* surface, which was
> interactive. The orchestration this project is built on lives in the **product**: one headless
> process per pipeline stage, each with its own prompt and its own tool allowlist, executors
> sharded round-robin across a plan and a separate verifier process per candidate finding. That
> structure is documented in [SYSTEM.md](SYSTEM.md). "No subagents" above means the development
> sessions did not delegate — not that the system does not.

### How the browser was wired

Each spawned process receives its MCP configuration **inline** (`--mcp-config` plus
`--strict-mcp-config`) rather than inheriting a project-level config file. The server is pinned
to an exact version, launched isolated, and handed a storage-state file so the process starts
authenticated.

Pinning is load-bearing, not hygiene: a browser installed by a different server version does
not satisfy the one that runs, and the resulting failure looks like a product bug — every
navigation fails and the process is killed by the navigation deadline.

### Division of labour

The operator sets scope, approves method, performs all merges, and decides what reaches the
team. The agent implements, runs the gates, and — as the evidence below shows — diagnoses and
repairs its own breakage without being asked. Nothing was merged by the agent.

---

## Iterations that mattered

### 1. Two safety rules that cancelled each other out

**Commit `0b4600d` · 2026-08-25 19:24**

*What was there.* A watchdog killed any browser process that had not completed a successful
navigation within two minutes of spawn. Separately, the planner had been instructed to open
scenarios with API setup steps rather than assuming state existed.

*How the problem surfaced.* The two rules were written for different reasons and met in
production: a correctly-working process doing exactly what the planner asked — setup first,
browser second — was killed mid-setup and reported as an *environment failure*. The commit
states it plainly: the two rules contradicted each other and the deadline won.

*What changed.* The window now starts when the process first **attempts** to navigate — the
moment "cannot reach the app" becomes provable — rather than at spawn. A process that never
reaches for the browser is still bounded by the global run timeout.

*Result.* The watchdog logic was also moved out of a module that could not be imported by a
test (it had a top-level `await main()`), into one that could. The arm/fire/cancel decision
went from **0 to 15 tests**.

---

### 2. The probe that blamed the network for its own stale credential

**Commit `19144e9` · 2026-09-05 18:55**

*What was there.* A pre-flight probe that checks whether local checkouts are behind their
remote before a run.

*How the problem surfaced.* The probe reported the remote as unreachable from inside the
runner, while the identical command succeeded standalone in about a second.

*What changed.* The cause was established empirically rather than guessed, and the commit
records the chain: all repositories use HTTPS remotes; the credential helper defers to an
environment token when one is set; the configuration module loads the target repository's
`.env` at import time; that file carried an **expired** token. So inside the runner the VCS was
handed a dead credential, and standalone the variable was simply unset.

Demonstrated across all five repositories: every one fails with the variable set, every one
succeeds with it stripped. The fix reused a sanitising helper that already existed one module
away, written for this exact class of problem.

*Result.* A warning that had been silently wrong became correct. The commit notes the deeper
finding — **the defence already existed and was not reached from this code path.**

---

### 3. Eight documented commands, none of which existed

**Commit `69fbb2e` · 2026-08-25 20:09**

*What was there.* A README instructing the reader to run eight npm scripts. `package.json`
defined three, none of them those eight. Anyone following the documentation could not start the
tool at all.

*How the problem surfaced.* Verification of the documented entry point against the actual
package manifest.

*What changed.* Root cause was extraction residue: the README was two documents stitched
together, with two top-level titles and a note telling the reader to mentally substitute
commands because the documentation was "shared with" another installation. That installation was
verified not to exist anywhere on the machine. The document deferred to a fiction instead of
describing the one real path.

*Result.* One title, one invocation, no translation layer — and a decision recorded from the
operator: fix the documentation, do not add scripts to match it.

---

### 4. The documentation fix that reintroduced the defect it fixed

**Commit `3629211` · 2026-09-05 19:41**

*What was there.* After iteration 3, an automated review of the batch returned **15 findings**.
Sorted by which gate covers the file they live in: **6 in code** (covered by typecheck and unit
tests), **7 in documentation and comment blocks** (covered by nothing), **2 in configuration**.

*How the problem surfaced.* Two thirds of the findings landed where no gate existed — and four
of them were produced by a single 83-replacement regex sweep performed by the *fix in iteration
3*. The remedy for "the docs document commands that cannot run" had reintroduced that same
defect in four places.

*What changed.* The response was not "be more careful". A command that cannot run is a defect
whether it lives in a `.ts` file or a README, and only one of those had a gate. A new
`check:docs` gate lifts every command out of markdown files **and** comment blocks — following
line continuations, and reading inline code spans as well as fenced blocks — then checks three
things a machine can decide: that it parses, that the paths it names exist, and that it does not
invoke a script the package does not define.

*Result.* Validated the way a test should be: run against the commit **before** the batch's
fixes, it reports exactly the six real defects and nothing on the repaired tree. The first pass
produced three false positives, all fixed before shipping, with the reasoning recorded in the
commit — *a gate that cries wolf is one people learn to ignore.*

Scope was deliberately limited to what a machine can decide. Prose staleness stays a human
problem, and the gate claims only what it checks.

---

### 5. Deciding not to repair a subsystem

**Commit `ffe0359` · 2026-09-05 19:01**

*What was there.* A catalogue that captured user journeys after each run and fed them back into
the planner, which was instructed to prefer stored steps over deriving its own.

*How the problem surfaced.* Inspection of what it had actually stored. The single journey still
shipping showed every defect at once: ten steps of which six were the same action, only two
interaction types and no keyboard input at all — because the key-press tool was missing from the
capture list, making some journeys *structurally* uncapturable — plus a field asserting the
journey had been revalidated, which the deduplication logic could not actually establish.

*What changed.* The subsystem was **gated off at both ends** rather than repaired. Gating only
the read side would have left the capture side accumulating unusable data and spending a model
call per new segment for data nobody reads.

*Result.* The commit records the trade explicitly: repair was the other option and was rejected,
because a separate pending decision may delete the subsystem outright in favour of the
applications' own end-to-end suites — which encode the same knowledge correctly, are reviewed,
and break loudly when the app changes. Seven defects' worth of design work on something that may
be deleted was judged a bad trade. The code stays in place, behind an opt-in flag, because the
decision it waits on belongs to another piece of work.

**This is the iteration that best characterises the project**: the output of the exercise was a
justified *no*, and the reasoning was written down where the next person will find it.

---

## Failures and recovery

### A test broke and was repaired in 24 seconds, with no human in the loop

**Session `[idx 2188–2193]` · 2026-09-05 22:13:24 → 22:13:48 UTC · shipped in commit `d25d22e`**

The mechanical test for autonomy: no human turn exists between the failure and the passing
re-verification. Verified programmatically — the previous human turn is `[idx 1996]` at
**21:48:30**, the next is `[idx 2201]` at **22:24:19**. The loop sits in a 35-minute window with
nothing from the operator inside it.

| Step | Time (UTC) | What happened |
| --- | --- | --- |
| **Act** | 22:13:24 | Rewrote a test's module mock to be mutable so a feature flag's *disabled* branch became reachable; ran typecheck and the full suite. |
| **Verify → red** | 22:13:30 | `✓ tsc OK` · `FAIL …` · `Test Files 1 failed | 39 passed (40)` |
| **Diagnose** | 22:13:36 | Named the mechanism unprompted: the mocking call is hoisted above the `const` declaration, leaving the reference in the temporal dead zone. |
| **Fix** | 22:13:43 | One line — wrapped the mock object in the framework's `hoisted` helper. |
| **Verify → green** | 22:13:48 | `✓ tsc OK` · `Test Files 40 passed (40)` · `Tests 355 passed (355)` |

Committed 40 seconds later. The commit's diff contains exactly that one-line change.

**The part that matters more than the repair:** the same passing run then deleted a guard line
from the *production* file to confirm the newly added test would actually go red, and restored
it. Green → deliberately broken → green, still with no human turn. The agent did not settle for
a passing suite; it checked that the new test was load-bearing.

---

## Context changes

What each process stopped or started receiving, and why. All three are enforced in
`shared/tools-allowlist.ts` and `runner/generate-test-plan.ts`.

### The browser process lost the ability to read source

The process that drives the app is denied `Read`, `Grep` and `Glob` **explicitly**.

The reason is recorded in the code: without the denial, browser processes would search the local
working tree and emit "this feature is missing" findings grounded in **local source rather than
the deployed application under test**. The comment cites a specific incident dated 2026-06-14.

The enforcement detail is the interesting part. Listing tools in the *allow* list is not
sufficient — in headless mode the read-only built-ins are auto-permitted unless **explicitly
denied**. An allowlist that simply omits them silently grants them. This was found the hard way.

The framing that follows from it: the browser process is a manual tester acting through the UI.
Its only context is the change under test, a navigation map, and the ticket — all injected into
the prompt. It does not get to look at the code.

### The planner lost every agentic tool

> **Correction to the brief.** The question asked when the planner lost *browser* access. No
> evidence was found that it ever had it. What is evidenced is stronger: the planner has **all**
> agentic tools disabled — shell, file read, file write, search, web fetch, web search, task
> delegation and task tracking.

It is a pure prompt-to-JSON call. The recorded reasoning: disabling tools forces the model to
answer from the prompt it was given, and eliminates round-trips that were causing **inconsistent
generation time**. Every input it needs — the change summary, the diff, the navigation document,
learned corrections — is baked into the prompt instead.

### The browser process gained two narrow, non-source capabilities

Against the "no source access" rule, two scoped command-line tools were added back, each argued
for individually:

1. **A fixture generator**, so a plan that needs a file upload can produce the file. Read-only
   against source; writes only into a fixtures directory.
2. **A read-only API client**, pinned to the `GET` verb. Justified as *black-box*, not source
   reading: it queries the application's own API with the run's session token, the way a manual
   tester reads the network tab, so the process can verify state that no screen displays.

A later addition allowed a seeding verb. Note the defence-in-depth: the verb is pinned in the
shell pattern **and** the permitted operations are allowlisted inside the tool itself, because —
as the comment states — the shell pattern alone cannot restrict *which* operation runs.

Four browser capabilities were also removed as unused: screenshot, drag, code execution and
expression evaluation. File upload was kept, because the fixture flow needs it.

---

## Human decisions

> **Marked as draft, as requested.** What follows is only **what** was decided, read off the
> session history with its source. The **why** is deliberately absent — it belongs to the
> operator to write, and inferring it here would be exactly the invention this document avoids.

| # | Decision | Source |
| --- | --- | --- |
| 1 | Every change must be validated as working, checked for collision with existing behaviour, and leave no dead or unused code; doubts arising from the agent's own analysis get raised as questions before acting. | `[idx 982]` 2026-08-23 00:01 |
| 2 | Apply the same rigour to our own code that we apply to the code we test. | `[idx 1376]` 2026-08-25 15:29 |
| 3 | Close what is open before starting anything new; defects we introduce ourselves take priority. | `[idx 1497]` 2026-08-25 22:01 |
| 4 | Batch several pieces of work into one branch instead of opening many pull requests. | `[idx 1724]` 2026-08-25 22:49 |
| 5 | Fix the documentation to match reality; do not add scripts to match the documentation. | operator decision of 2026-08-24, recorded in commit `69fbb2e` |
| 6 | Define the trigger for refreshing the installed build: on confirmation of a merge, and at the start of each new piece of work. | `[idx 1973]` 2026-09-05 21:39 |
| 7 | Do not open the pull request yet — resolve the review findings first. | `[idx 2201]` 2026-09-05 22:24 |
| 8 | Documentation must always reflect the current state of the tool. Stated as a hard rule. | `[idx 2517]` 2026-09-09 15:08 |
| 9 | Work already validated gets closed rather than re-tested. | `[idx 1314]` 2026-08-24 13:29 |
| 10 | All merges are performed by the operator, never the agent. | `[idx 869]`, `[idx 1249]`, `[idx 2563]` |

---

## What could not be sourced

Listed rather than filled in.

- **Subagent / fan-out usage is unresolved.** The engine does not delegate to subagents — task
  delegation appears only in a *disallow* list. Whether the automated review skill used in
  development fans out internally was not established from the logs.
- **Two commits are bulk imports.** The engine repository was synchronised from code developed
  elsewhere on 2026-06-09 and 2026-08-04. For anything predating those points, a
  "when was this introduced" query resolves to the **sync commit, not the original authorship**.
  Iterations 1–5 all postdate the last sync, so they are unaffected.
- **The 2026-06-14 incident** cited in the source-access comment predates this repository's
  history for that file and could not be verified here.
- **Reasoning traces are unavailable.** `thinking` blocks in the session logs are stored with
  empty content (signature only). Everything quoted as agent reasoning is its **visible output
  text**, which is verbatim.
- **No pull-request review threads were read.** Statements about reviews come from commit bodies
  and session logs only.

---

## Redaction log

| Redacted | Replaced with | Why |
| --- | --- | --- |
| Company name, product names (3 internal applications, 2 shared libraries) | "the application under test", "the engine", "a shared library" | Employer and internal product identity. |
| Repository names, including the engine's own | "the engine repository", "the target repository" | Internal repository identity. |
| Issue keys (two schemes, ~25 distinct keys) | Commit hashes and event indices | Explicitly excluded; commit hashes are opaque and serve the same citation purpose. |
| Commit subject lines | Paraphrase | Every subject embeds an issue key. |
| All personal names (operator, 4 colleagues) | "the operator", "the team" | No personal names. |
| Third-party vendor names (identity provider, ticket tracker, code host, a data-feed vendor) | Category descriptions | Reveals the internal stack. |
| Hostnames and URLs (deployed environments, API bases, tracker instance) | Omitted entirely | Internal infrastructure. |
| Domain vocabulary (entity, metric and document types from the business domain) | "records", "a file-upload flow" | Reveals the industry and data model. |
| Absolute filesystem paths containing the company name | Repository-relative paths | Explicitly excluded. |
| Environment-variable **values**, tokens, credentials | Never included | No real values anywhere. |
| Verbatim operator quotes | Neutral paraphrase in a table | Written in first person, in Spanish, with internal references throughout. |

**Not redacted, deliberately:** engine-internal file paths (`shared/tools-allowlist.ts`,
`runner/generate-test-plan.ts`), commit hashes, event indices, timestamps, and test counts. None
carry company, product, person or domain identity, and removing them would make the document
unverifiable — which is the one property it needs.
