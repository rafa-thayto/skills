---
name: pr-review-opinionated-thayto
description: Review a GitHub pull request and post the review through gh. Use whenever the user asks to review a PR, pastes a github.com/<owner>/<repo>/pull/<n> URL, or says things like "review this PR", "check this PR", "look at this pull request", "give me a review on this", even when they don't explicitly say "review". Performs a multi-agent parallel review covering security, performance, TypeScript best practices, and code quality, then posts inline review comments and either approves with LGTM or requests changes with an empty top-level body.
---

# Review a GitHub PR

Review a pull request on GitHub and post the result through the `gh` CLI. The local repo is **read-only for this skill**: never run `git commit`, `git push`, `git checkout`, or modify any files on disk. All output goes to GitHub via `gh`.

## Workflow

1. **Fetch the PR.** Use `gh pr view <url> --json author,title,number,headRefName,baseRefName,headRepository,baseRepository,files,reviews,comments,state,isDraft` plus `gh pr diff <url>` to load metadata and the diff.
2. **Skip bots.** If `author.login == "github-actions"` or matches `*[bot]`, stop and tell the user the PR is bot-authored and was skipped. Do not review bot PRs.
3. **Check prior review state.** Look at existing review comments left by you or the user on this PR. If the latest commits address them, lean toward approval. See "Re-review".
4. **Dispatch parallel review agents.** Launch all agents from "Parallel agents" in a single message so they run concurrently. Each returns structured findings.
5. **Merge findings.** Dedup overlapping comments across agents. Drop anything outside the diff. Drop anything that is style-only with no concrete suggestion.
6. **Post the review.** Use `gh api` to attach inline comments to specific lines, then submit the review with `APPROVE` or `REQUEST_CHANGES`. Submit with an empty body for change requests, or exactly `LGTM` for approvals. See "Posting via gh".
7. **Confirm to the user.** Report what was posted (count of inline comments + verdict) and the review URL.

## Parallel agents

Dispatch these in one message via the Agent tool. Pass each agent the diff plus PR metadata. Each agent must return findings as a JSON list of `{file, line, severity, comment, suggestion}` and nothing else.

- **Security agent**: auth bypass, injection, secret leakage, unsafe deserialization, prototype pollution, command injection, missing input validation at trust boundaries, broken access control, unsafe shell calls, dependency risks, regex DoS.
- **Performance agent**: quadratic loops over user-controlled input, awaits inside loops where parallelism applies, sync I/O on hot paths, missed memoization where re-renders or recompute matters, N+1 calls, oversized bundle imports, blocking work on the main thread.
- **TypeScript best-practices agent**: `any`/`unknown` misuse, missing discriminated unions where state machines exist, unsound casts, narrow types where wider would help (or vice versa), missing `as const`, `Promise<void>` where errors are silently swallowed, exported public APIs without explicit return types. Skip this agent if the PR has no TypeScript files.
- **Code-quality agent**: flag and rewrite **nested ifs/else**, **nested ternaries**, and **repeated code**. These are the highest-priority dislikes. Suggest early returns, lookup objects, or extracted helpers. Also flag dead code, misleading names, and obvious bugs.

Each agent must:
- Only flag issues inside the PR diff.
- Always include a concrete `suggestion` (the actual fix), not just a description of the problem.
- Never include effort estimates ("this is a small change", "quick fix", "low effort", "easy win"). Drop the framing entirely.

Pick whichever subagent type your harness exposes (e.g. `general-purpose`, or a code-review specialist if available). Spawn them in a **single message with multiple tool uses** so they execute in parallel.

## Comment style

Comments are posted to GitHub and read by humans. Tone matters.

- Friendly, concise, human. Sound like a teammate, not a linter.
- One short paragraph max, plus the suggestion.
- Talk about the **code change**, not the PR itself. Do not write things like "Nice PR!", "Thanks for working on this", "This PR introduces…". Go straight to the code observation.
- Always include a fix suggestion. Use a GitHub `suggestion` block when the fix replaces specific lines:
  ````markdown
  ```suggestion
  const user = users.find((u) => u.id === id)
  if (!user) return null
  return user.email
  ```
  ````
- Never use em dash `—`, en dash `–`, or horizontal bar `―`, anywhere, including inside suggestion blocks. Use a comma, period, or parentheses instead. This is absolute.
- No effort labels ("nit:", "small:", "minor:", "low-effort:"). Just say what to change.

## Re-review (the PR was already reviewed once)

If your prior comments are visible on the PR:

- For each prior comment, check the latest diff or thread state. Resolved or addressed: do not re-comment.
- Only add a *new* comment if it is **extremely necessary** (real bug, security issue, broken behavior). Polish-level findings do not qualify on a re-review.
- If everything from the prior round is addressed and nothing critical is new, **approve**.
- When approving, the approval body is exactly `LGTM`. No extra text, no em dash, no trailing thanks.

## Approval rule

If after the parallel pass there are no findings worth posting:
- Submit `APPROVE` with body `LGTM`.
- Do not invent comments to look thorough.

If there are findings:
- Submit `REQUEST_CHANGES` with an **empty body** (`-f body=""`). All feedback lives in the inline comments.

## Posting via gh

Inline comments must attach to lines that exist in the diff. Use the GitHub review API in two steps so you can batch comments and submit one review.

```bash
# 1. Resolve owner/repo/number from the PR URL the user gave you.
#    Example URL: https://github.com/<owner>/<repo>/pull/<number>
OWNER=...
REPO=...
PR=...

# 2. Get the latest commit SHA on the PR head (required for line-anchored comments).
COMMIT_SHA=$(gh api "repos/$OWNER/$REPO/pulls/$PR" --jq .head.sha)

# 3. Build the comments array. Each entry: { path, line, side, body }.
#    `line` is the line number in the file (RIGHT side for additions).
#    For multi-line, add `start_line` and `start_side`.
#    Put `gh suggestion` blocks inside `body`.

# 4. Submit one review with all comments. Empty body, single event.
gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
  -X POST \
  -f commit_id="$COMMIT_SHA" \
  -f event="REQUEST_CHANGES" \
  -f body="" \
  -F "comments=@/tmp/pr-review-comments.json"
# Use APPROVE + body="LGTM" instead when approving (still pass an empty comments array if no inline comments).
```

The `comments` payload is a JSON array like:

```json
[
  {
    "path": "src/foo.ts",
    "line": 42,
    "side": "RIGHT",
    "body": "Nested ternary here is hard to read. Consider an early return.\n\n```suggestion\nif (!user) return null\nreturn user.isAdmin ? 'admin' : 'member'\n```"
  }
]
```

If a finding does not map to a specific line (e.g., an architectural concern), anchor it to the closest relevant line as an inline comment. Still no top-level body.

## Hard rules

- **Read-only on the filesystem.** No `git commit`, no `git push`, no `git checkout` of the PR branch, no edits to local files. All review action goes through `gh`.
- **No top-level review body.** The submitted review's body is `""` for change requests, `"LGTM"` for approvals. Nothing else.
- **No em dash, en dash, or horizontal bar** anywhere in posted output.
- **No effort estimates.** Drop "small change", "quick fix", "easy win", "low effort", etc.
- **No meta-PR commentary.** No "great work", "thanks for this", "this PR…". Talk about the code.
- **Always include a suggested fix** in every comment, ideally as a `suggestion` block.
- **Skip `github-actions` and `*[bot]` authors.** Tell the user the PR was skipped, do not review.

## Why these rules exist

This skill encodes the user's calibration of an experienced human reviewer: dense signal, no fluff, always actionable. Top-level review bodies become noise after a few PRs. Effort estimates are condescending and often wrong. Em dashes are a tell that a comment was written by an LLM. Bot PRs auto-resolve and reviewing them wastes everyone's time. Re-reviewing aggressively when the author already addressed the first round burns trust. `LGTM` is the team shorthand for a clean approval.
