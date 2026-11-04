---
name: repro-code-review
description: How the main agent dispatches independent reviewer subagents to review a diff, where every reported finding must carry a reproduction that the reviewer actually ran (failing test, script, curl, browser steps) plus its real output; UI changes get a separate browser-driven reviewer that clicks through the rendered page. TRIGGER：code is written and about to become a PR / leave Draft; user says "review / code review / 审一下 / 起个 review 子代理 / 找找 bug"; a project rule requires an independent subagent review. SKIP：no code changed (pure docs / content / config text); user only wants style or cleanup suggestions (→ /simplify); user explicitly wants the built-in /code-review.
---

# repro-code-review

An independent code review run by subagents, where **a finding only counts if it was reproduced**.
Reading the diff produces *suspicions*; running something produces *findings*.

The search angles in section 3 are adapted (rewritten, not copied) from Claude Code's built-in
`/code-review`. What this skill adds: a hard reproduction gate, a separate browser-driven UI
reviewer, and an explicit split of who dispatches, who reviews, and who fixes.

## 1. Roles

| Role | Who | Does | Never does |
|---|---|---|---|
| **Orchestrator** | the main agent | writes briefs, dispatches reviewers, dedups, fixes (or dispatches a fixer), re-runs repros, writes the PR record | reviews its own code and calls that "independent" |
| **Code reviewer** | fresh subagent | hunts candidates (section 3), reproduces each (section 4), reports | edits the code under review |
| **UI reviewer** | fresh subagent with a browser tool | drives the rendered page (section 5) | judges UI from source alone |
| **Verifier** (large diffs) | fresh subagent | re-runs every `REPRODUCED` repro from scratch, cold | trusts the reviewer's pasted output |

Rules:

- **The implementer is never the reviewer.** Whoever wrote the code has the same blind spots
  reading it. If a subagent wrote the code, it returns to the orchestrator when done; the
  orchestrator dispatches a *different* subagent to review.
- **Subagents do not spawn subagents.** In some harnesses (e.g. Claude Code on the web) a subagent
  has no Agent tool. Never tell an implementer "review your work with a subagent when done" — it
  will stall or pretend. All reviewers, verifiers, and fixers are dispatched by the orchestrator.
- Reviewers are **read-only on the code under review**. Repro files go in a scratch directory or a
  separate worktree, never into the branch being reviewed.

## 2. Writing the brief

The brief is the reviewer's only context. Templates: [references/reviewer-briefs.md](references/reviewer-briefs.md).
Every brief contains:

1. **What the change is supposed to do** — the user's own words / the agreed spec, verbatim where
   possible. Reviewers check the diff against the requirement, not just against itself.
2. **Scope** — repo path (absolute), branch, base, the exact diff command
   (`git -C <path> diff <base>...<branch>` plus `git diff HEAD` if uncommitted work is in scope).
3. **How to run things** — build command, test command(s), how to start a local/test environment,
   test accounts, which endpoints/pages are affected.
4. **Boundaries** — what the reviewer may run (local tests, local server, test DB) and must not
   (production writes, real users' accounts, paid APIs, force pushes, deleting data). Where scratch
   files go, and that they get cleaned up.
5. **Return format** — section 7, pasted into the brief.

**Do not put your own conclusions in the brief** ("I think the cache key might be wrong, check
it"). A reviewer handed a hypothesis tends to come back with it confirmed. Give code and
requirements; let it find things. If you have a specific worry, send it to a *separate* reviewer
as an explicit hypothesis to test, and treat a "confirmed" answer with the same repro gate.

## 3. Where to look (candidate angles)

Each reviewer works through these and writes down every candidate with a nameable failure, before
deciding which to reproduce. Do not silently drop half-believed candidates — the repro step is
where they get decided.

- **A. Line by line.** Read every hunk, then the *whole enclosing function* — bugs in untouched
  lines of a touched function are in scope. For each line: what input, state, timing, or platform
  makes it wrong? Inverted conditions, off-by-one, nil/null deref, missing `await`, falsy-zero
  treated as "missing", copy-paste with the wrong variable, swallowed errors, unescaped regex.
- **B. Removed behavior.** For every deleted or replaced line, name what it guaranteed (a guard, a
  validation, an error path, a test case), then find where the new code guarantees it again. Not
  found → candidate.
- **C. Callers and callees.** For each changed function, grep its callers: new precondition,
  changed return shape, new error, changed ordering? Does another change in the same diff make an
  existing call unsafe?
- **D. Language / framework pitfalls.** Go nil-map write, loop-variable capture, unchecked error;
  JS `==` coercion, falsy `0` / `""`, closure over loop var, unhandled promise; Python mutable
  default args, late-binding closures; SQL injection, missing transaction, N+1; timezone/DST; float
  equality; integer overflow.
- **E. Wrappers.** For a cache / proxy / decorator / adapter: does every method forward to the
  wrapped object (not back through a global registry, which re-enters or recurses)? Does it
  forward every method callers actually use?
- **F. Requirement fit.** Does the behavior match the brief's spec, including edge cases the user
  mentioned? Anything changed that the request did not ask for?
- **G. Permissions, visibility, data exposure.** Can a user see or change what they should not?
  Hidden / draft / admin-only items leaking through a new endpoint, list, search, or export?
- **H. Concurrency and repetition.** Two requests at once, the same request twice, retry after a
  partial failure, a crash mid-way — still correct?
- **I. Root cause vs. bandaid.** Does the change fix the cause, or special-case a symptom on top of
  shared code? Name the deeper fix if there is one.
- **J. Test gaps.** Which realistic case would break this and is not covered by any test? (Reported
  as a finding only if you can show the case actually fails — otherwise list it under "not covered".)

Reuse / simplification / efficiency suggestions are **out of focus**. Report one only when it has
a concrete, demonstrable cost (measurably slower on a realistic input; duplicated logic that has
*already* diverged), and rank it below every correctness finding. Pure style is never reported.

## 4. The reproduction gate

Every candidate is turned into something executable and **actually run**:

| Kind of bug | Typical repro |
|---|---|
| logic / edge case | a failing unit test, or a small script calling the function |
| API / backend | `curl` (or equivalent) against a local or test server, with the exact request and response |
| data / migration | run against a scratch DB or a copy; show the rows before / after |
| concurrency | a script firing N parallel requests or goroutines; report how many runs out of how many failed |
| UI | browser steps + screenshot (section 5) |

Each candidate ends in exactly one status:

- **`REPRODUCED`** — you ran it and got the wrong result. Include the command(s) and the real
  output excerpt next to the expected output. Only this status counts as a confirmed finding.
- **`UNREPRODUCED`** — the mechanism looks real but you could not trigger it. Say exactly what
  blocked you (missing environment, needs a timing window, needs production-only data) and what
  would confirm it. Listed separately, after all reproduced findings — never mixed in with them.
- **`REFUTED`** — you ran it and the behavior was correct, or a guard elsewhere prevents it (cite
  the line). Dropped from the report body; listed in one line each at the end so the orchestrator
  knows it was checked.

Hard rules:

- **No run, no `REPRODUCED`.** "Clearly wrong by reading" is `UNREPRODUCED` until something runs.
- **Paste real output**, trimmed to the relevant lines. Never paraphrase what a command printed.
- **The repro must be pasteable**: absolute paths, exact commands, any setup steps, in order.
  Someone else should get the same result by copy-pasting it.
- **Reproduce safely**: local or test environment only, test accounts only, scratch files outside
  the reviewed branch, clean up servers / temp DBs / browser tabs you opened.
- If the environment truly cannot run anything (no runtime, no network), say so at the top of the
  report; every finding is then `UNREPRODUCED` and the orchestrator must know the gate was not met.

## 5. UI review (when the diff touches pages)

If the diff changes anything a user sees (components, styles, templates, page routes, static
assets), dispatch a **separate UI reviewer** in parallel with the code reviewer. It opens the real
rendered page in a test/local environment with whatever browser automation is available (Claude in
Chrome, Playwright, the harness's built-in browser / preview), performs the interactions the change
affects, and checks [references/ui-review-checklist.md](references/ui-review-checklist.md):
overlap and occlusion, elements that cannot be clicked or reached, text overflow / truncation,
desktop (1440×900) and mobile (390×844) widths, light and dark themes, console errors, keyboard
focus, loading / empty / error states.

Every UI finding carries: the URL, the viewport, the exact steps, a screenshot path, and what
should have happened. Same status rules as section 4 — a UI problem seen in a screenshot is
`REPRODUCED`; one inferred from CSS alone is `UNREPRODUCED`.

Do not use desktop "computer use" control unless the user explicitly asked for it; browser
automation tools are enough. If no browser tool works, the UI reviewer says so plainly — it must
not claim the page was visually checked.

## 6. How many reviewers

| Diff size | Dispatch |
|---|---|
| Small (one concern, a few files) | 1 code reviewer (+ 1 UI reviewer if pages changed) |
| Medium | 1 code reviewer covering all angles + 1 UI reviewer if pages changed |
| Large (many files, several concerns, risky infra) | 2–3 code reviewers, angles split between them (e.g. A+B+D / C+E+H / F+G+I+J) + UI reviewer if pages changed; then 1 **verifier** that re-runs every `REPRODUCED` repro cold; then 1 **gap sweep** reviewer given the verified list and told to find only what is *not* on it |

Independent reviewers go out in a single message so they run in parallel. Verifier and sweep run
after the first wave returns. Two reviewers flagging the same line for different reasons are two
findings, not one.

## 7. What a reviewer returns

A short header (what was reviewed, which commands ran, environment limits), then findings ranked
most severe first:

```markdown
### 1. [high] backend/handlers/submit.go:142 — retry re-inserts the submission
- Status: REPRODUCED
- Trigger: client retries POST after a timeout while the first request is still committing
- Repro:
    cd /abs/path/worktree
    go test ./backend/handlers -run TestSubmitRetryDuplicate -count=20
- Expected: exactly 1 row in submissions
- Actual: 2 rows in 7/20 runs
    --- FAIL: TestSubmitRetryDuplicate (0.04s)
        submit_test.go:88: got 2 submissions, want 1
- Evidence: /abs/path/scratch/submit_retry_test.go (the test file used)
```

Severity: `high` (wrong data, security / permission hole, crash, data loss), `medium` (wrong
behavior on a realistic path), `low` (edge case, cosmetic UI defect, measurable inefficiency).

Then the `UNREPRODUCED` list (same fields, plus "blocked by" and "would confirm by"), then one line
per `REFUTED` candidate, then "not covered by tests" cases. Zero findings is a valid result — say
"no reproduced findings" rather than padding.

## 8. What the orchestrator does after

1. **Dedup** across reviewers (same defect, same place, same cause → keep the one with the best repro).
2. For each `REPRODUCED` finding: fix it (or dispatch a fixer subagent), then **re-run the same
   repro** and confirm it now passes. Where the repro is a test, keep it as a regression test.
3. `UNREPRODUCED` findings: decide — try a better repro, fix defensively, or note why not.
   High-severity ones are not silently ignored.
4. If fixes were large, dispatch a fresh reviewer on the fix diff (not the same reviewer, not the fixer).
5. **Record it** where the next reader will look (PR description `## Code Review` section, or the
   final reply): each finding → status → what changed → commit; each unfixed finding → why
   (false positive / intentional / out of scope, reported separately). Mention if any part of the
   gate was not met (no browser available, no runnable environment).

## 9. Common failures

- The reviewer only read the diff and reported plausible-sounding bugs with no runs.
- The orchestrator's guess was written into the brief and came back "confirmed".
- The implementer reviewed its own code, or was told to spawn its own reviewer.
- UI was judged from JSX / CSS without opening the page.
- A repro was run against production, or with a real user's account.
- A fix was declared done without re-running the repro that demonstrated the bug.
- `UNREPRODUCED` items were mixed into the confirmed list, or `REFUTED` ones were dropped without
  a trace so nobody knows they were checked.
