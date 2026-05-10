---
name: self-review-thayto
description: Use whenever the user wants to self-review their own pull request before flagging it for human review — runs the same multi-agent review (security, performance, TypeScript, code-quality) as `pr-review-opinionated-thayto`, then applies the `address-my-prs-thayto` rubric (apply / escalate / skip) to act on each finding locally without posting anything to GitHub. Triggers on `/self-review-thayto`, "self-review", "self review my PR", "review my own PR before sending it out", "pre-flight my PR", "review and address my own PR", "fix my PR before review", "review my PR myself first", or pasting the user's own PR URL with phrasing like "review this and fix it" or "go through this myself first". Operates on a single PR — the current branch's PR (resolved via `gh pr view --json number`) or a pasted github.com/<owner>/<repo>/pull/<n> URL. NOT for reviewing someone else's PR (use pr-review-opinionated-thayto). NOT for batch-triaging multiple PRs (use address-my-prs-thayto).
---

# Self-review my PR

Self-review a single pull request authored by the user before flagging it for human review. Runs the multi-agent review pipeline from `pr-review-opinionated-thayto` (security, performance, TypeScript best-practices, code-quality) over the PR diff, then applies the `address-my-prs-thayto` rubric (apply / escalate / skip) to act on each finding. **Findings are never posted to GitHub** — they live in memory, get resolved locally in the working tree, and the user reviews the diff and commits themselves.

## What this skill does to the target repo

- **Reads on GitHub:** PR metadata, diff, current commit SHA. Read-only against the GitHub side; nothing is posted.
- **Writes locally:** edits files in the working tree of the current checkout to apply review findings the rubric clears.
- **Never:** posts a review, leaves inline comments, resolves threads, force-pushes, branches, stages, commits, pushes, rebases, or touches any git ref. The user reviews the resulting working-tree diff and commits.
- **Scope:** one PR per invocation — the PR for the current branch, or one whose URL the user pasted. Not for batch self-review across multiple PRs.

## Hard rules

- **Self-review only.** If `pr.author.login != $GH_LOGIN`, stop and tell the user this skill is for their own PRs (point them at `pr-review-opinionated-thayto` for someone else's). Reason: the apply/escalate rubric is calibrated for editing your own code in your own working tree; applying findings to someone else's branch is out of scope and silently wrong.
- **No `gh api` mutations.** No review submission, no inline comments, no thread resolution, no reviewer re-request. The whole point of the skill is the in-memory loop. Reason: a self-`REQUEST_CHANGES` clutters the PR timeline forever, and a self-`APPROVE` is impossible on GitHub anyway.
- **No git mutations.** No commits, no pushes, no rebases, no checkouts, no `git add`. The user owns the commit boundary. Reason: the user wants to see exactly which findings were applied, in one diff, before deciding what to ship — same model as `simplify-thayto`.
- **Skip bot PRs.** If author matches `*[bot]` or `github-actions`, stop. Reason: there is no "self" to review.
- **Read project standards before editing.** CLAUDE.md, contributing docs, lint config, and patterns elsewhere in the repo override generic best practices. The user has emphasized this — apply project conventions, not your defaults.
- **Never widen the change beyond the PR diff.** If a finding requires touching code outside the diff, escalate; do not silently rewrite adjacent files. Reason: the user's mental model for this skill is "self-review *the diff*", not "rewrite whatever you notice while you're there".

## Workflow

### 1. Resolve the PR

Two paths:

```bash
# A) The user pasted a PR URL → use it directly.
PR_URL="https://github.com/<owner>/<repo>/pull/<n>"

# B) No URL → resolve the PR from the current branch.
PR_NUMBER=$(gh pr view --json number -q .number)   # fails loudly if no PR exists for this branch
```

If neither resolves, stop and ask: "No PR for the current branch and no URL provided. Open a PR first or paste the URL."

Then load metadata and diff:

```bash
gh pr view "$PR" --json author,title,number,headRefName,baseRefName,headRepository,baseRepository,files,reviews,comments,state,isDraft,statusCheckRollup
gh pr diff "$PR"
```

### 2. Verify identity

```bash
gh auth status                            # fail loudly if unauthenticated
GH_LOGIN=$(gh api user --jq .login)
```

Stop if `pr.author.login != $GH_LOGIN`. Stop if author matches `*[bot]` or `github-actions`. Announce one line back to the user before doing anything else:

> `Self-reviewing PR #<n> "<title>" on <repo> (<head> → <base>) as @$GH_LOGIN.`

### 3. Dispatch parallel review agents

Spawn all four agents in **a single message with multiple Agent tool uses** so they execute concurrently. Pass each the diff plus PR metadata. Each returns findings as a JSON list of `{file, line, severity, comment, suggestion}` and nothing else.

- **Security agent**: auth bypass, injection, secret leakage, unsafe deserialization, prototype pollution, command injection, missing input validation at trust boundaries, broken access control, unsafe shell calls, dependency risks, regex DoS.
- **Performance agent**: quadratic loops over user-controlled input, awaits inside loops where parallelism applies, sync I/O on hot paths, missed memoization where re-renders or recompute matters, N+1 calls, oversized bundle imports, blocking work on the main thread.
- **TypeScript best-practices agent**: `any`/`unknown` misuse, missing discriminated unions where state machines exist, unsound casts, narrow types where wider would help (or vice versa), missing `as const`, `Promise<void>` where errors are silently swallowed, exported public APIs without explicit return types. Skip this agent if the PR has no TypeScript files.
- **Code-quality agent**: flag and rewrite **nested ifs/else**, **nested ternaries**, and **repeated code** — the highest-priority dislikes. Suggest early returns, lookup objects, or extracted helpers. Also flag dead code, misleading names, and obvious bugs.

Each agent must:

- Only flag issues inside the PR diff.
- Always include a concrete `suggestion` (the actual fix), not just a description of the problem.
- Never include effort estimates ("small change", "quick fix", "easy win", "low effort"). Drop the framing entirely.

Use `subagent_type: general-purpose` (or a code-review specialist if your harness exposes one).

### 4. Aggregate findings

Dedup overlapping findings across agents. Drop anything outside the diff. Drop style-only findings without a concrete suggestion. Print the merged list to the user as a numbered plan **before touching any file**:

```
Self-review of PR #<n> — N findings:
  1. [security]      src/foo.ts:42 — <one-line summary>
  2. [code-quality]  src/bar.ts:88 — <one-line summary>
  ...
```

This is the user's chance to bail or redirect before the working tree changes.

### 5. Apply the rubric per finding

For each finding, decide using the same rubric as `address-my-prs-thayto`. The user explicitly does not want findings applied blindly — even their own.

**Apply** (edit the file, do NOT commit) when:

- It's a clear bug, type error, or correctness issue verifiable from the diff.
- It enforces a project standard documented in CLAUDE.md, contributing guidelines, or lint config, or matches established patterns elsewhere in the repo.
- It's a small refactor (rename, extract, dedupe, early return) where the suggestion is plainly cleaner with no behavior change.
- The change is under ~30 lines and self-contained.

**Escalate to the user inline** (stop and ask, do NOT edit) when:

- The finding would require redesign, new abstractions, or scope creep beyond the PR's stated goal.
- The fix is non-obvious enough that the user's judgment is genuinely needed.
- Two agents disagreed on the same line, or one agent contradicted a pattern visible elsewhere in the repo.
- The finding asks to remove deliberate behavior the diff plainly intends.

Escalation surfaces here, not on GitHub: ask the user one focused question per escalated finding, or batch them via `AskUserQuestion`. Wait for the answer before moving on. Never silently apply, never silently drop.

**Skip silently** when:

- The finding is praise-equivalent or duplicates one already applied.
- The cited line was already changed by an earlier applied finding in the same pass (re-running the agent over the new line would be cheaper than guessing — when in doubt, skip and note it).

When uncertain, **escalate**. The cost of silently rewriting your own code in a way you'd object to is much higher than one extra question to the user.

### 6. Report

After the loop, print one summary:

```
PR #<n> — <title>

Findings: N total
  applied:    A   (file:line list)
  escalated:  E   (with the user's decision per item)
  skipped:    S   (one-line reason each)

Working-tree diff is ready for your review. Nothing was committed or pushed.
Run `git diff` to inspect, then commit when you're happy with it.
```

If the four agents returned nothing applicable, say so plainly: `No findings worth applying. PR looks clean to flag for review.` Do not invent findings to look thorough.

## Why this skill exists

Pre-flight is the highest-leverage moment to fix your own code: the diff is fresh in your head, no reviewer is waiting, and the cost of rework is zero. `pr-review-opinionated-thayto` produces the same dense signal you'd give a teammate; `address-my-prs-thayto` provides the rubric for acting on review feedback without applying everything blindly. Combined, they're the "review yourself the way you'd review someone else, then act on it the way you'd act on a real reviewer's notes" loop — kept entirely local so the PR timeline stays clean and you ship the result in one hand-reviewed commit.

## Failure & escalation

If something prevents the loop from completing cleanly — agent dispatch fails, a finding's fix would touch code outside the diff, the PR has zero diff (already merged or empty), the user pastes a URL for a PR they didn't author — stop and report it rather than guessing. The skill's value is doing the boring 80% reliably; the user takes the remaining 20% by hand.
