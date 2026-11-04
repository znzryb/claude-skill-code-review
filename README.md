# repro-code-review

A Claude Code skill that tells the **main agent how to have subagents review code** — with a hard
rule that every reported finding comes with a reproduction the reviewer actually ran.

- Independent reviewers (never the implementer), all dispatched by the main agent.
- Search angles adapted from Claude Code's built-in `/code-review` (line-by-line, removed
  behavior, cross-file callers, language pitfalls, wrapper forwarding), plus requirement fit,
  permissions, concurrency, and root-cause checks. Rewritten, not copied.
- **Reproduction gate**: each finding is `REPRODUCED` (ran, wrong output pasted), `UNREPRODUCED`
  (plausible, blocker stated), or `REFUTED`. No run, no confirmed finding.
- A separate **UI reviewer** opens the rendered page in a browser and checks overlap, clickability,
  overflow, mobile width, themes, and console errors, with screenshots.
- Scales from one reviewer to parallel reviewers + a cold verifier + a gap sweep for large diffs.

See [SKILL.md](SKILL.md), [references/reviewer-briefs.md](references/reviewer-briefs.md), and
[references/ui-review-checklist.md](references/ui-review-checklist.md).

## Install

As a git submodule in a project:

```sh
git submodule add https://github.com/znzryb/claude-skill-code-review.git .claude/skills/repro-code-review
```

Or for all projects on your machine:

```sh
git clone https://github.com/znzryb/claude-skill-code-review.git ~/.claude/skills/repro-code-review
```

## License

MIT
