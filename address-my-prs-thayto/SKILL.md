---
name: address-my-prs-thayto
description: Use whenever the user wants to triage, babysit, or address feedback across all of their open pull requests on the current GitHub repo at once. Triggers on phrases like "check my PRs", "address PR comments on my PRs", "fix CI on my PRs", "triage my open PRs", "babysit my PRs", "go through my pull requests", or pasting a PR-list URL like https://github.com/<owner>/<repo>/pulls (note: list URL, not a single /pull/<n>). Triggers when the user references multiple of their own PRs (author resolved dynamically from `gh api user` and filtered with `--author @me` — never hardcoded; e.g. rafa-thayto when that's the authenticated login) and wants review comments addressed, broken CI fixed, base-branch rebases performed, threads resolved silently (the skill never replies on a thread — only resolves or escalates), and reviewers re-requested. NOT for reviewing a single PR as a reviewer (use pr-review-opinionated-thayto). NOT for opening a new PR.
---

# Address my open PRs

Triages every open PR authored by the user on the current GitHub repo. The repo slug is detected from `gh repo view --json nameWithOwner -q .nameWithOwner`; the author filter is `--author @me`. This skill **writes to the target repo**: it makes commits, force-pushes with `--force-with-lease`, resolves review threads via GraphQL, and re-requests review via `gh api`. State that to the user up front when the skill fires, so they know what is about to happen on their behalf.

## Hard rules

- **One worktree per PR.** Parallel subagents on the same checkout will corrupt each other's index. Use `git worktree add` per PR. Never `git checkout` the trunk to switch.
- **Never reply on a review thread, ever.** The skill resolves; it does not converse. No "done", no "good catch", no "I disagree because…", no clarifying questions, not a single character of comment-body output to GitHub. The only thread-state change this skill ever makes is `resolveReviewThread`. If a comment needs a reply (discussion, pushback, a question), escalate it to the user with the thread left open — the user replies, not the skill.
- **Never resolve a thread you did not address.** Resolution implies "done"; using it to clear backlog is dishonest to the reviewer.
- **Never `--no-verify`, `--no-edit` on rebase, or plain `--force`.** Use `--force-with-lease` after rebase. If a hook fails, fix the underlying issue.
- **Confirm before rebasing an approved PR.** On most repos, rebase invalidates approvals. If the PR has approving reviews, ask the user before rebasing.
- **Skip drafts and bot-authored PRs.** Drafts are work-in-progress; the user shouldn't be addressing comments on them yet.
- **Read project standards before editing.** CLAUDE.md, contributing docs, lint config, and patterns elsewhere in the repo override generic best practices. The user emphasized this — apply project conventions, not your defaults.

## Comment-evaluation rubric

The user explicitly does not want comments applied blindly. Before changing code in response to a thread, fetch enough context — the surrounding file, the PR description, related tests — to judge the comment on its merits.

**Apply and resolve** when:

- It's a clear bug, type error, or correctness issue that is verifiable from the diff.
- It enforces a project standard that is documented in CLAUDE.md, contributing guidelines, lint config, or matches established patterns elsewhere in the repo.
- It's a small refactor (rename, extract, dedupe, early-return) where the reviewer's version is plainly cleaner with no behavior change.
- The change is under ~30 lines and self-contained.

**Escalate to the user (do NOT resolve, leave the thread open)** when:

- The reviewer is wrong on the merits, contradicts a project pattern, or asks to remove deliberate behavior.
- The change would require redesign, new abstractions, or scope creep beyond the PR's stated goal.
- The comment is a question rather than a request — the user should answer.
- Two reviewers disagree on the same line.
- The required fix is non-obvious enough that the user's judgment is genuinely needed.

**Skip silently (do NOT resolve, do NOT escalate)** when:

- The thread is already resolved upstream or marked `outdated`.
- The comment is praise/ack ("nice!", "lgtm").

When uncertain, **escalate**. The cost of a missed escalation (silently overriding a reviewer's intent) is much higher than one extra question to the user.

## Workflow

### 1. Verify identity & repo, then discover PRs

Resolve the authenticated user and the target repo from local git/gh state — never hardcode a login or repo slug, and never silently rely on a single source.

```bash
# 1a. Authenticated user (the source of truth for `--author @me`)
gh auth status                              # fail loudly if unauthenticated
GH_LOGIN=$(gh api user --jq .login)         # who @me resolves to on GitHub
GIT_EMAIL=$(git config user.email)          # local git identity (announce only)

# 1b. Repo derived from the local origin remote, NOT just gh's default.
#     We push to origin, so origin is what we mutate. In a fork checkout,
#     gh's default may resolve to the upstream — that's fine, but we still
#     operate on origin.
ORIGIN_URL=$(git remote get-url origin)
REPO=$(gh repo view "$ORIGIN_URL" --json nameWithOwner -q .nameWithOwner)
GH_DEFAULT=$(gh repo view --json nameWithOwner -q .nameWithOwner)

if [ "$REPO" != "$GH_DEFAULT" ]; then
  echo "note: git origin = $REPO, gh default = $GH_DEFAULT — operating on origin"
fi
```

Announce one line back to the user before fetching anything mutable:

> `Authenticated as @$GH_LOGIN ($GIT_EMAIL); operating on $REPO.`

If `gh auth status` fails, the user has previously named a different login in this conversation, or the announced login doesn't match what the user expects, **stop and ask** before listing PRs. Do not silently operate on a different account's PRs.

Then list:

```bash
gh pr list --repo "$REPO" --author @me --state open \
  --json number,title,headRefName,baseRefName,url,isDraft,reviewDecision,mergeable
```

`--author @me` is server-resolved to `$GH_LOGIN`, so the filter is identity-correct without hardcoding any login. `--repo "$REPO"` pins it to origin even if the cwd's gh default differs.

Filter out drafts. For each remaining PR, dispatch one subagent (see step 2). Cap at 5 in flight; if the user has more, queue.

### 2. Per-PR subagent (run in parallel)

Dispatch all agents in a **single message with multiple Agent tool uses** so they execute concurrently. Use `subagent_type: general-purpose`. Each agent receives: PR number, head branch, base branch, repo slug, and a copy of this rubric.

Inside each agent, work in a dedicated worktree:

```bash
git fetch origin <headRefName>
git worktree add ../wt-<headRefName> <headRefName>
cd ../wt-<headRefName>
```

Then run the per-PR pipeline:

#### 2.1 Pull context

```bash
gh pr view <n> --json title,body,author,headRefName,baseRefName,reviews,reviewRequests,comments,statusCheckRollup,mergeable,mergeStateStatus
gh pr diff <n>
```

Also fetch unresolved review threads via GraphQL — REST does not expose `isResolved`:

```graphql
query($owner:String!, $name:String!, $num:Int!) {
  repository(owner:$owner, name:$name) {
    pullRequest(number:$num) {
      reviewThreads(first:100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          comments(first:20) {
            nodes { id author{login} body path line diffHunk createdAt }
          }
        }
      }
    }
  }
}
```

Run with:

```bash
gh api graphql -F owner=<owner> -F name=<name> -F num=<n> -f query='...'
```

Discard threads where `isResolved` or `isOutdated` is true.

#### 2.2 CI triage

If any check is `failure` or `error` and there are zero unresolved threads, the user wants the failure diagnosed and fixed. Get the failing run:

```bash
gh pr checks <n>
gh run view <run-id> --log-failed
```

Reproduce locally if cheap (lint/typecheck/unit tests). For flaky-looking failures, retry once via `gh run rerun <run-id> --failed` before editing code.

If checks fail **and** there are unresolved threads, address the threads first — the fix may already be among them.

#### 2.3 Thread loop

For each unresolved thread, apply the rubric above. For threads you decide to apply: edit the file, stage the change. Track which thread IDs were addressed in a list — you'll need them for step 2.6.

#### 2.4 Rebase if needed

If the PR is behind `origin/<baseRefName>`:

```bash
git fetch origin <baseRefName>
git rebase origin/<baseRefName>
```

Resolve conflicts only when the resolution is mechanical and obvious (e.g., both sides moved an import). Otherwise `git rebase --abort` and escalate to the user.

If the PR has any approving review, **stop and ask the user before rebasing** — most repos drop approvals on force-push.

#### 2.5 Commit & push

```bash
git commit -m "address review feedback"   # match the repo's commit style; check `git log` first
git push --force-with-lease origin <headRefName>   # if rebased
# otherwise:
git push origin <headRefName>
```

Use the repo's commit message convention. Check `git log --oneline -20` to mirror tense, prefix style, and length.

#### 2.6 Resolve addressed threads

For each thread ID you applied a change for:

```bash
gh api graphql \
  -f query='mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){ thread{id isResolved} } }' \
  -F id=<thread-id>
```

Do not call this for threads you escalated or skipped.

#### 2.7 Wait for CI

```bash
gh pr checks <n> --watch
```

Cap at ~10 minutes. If still red after the wait, escalate to the user with the failing job summary.

#### 2.8 Re-request review

Only when **CI is green AND every applied thread is resolved AND nothing was escalated**:

```bash
gh api -X POST repos/<owner>/<repo>/pulls/<n>/requested_reviewers \
  -f 'reviewers[]=<login1>' -f 'reviewers[]=<login2>'
```

Reviewers come from the PR's prior `reviews` (latest review per reviewer, dedup) plus any `reviewRequests` still pending. Skip yourself.

#### 2.9 Cleanup

Return to the original directory. **Do not remove the worktree** — the user may want to inspect or hand-fix something. Print a structured summary back to the parent agent:

```
PR #<n> — <title>
  threads addressed: N
  threads escalated: M (with one-line reasons)
  threads skipped: K
  CI fix: yes/no — <one-line>
  rebased: yes/no
  reviewers re-requested: <list> | skipped because <reason>
  worktree: ../wt-<branch>
```

### 3. Aggregate report

After all subagents return, print one table to the user:

| PR | Title | Addressed | Escalated | CI fix | Rebased | Re-requested | Worktree |
|----|-------|-----------|-----------|--------|---------|--------------|----------|

Then, for each PR with escalations, list the escalated threads with the comment URL and a one-line reason — these are the items the user needs to look at.

## What this skill does to the target repo

- **Reads:** PR list, PR metadata, diffs, review threads, CI logs.
- **Writes (per PR):** commits to the head branch; `git push` (or `git push --force-with-lease` after rebase); resolves review threads via GraphQL; re-requests review via REST. Creates worktrees under `../wt-<branch>` and leaves them in place.
- **Never:** opens new PRs, closes PRs, merges PRs, replies on review threads, edits PR titles/descriptions, modifies the trunk checkout.

## Failure & escalation

If anything in the pipeline cannot complete cleanly — rebase conflicts, CI fix that needs design judgment, a thread that needs discussion, an approving review on a PR that needs rebase — stop that PR's pipeline and report it under "escalated" rather than guessing. The skill's value is doing the boring 80% reliably; the user takes the remaining 20% by hand.
