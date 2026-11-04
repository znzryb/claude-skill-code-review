---
name: repro-code-review
description: Main agent dispatches independent reviewer subagents; every finding must be actually reproduced (a command was run and its real output pasted); diffs that touch pages also get a browser-driving UI reviewer. TRIGGER：code is written and about to become a PR / leave Draft; user says "review / code review / 审一下 / 起个 review 子代理 / 找找 bug / check the page / 页面验收 / UI review"; a project rule requires an independent subagent review. SKIP：no executable change (pure prose / docs); user only wants style or cleanup suggestions (→ /simplify); user explicitly wants the built-in /code-review.
---

# repro-code-review

An independent code review run by subagents, where **a finding only counts if it was reproduced**.
Reading the diff produces *suspicions*; running something produces *findings*.

Several search angles in section 3 are inspired by Claude Code's built-in `/code-review`
(rewritten with our own wording and examples). What this skill adds: a hard reproduction gate, a
separate browser-driven UI reviewer, and an explicit split of who dispatches, who reviews, who
re-checks, and who fixes.

## 1. Roles

| Role | Who | Does | Never does |
|---|---|---|---|
| **Orchestrator** | the main agent | writes briefs, dispatches reviewers, re-runs repros, dedups, fixes (or dispatches a fixer), writes the record | reviews its own code and calls that "independent" |
| **Code reviewer** | fresh subagent | hunts candidates (section 3), reproduces each (section 4), reports (section 7) | edits the code under review |
| **UI reviewer** | fresh subagent that has browser tools | drives the rendered page (section 5) | judges UI from source alone |
| **Verifier** (large diffs) | fresh subagent | re-runs every reproduced finding cold | trusts the reviewer's pasted output |

Rules:

- **The implementer is never the reviewer.** Whoever wrote the code has the same blind spots
  reading it. If a subagent wrote the code, it returns to the orchestrator when done; the
  orchestrator dispatches a *different* subagent to review.
- **Subagents do not spawn subagents.** In some harnesses (e.g. Claude Code on the web) a subagent
  has no Agent tool. Never tell an implementer "review your work with a subagent when done" — it
  will stall or pretend. All reviewers, verifiers, and fixers are dispatched by the orchestrator.
- **Reviewers are read-only on the branch under review.** Repro scripts go in a scratch directory.
  A test that must live inside the package to run (e.g. Go `_test.go`, a Jest file next to the
  module) goes into a **throwaway worktree** of the same commit
  (`git worktree add <scratch>/review-wt <commit>`), never into the reviewed checkout; remove the
  worktree when done.
- **Reviewers do not get the skill automatically.** Every brief tells the subagent to read this
  file (absolute path) first — see the templates.

## 2. Writing the brief

The brief is the reviewer's only context. Templates: [references/reviewer-briefs.md](references/reviewer-briefs.md).
Every brief contains:

1. **Read this skill first** — the absolute path of this `SKILL.md`; sections 3, 4 and 7 are binding.
2. **What the change is supposed to do** — the user's own words / the agreed spec, verbatim where
   possible. Reviewers check the diff against the requirement, not just against itself.
3. **Scope** — repo path (absolute), branch, base, the exact diff command
   (`git -C <path> diff <base>...<branch>` plus `git diff HEAD` if uncommitted work is in scope).
4. **How to run things** — build command, test command(s), test accounts, which endpoints / pages
   are affected, and the **URL of the test environment the orchestrator already started** (see
   below).
5. **Boundaries** — what the reviewer may run (local tests, the shared test environment, a scratch
   DB) and must not (production writes, real users' accounts, paid APIs, pushes, deleting data).
   Where scratch files go, and that they get cleaned up.
6. **Return format** — section 7.

**Do not put your own conclusions in the brief** ("I think the cache key might be wrong, check
it"). A reviewer handed a hypothesis tends to come back with it confirmed. Give code and
requirements; let it find things. If you have a specific worry, send it to a *separate* reviewer
as an explicit hypothesis to test, under the same reproduction gate.

**One test environment, started by the orchestrator.** When reviewers run in parallel and need a
running app, the orchestrator starts it once and passes the URL. A reviewer that needs its own
server or DB (e.g. to test a migration) uses a distinct port and database name, and says so.

## 3. Where to look (candidate angles)

Each reviewer works through every angle it was assigned and writes down every candidate with a
nameable failure *before* deciding what to reproduce. Do not silently drop half-believed
candidates — the reproduction step is where they get decided.

- **A. Every changed line, in its function.** Read each hunk, then the whole function or block
  around it: an old bug in an untouched line of a touched function is in scope. Ask of each line
  which input or state breaks it — the empty list, the first / last element, a missing optional
  field, a zero that means "set to zero" rather than "not set", a comparison pointing the wrong
  way, an error that is logged and then ignored, a value read from the variable next to the one
  intended.
- **B. What the deleted lines used to guarantee.** For each removed or rewritten line, state what
  it protected (a permission check, input validation, an error branch, an ordering, a test that
  covered a real case) and find where the new code still protects it. Cannot find it → candidate.
- **C. Who else depends on this.** Grep the callers of every changed function, handler, column, or
  config key. Did the input it accepts, the shape it returns, the errors it raises, or the order
  of its side effects change under them? Also the reverse: does something else in this diff now
  call an older function in a way it does not support?
- **D. Known traps of this stack.** Whatever is notorious in the languages and frameworks
  involved — for example: shell word-splitting and unquoted variables, `set -e` not firing inside
  pipelines or `if`; Go goroutines outliving their request; JS `parseInt` without a radix,
  `Array.sort` sorting numbers as strings; Python `is` vs `==` on strings, catching bare
  `Exception` around a retry; SQL built by string concatenation, a missing index on a new filter;
  dates parsed without a timezone; money in floats.
- **E. Layers that sit in front of something.** A cache, retry wrapper, middleware, adapter or
  mock in front of a real component: does every operation it offers reach the real component (and
  the *same* instance), are errors from below passed up instead of turned into "empty", and is
  every operation the callers use actually provided?
- **F. Requirement fit and claims.** Does the behavior match the brief's spec, including edge
  cases the user named? Did anything change that the request did not ask for? Also test the
  *claims* around the change — the PR description's "how to deploy", "tested on X", "step N is
  harmless to re-run", "backward compatible": each one is a statement you can try to break.
- **G. Permissions, visibility, data exposure.** Can a user see or change what they should not?
  Hidden / draft / admin-only items leaking through a new endpoint, list, search, count, or export?
- **H. Concurrency and repetition.** Two requests at once, the same request sent twice, a retry
  after a partial failure, a crash half-way — is the end state still correct?
- **I. Root cause vs. bandaid.** Does the change fix the cause, or special-case a symptom on top
  of shared code? Name the deeper fix if there is one.
- **J. Test gaps.** Which realistic case would break this and is not covered by any test? It
  becomes a finding only if you show the case actually fails; otherwise list it under
  "not covered by tests".

Reuse / simplification / efficiency suggestions are **out of focus**. Report one only when it has
a concrete, demonstrated cost (measurably slower on a realistic input; duplicated logic that has
*already* diverged). Pure style is never reported.

## 4. The reproduction gate

Every candidate is turned into something executable and **actually run**:

| Kind of bug | Typical repro |
|---|---|
| logic / edge case | a failing unit test, or a small script calling the function |
| API / backend | `curl` (or equivalent) against the test server, with the exact request and response |
| data / migration | run against a scratch DB or a copy; show the rows before / after |
| concurrency | a script firing N parallel requests or goroutines; report how many runs out of how many failed |
| shell scripts | run the script with stub commands placed first on `PATH` so nothing real is contacted |
| UI | browser steps + screenshot (section 5) |

Each candidate ends in exactly one status:

- **`REPRODUCED`** — you ran it and got the wrong result. Include the command(s) and the real
  output excerpt next to the expected output. Only this status counts as a **finding**.
- **`UNREPRODUCED`** — the mechanism looks real but your attempt did not trigger it. **Requires at
  least one executed attempt**: paste that attempt's command and output, then say what blocked you
  (missing environment, a timing window, production-only data) and what would confirm it. These
  are reported as *suspicions*, never mixed in with findings. "Clearly wrong by reading, did not
  try" is not a status — go try.
- **`REFUTED`** — you ran it and the behavior was correct, or a guard elsewhere prevents it (cite
  the line). One line each at the end of the report, so the orchestrator knows it was checked.

**When the trigger depends on state you cannot see** (what production currently holds, a real
user's data): reproduce the mechanism by setting up that state yourself, mark it `REPRODUCED`,
and add a **"Precondition (assumed)"** line saying what you assumed, why you believe it (quote the
PR text, log, or commit that implies it), and how someone with access could check it. If you
cannot reconstruct the state at all, it stays a suspicion.

Hard rules:

- **No run, no `REPRODUCED`.** Paste real output trimmed to the relevant lines; never paraphrase
  what a command printed.
- **The repro must be pasteable**: absolute paths, exact commands, setup steps in order. Someone
  else copy-pasting it should get the same result.
- **Reproduce safely**: local / test environment only, test accounts only, scratch files outside
  the reviewed branch (section 1), clean up servers, temp DBs, worktrees and browser tabs you opened.
- **Keep the evidence runnable.** Before cleaning up, copy everything a repro needs (the repro
  files *and* whatever they import or exec, e.g. the exported copy of the code under test, stubs,
  fixtures) into one evidence directory with the same relative layout, and write the repro
  commands relative to it. Report that directory's absolute path.
- If the environment truly cannot run anything (no runtime, no network), say so at the top of the
  report. The orchestrator then knows the gate was not met and must say so in the record.

## 5. UI review (when the diff touches pages)

If the diff changes anything a user sees (components, styles, templates, page routes, static
assets), dispatch a **separate UI reviewer** in parallel with the code reviewer.

- **Pick a subagent that actually has browser tools** (Claude in Chrome, Playwright, the harness's
  built-in browser / preview). Not every subagent type inherits MCP tools — check the type's tool
  list before dispatching. If none can, the orchestrator does the UI pass itself and says so.
- It opens the real rendered page in the test environment, performs the interactions the change
  affects, and works through [references/ui-review-checklist.md](references/ui-review-checklist.md):
  overlap and occlusion, elements that cannot be clicked or reached, text overflow / truncation,
  desktop and mobile widths (defaults 1440×900 and 390×844; use the project's own breakpoints if
  it defines them), light and dark themes, console errors, keyboard focus, loading / empty / error
  states.
- Every UI finding carries: URL, viewport, theme, exact steps, screenshot path, and what should
  have happened. A problem seen in a screenshot is `REPRODUCED`; one inferred from CSS alone is
  `UNREPRODUCED`.
- Do not use desktop "computer use" control unless the user explicitly asked for it; browser
  automation is enough. If no browser tool works, say so plainly — never claim the page was
  visually checked.

## 6. How many reviewers

| Diff | Dispatch |
|---|---|
| Normal | 1 code reviewer covering angles A–J (+ 1 UI reviewer if pages changed) |
| Large: more than ~15 files or ~800 changed lines, **or** it touches auth / permissions, schema migrations, money, concurrency, or deploy scripts | 2–3 code reviewers with the angles split (e.g. A+B+D / C+E+H / F+G+I+J) (+ UI reviewer); then 1 **verifier** that re-runs every reproduced finding cold; then 1 **gap sweep** reviewer given the verified list and told to find only what is *not* on it |

Independent reviewers go out in a single message so they run in parallel. Verifier and sweep run
after the first wave returns. Two reviewers flagging the same line for different reasons are two
findings, not one.

## 7. What a reviewer returns

A short header (what was reviewed, which commands ran, environment limits), then sections in this
order:

**Findings** (`REPRODUCED` only), most severe first:

```markdown
### 1. [high] orders/create.go:142 — a retried request creates a second order
- Status: REPRODUCED
- Trigger: the client retries POST /orders after a timeout while the first request is still committing
- Repro:
    git -C /abs/repo worktree add /abs/scratch/review-wt <commit>
    cp /abs/scratch/create_retry_test.go /abs/scratch/review-wt/orders/
    cd /abs/scratch/review-wt && go test ./orders -run TestCreateRetryDuplicate -count=20
- Expected: exactly 1 row in orders
- Actual: 2 rows in 7 of 20 runs
    --- FAIL: TestCreateRetryDuplicate (0.04s)
        create_retry_test.go:31: got 2 orders, want 1
- Evidence: /abs/scratch/evidence/ (create_retry_test.go + fixtures; rerun with the Repro lines)
```

Severity: `high` (wrong or lost data, security / permission hole, crash), `medium` (wrong behavior
on a realistic path), `low` (rare edge case, cosmetic UI defect). Within a severity, correctness
comes first and inefficiency last.

**Suspicions** (`UNREPRODUCED`): same fields, plus "attempt made" (command + output), "blocked by",
and "would confirm by".

**Checked and refuted**: one line each — what was suspected, what was run, why it is fine:

```markdown
- Empty `ARGS` array under `set -u` on bash 3.2 — ran the suite on /bin/bash 3.2.57, 17/17 pass.
- Trailing newline missing in the record file — fed one to the parser, got the 3 expected fields.
```

**Not covered by tests**: realistic cases no test exercises, one line each
(`- the push succeeds but writing the record afterwards fails`).

Zero findings is a valid result — write "no reproduced findings" rather than padding.

## 8. What the orchestrator does after

1. **Re-run first.** Before fixing anything, the orchestrator itself re-runs every reproduced
   finding's repro and confirms it fails. One that does not fail is downgraded to a suspicion.
   (For large diffs the verifier does this; its outcomes map as: still reproduces → finding;
   does not reproduce → suspicion; repro broken → back to the original reviewer to fix the steps.)
2. **Dedup** across reviewers (same defect, same place, same cause → keep the one with the best repro).
3. **Fix** each finding (or dispatch a fixer subagent), then **re-run the same repro** and confirm
   it now passes. Where the repro is a test, keep it as a regression test.
4. **Suspicions**: try a better repro, fix defensively, or write down why not. High-severity ones
   are not silently dropped.
5. If the fixes were substantial, dispatch a fresh reviewer on the fix diff (not the earlier
   reviewer, not the fixer).
6. **Record it** where the next reader will look (the PR description's `## Code Review` section,
   or the final reply): each finding → status → what changed → commit; each unfixed one → why
   (false positive / intentional / out of scope, reported separately). Say if any part of the gate
   was not met (no browser available, nothing runnable).

## 9. Common failures

- The reviewer only read the diff and returned plausible-sounding bugs with no runs.
- Every candidate was parked as a suspicion without a single attempt.
- The orchestrator's guess went into the brief and came back "confirmed".
- The brief said "follow the procedure" but never gave the path to this file.
- The implementer reviewed its own code, or was told to spawn its own reviewer.
- UI was judged from JSX / CSS without opening the page, or by a subagent with no browser tools.
- A repro test was dropped into the reviewed branch, or run against production / a real account.
- A fix was declared done without re-running the repro that demonstrated the bug.
