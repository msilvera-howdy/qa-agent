# Test harness and feedback loop

What tests exist, what each layer covers, what runs automatically, and one worked example of a
gate catching a real defect during development.

> The implementation is not published, so this document describes the harness rather than
> showing it. Every number below comes from an actual recorded run or from counting the tree —
> the two disagree slightly, and the disagreement is reported rather than smoothed over.

---

## Size, from the last real run

The most recent complete verified run — **2026-09-09, 15:12:38 UTC**, corresponding to the final
commit on the branch — reported all three gates green together:

```
✓ typecheck
✓  Test Files  41 passed (41)
✓       Tests  374 passed (374)
✓ Docs gate: OK — every documented command parses and names paths that exist.
```

### Distribution across the tree

Counted statically from the current working tree:

| Area | Test files | `it(` blocks |
| --- | --- | --- |
| Shared infrastructure | 13 | 136 |
| Pipeline runners | 14 | 87 |
| Selector / journey playbooks | 8 | 70 |
| Tooling and gates | 3 | 38 |
| API client | 3 | 37 |
| **Total** | **41** | **368** |

**The two counts disagree: the runner reports 374 tests, a static count of `it(` blocks finds
368.** The likely explanation is parameterised cases that expand at runtime, but that was not
verified, so both numbers are given as measured rather than one being presented as the answer.
The file count — 41 — agrees exactly across both methods.

---

## What each layer covers

### Unit tests — the decision logic

The largest layer. Its subject is the pure logic that decides what a human sees: how verdicts
merge across parallel processes, what is allowed to reach a report, how duplicate findings
collapse, how malformed model output is parsed or rejected.

These are the parts that are *pure functions on purpose*, so that the rules about publishing can
be verified without a browser, a network or an application. A representative case: the
tool-permission boundary has its own tests asserting the negative — that the browser-driving
process cannot reach file-editing or source-reading tools in any mode.

### Integration-flavoured tests — the parts that touch the machine

**19 of the 41 files** exercise real filesystem, subprocess or network surfaces rather than
mocking them: temporary directories, command invocation, credential resolution. These cover the
places where the engine meets its environment, which is where its worst failures have come from
— a stale credential inherited from a configuration file, a path that exists in one working
directory and not another.

### Prompt-facing tests

Two test files target prompt construction rather than prompt *output*: what the assembled
instructions contain, and — the case that matters — what they correctly omit when a feature is
switched off.

This is a deliberate boundary. The harness tests the prompt **assembly**, which is
deterministic. It does not score model responses; there is **no automated eval suite for output
quality**, and that is a real gap rather than an omission from this document.

### The documentation gate

A third gate alongside typecheck and unit tests. It extracts every command from markdown files
and from comment blocks — following line continuations, and reading inline code spans as well as
fenced blocks — then checks three things a machine can decide:

1. the command parses,
2. the paths it names exist,
3. it does not invoke a package script that is not defined.

Its scope is narrow by design. Prose that describes a deleted feature is a semantic problem and
stays with human review; the gate claims only what it actually checks. It has its own unit tests.

### Mutation checks — a practice, not a suite

Recurring in the development history but **not automated**: after adding a test, deliberately
break the production code it is supposed to protect, confirm the test goes red, and restore.

This is done by hand, in-session, and leaves no artefact in the repository. It is listed here
because it appears repeatedly in the logs and materially affected what shipped — but it is a
discipline, not a gate, and nothing enforces it.

### Automated code review

A review pass was run before opening pull requests. On one batch it returned **15 findings**,
and the distribution is the most useful thing the harness ever produced about itself:

| Where the finding lived | Count | Covered by a gate? |
| --- | --- | --- |
| Code | 6 | Yes — typecheck and unit tests |
| Documentation and comment blocks | 7 | **No** |
| Configuration | 2 | **No** |

Two thirds landed where nothing was checking. That measurement is what caused the documentation
gate to exist.

---

## What enforces the gates

**The gates run on demand; nothing enforces that they run.**

Worth separating, because the two are different claims. The three gates *are* the
automated feedback mechanism — they are machine-checkable, they run in seconds, and the
development log shows the agent invoking them, reading their output and acting on a
failure without being asked. What does not exist is any mechanism that would *stop* a
commit that skipped them.

Enforcement was verified absent rather than assumed, in four places:

| Mechanism | Status |
| --- | --- |
| CI pipeline | **Absent** — no workflow directory, no pipeline configuration of any kind in the repository. |
| Git hooks | **Absent** — the hooks directory contains only the stock samples. |
| Agent-harness hooks (project scope) | **Absent** — project settings contain a permission entry and nothing else. |
| Agent-harness hooks (user scope) | **Absent** — user settings define no hooks. |

All three gates are run **on demand**, in practice by the agent within a working session before
proposing a commit. The evidence throughout the session logs is consistent with that: gates run
in tight succession, seconds apart, immediately after an edit — see the 24-second failure and
repair in *Worked example*, which is the loop running exactly this way.

This is a genuine weakness and is stated as one. The gates are good; their enforcement depends
on the agent choosing to run them and the operator noticing if it did not. Nothing would stop a
commit that skipped them.

---

## Worked example: a gate catching a real defect

### The live catch

**2026-09-09, 15:11:11 UTC**

While updating documentation, the documentation gate failed with a `stale-reference` violation:
an architecture document pointed at a data file that had been **renamed**, naming a path that no
longer existed. The reference had survived every prior review because it read perfectly well —
nothing about the sentence was wrong, only the filename underneath it.

Typecheck could not have found this: the reference was in prose. Unit tests could not have found
it: no code imported that path. It was caught because the gate resolves documented paths against
the actual tree, which is precisely the class of defect it was built for.

### How the gate itself was validated

**Commit `3629211` · 2026-09-05**

The gate was not trusted on the strength of passing. It was validated the way a test should be —
**run against the commit before the batch it was meant to catch**, where it reported exactly the
six real defects that batch contained: four unterminated quotes and two bare relative paths.
Against the repaired tree it reported nothing.

The first implementation also produced **three false positives**. All three were fixed before the
gate shipped, and the reasoning is recorded in the commit: *a gate that cries wolf is one people
learn to ignore.* A checker that is noisy on day one is a checker nobody reads on day thirty.

### A second kind of catch: the suite catching a defect in itself

**2026-09-05, 22:13:30 → 22:13:48 UTC · shipped in commit `d25d22e`**

A change to a test's module mocking broke that test file — one file red out of forty — while
typecheck stayed green. The failure was diagnosed as a hoisting problem (the mocking call is
lifted above the declaration it referenced), fixed in one line, and the same command re-run to
`40 files / 355 tests` green, **24 seconds after the failure, with no human turn in between.**

The same passing run then deleted the guard line from the production file to confirm the newly
added test would actually go red — and it did — before restoring it. Worth separating: the first
half is the suite doing its job, the second is the mutation-check discipline described above,
applied without being asked.

---

## One property that was measured rather than assumed

A review raised the possibility that environment variables set by one test file could leak into
another and silently weaken an assertion.

Rather than adding a cleanup hook and declaring the problem solved, a canary test was written
and run under both configurations. The result: **with the repository's real configuration the
leak does not occur** — file-level isolation prevents it — and **with isolation disabled it does**,
and the canary fails as designed.

The conclusion recorded was accordingly narrow: this is a latent hazard that bites only if
someone turns isolation off, not a live defect. The defensive cleanup was kept as
defence-in-depth; the comment claiming an observed problem was corrected to say what was
actually observed. A follow-up commit exists for exactly that — correcting comments that claimed
more than had been verified.

---

## Redaction log

| Redacted | Replaced with | Why |
| --- | --- | --- |
| Test **file names** that embed an external data vendor's name | Described by layer instead | Vendor name identifies the internal stack. |
| Test file names referencing internal applications and record types | Layer descriptions and counts | Internal product and domain identity. |
| The renamed data file in the live-catch example | "a data file that had been renamed" | Its name and path contain internal directory naming. |
| Issue keys attached to every gate and iteration | Commit hashes and UTC timestamps | Explicitly excluded; hashes are opaque and serve the same purpose. |
| Commit subject lines | Paraphrase | Each embeds an issue key. |
| The test runner's and type checker's configuration values | Described in prose | Configuration would identify the toolchain revision. |
| Repository and directory names | "the repository", area descriptions | Internal repository identity. |
| Any test source code | Nothing — none is quoted | Requested; also unnecessary for the claims made. |

**Not redacted, deliberately:** all counts (41 files, 374 / 368 tests, 19 integration files, 15
review findings, 6 real defects, 3 false positives), all timestamps, and the three gate names.
These are the claims a reviewer would want to check, and the document is worth little without
them.
