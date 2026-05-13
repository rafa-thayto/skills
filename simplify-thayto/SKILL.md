---
name: simplify-thayto
description: Use whenever the user wants to review and simplify the code they just wrote on the current branch — refactor with KISS and DRY, kill duplicates, strip unnecessary comments, then fix TypeScript errors and warnings, then verify tests pass. Triggers on `/simplify-thayto`, and on phrases like "simplify our changes", "simplify the code we just wrote", "review and improve our changes", "DRY this up", "refactor with KISS", "deduplicate this", "tighten up the diff", "clean up what we did", "kill the duplication", "remove unnecessary comments", "strip dead comments", "kill the comment noise", or any equivalent ask to revisit and tighten the branch diff before shipping. Applies hard structural rules — avoid classes, no nested ifs, no nested ternaries, prefer small functions — keeps only WHY comments and removes WHAT-comments / tombstones / ticket references, and delegates type-level work to the `typescript-advanced-types` skill so types stay simple. Never creates a branch and never commits; edits the working tree in place and runs the test suite locally so the user reviews the diff themselves.
---

# simplify-thayto

Refactor the current branch's diff for KISS/DRY, fix TypeScript noise, run the tests. **Read this once, then work the four passes in order.** Each pass has a single job; don't blur them — that's how scope creeps and the diff stops being reviewable.

## Write surface (read first)

This skill **edits source files in the current checkout** and **runs the test suite locally**. It does not:

- create branches (`git checkout -b`, `git switch -c`) — explicitly disallowed by the user's standing instruction
- commit, stage with `git add`, push, or touch any git ref
- open PRs, post comments, or call any remote API

If you find yourself reaching for `git checkout` or `git commit`, stop — that's out of scope. The user reviews the working-tree diff themselves and commits when they're ready.

## Step 0 — Establish the diff

Default scope is **everything this branch adds vs. `main`** plus uncommitted edits:

```bash
git merge-base HEAD main
git diff $(git merge-base HEAD main)...HEAD
git diff                    # unstaged
git diff --cached           # staged
```

If `main` doesn't exist locally (some repos use `master`, `develop`, or a custom default), fall back to `git remote show origin | sed -n '/HEAD branch/s/.*: //p'` and use that. If the user named a different base ("simplify what's on top of `release/v3`"), use that base instead.

The point: **only touch lines this branch introduced**. Don't refactor unrelated code that happens to be in files you're editing — that pollutes the diff and makes review harder. If you spot something worth fixing outside the diff, mention it at the end so the user can decide.

## Step 1 — KISS / DRY refactor pass

Read the diff with the rules below in mind and apply them. The rules aren't decoration — each one exists because the alternative is the failure mode the user has been bitten by before.

### Hard rules

| Rule | Why it's a rule |
|---|---|
| **Avoid classes.** Prefer modules of small pure functions. | Classes accumulate state and methods that lock callers into one shape. Functions compose freely and are easier to test in isolation. Keep classes only when the problem really is stateful (e.g. a long-lived connection wrapper) and the alternative is uglier. |
| **No nested ifs.** Flatten with early returns / guard clauses. | Nested conditionals hide the path through the function. A flat sequence of guards reads top-to-bottom and each branch is a single hop. |
| **No nested ternaries.** Use a small helper, a `switch`, or an object lookup. | Nested `?:` collapses control flow into an expression that can't be debugged or annotated. Lift it out and name it. |
| **Prefer small functions.** If a function does two things, split it. | Small functions name intent, make diffs surgical, and let you reuse pieces. The win compounds — once one big function is split, duplication elsewhere becomes visible. |

These are preferences with teeth, not absolutes. If breaking a rule is genuinely the cleaner outcome (e.g. a one-off ternary that reads better than a helper, or a class that mirrors a domain object), keep it and note the reasoning at the end.

### Look for duplication

Across the diff, find:

- **Copy-pasted blocks** — even with renamed variables. Extract to a function.
- **Parallel structures** — two functions that differ only in one type or one literal. Parameterise.
- **Repeated literals** — magic numbers or strings used in 3+ places. Pull to a `const`.
- **Repeated guard clauses** — same null/empty check at the top of multiple call sites. Push the check into the helper, or wrap behind a single entry point.

Three is the threshold for "this is a pattern, not a coincidence." Two occurrences is usually fine; deduping prematurely creates abstractions that don't fit later requirements. Don't invent indirection to satisfy DRY when the duplicates aren't really the same concept.

### Strip unnecessary comments

After the structural rewrites land, scan the surviving lines for comments that don't earn their place. Do this *after* dedup and extraction — half the comments you'd have judged are gone with the code they explained, and a few you'd have kept now turn out to be redundant with a freshly named helper. The rubric: **keep WHY, remove WHAT.**

Remove:

- **Paraphrase comments** — `// increment the counter` above `counter++`, `// loop over users` above `for (const user of users)`. The name and the operator already say it.
- **Tombstones and changelog notes** — `// removed X`, `// was: oldFn()`, `// TODO from PR #123`, `// kept for backwards compat (delete after Oct)` when the code below it is current. Git history is the changelog; the file is the present.
- **"Used by" / "called from" pointers** — they go stale silently as callers move. Use `git grep` to find callers, not comments.
- **Task / ticket references inside code** — `// fixes JIRA-1234`, `// added for the onboarding flow`, `// per Bernardo's request`. That context belongs in the commit message and PR description, not the file.
- **Defensive narration of obvious branches** — `// if user is not logged in, redirect` above `if (!user) redirect()`.
- **Redundant docstrings on exported functions** — `/** Returns the user. */` above `export function getUser(id: string): User` adds nothing the signature doesn't carry. Strip JSDoc/TSDoc that re-states the name and types. Keep it when it documents WHY (a constraint, an invariant, a non-obvious caller contract).

Keep:

- **WHY comments** — hidden constraints, invariants, workarounds for specific bugs, surprising behavior a reader would otherwise have to reverse-engineer. These pay rent.
- **Pointers the code itself can't carry** — RFC numbers, vendor bug URLs, regulatory rule citations, links to the discussion that explains a non-obvious choice.
- **License headers and required attribution.**

If a comment is borderline, leave it. The pass is about removing cruft, not enforcing style; deleting a comment a future reader needed is more expensive than skipping one that was merely redundant. Only touch comments on lines this branch added or modified — pre-existing comments outside the diff are out of scope, same rule as everything else in Step 1.

### What to leave alone

- Code outside the branch diff. (Out of scope — see Step 0.)
- Tests, unless they're directly broken by your refactor. Tests are the safety net; rewriting them in the same pass as the code makes regressions invisible.
- Style-only changes (renames, formatting) that aren't load-bearing. Those belong in a separate pass; mixing them with structural refactors hides the structural change in noise.

## Step 2 — TypeScript fix pass

Only after Step 1's refactor is on disk. Don't interleave — finishing the structural pass first means the type errors you fix are the ones that matter, not phantoms that disappear when functions get split.

### Package manager preference

Detect by lockfile first — the lockfile wins, always: `bun.lockb` or `bun.lock` → bun, `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm. Running the wrong one regenerates a sibling lockfile and corrupts the workspace, so this isn't a style preference, it's correctness.

When **no lockfile is present** (fresh init, lockfile gitignored, monorepo root with no manager pinned), prefer **`bun` > `pnpm` > `yarn` > `npm`** — bun is fastest and the user's default. Same preference applies in every command block below.

```bash
# Run the project's type checker. Pick the row that matches the lockfile;
# if none, use the top row (bun).
bun x tsc --noEmit        # bun.lockb / bun.lock
pnpm tsc --noEmit         # pnpm-lock.yaml
yarn tsc --noEmit         # yarn.lock
npx tsc --noEmit          # package-lock.json
```

Read the output and fix:

- **Errors** introduced or surfaced by the diff.
- **Warnings** in files this branch touched. Don't go on a project-wide warning hunt — that's a separate task and pollutes the diff.

For anything that requires non-trivial type machinery — generics that need constraints, conditional types, mapped types, template literals, inferring from a function signature, distributing unions — **invoke the `typescript-advanced-types` skill** rather than improvising. It exists exactly for this and will keep the type code simpler and more idiomatic than ad-hoc inference. Examples that should route there:

- "I have an object whose values depend on its keys" → mapped type
- "the return type changes based on the argument" → conditional type / overloads
- "I want to extract the parameter types of this callback" → utility types like `Parameters<T>`
- "this generic function needs to constrain `T` to objects with a `.id` field" → constraint with `extends`

If the project isn't TypeScript, this step is a no-op — say so and move on. If `tsc` isn't configured (e.g. JS-only with JSDoc), still scan the diff for any TS files added and surface concerns; otherwise skip.

### Type-quality rules (consistent with `typescript`)

- No `any`. Use `unknown` when the type is genuinely unknown and narrow.
- Avoid `as` casts unless asserting at a boundary the type system can't see (e.g. `JSON.parse`).
- Prefer narrow input/output types over wide unions you have to discriminate later.

## Step 3 — Verify with the test suite

Same package-manager rule as Step 2 — lockfile wins; otherwise prefer `bun > pnpm > yarn > npm`.

```bash
bun test                  # bun.lockb / bun.lock  (or: bun run test if the project defines a test script)
pnpm test                 # pnpm-lock.yaml
yarn test                 # yarn.lock
npm test                  # package-lock.json
```

Run the project's full test command. Read the output, don't just glance at the exit code.

- **All green** → done. Move to Step 4.
- **Failures caused by your refactor** → fix them. The fix is usually a missed callsite update or a broken assertion that was relying on the old shape. If the fix would require changing test *expectations* (not just call shape), stop and surface it — that's a behaviour change disguised as a refactor and the user has to make the call.
- **Pre-existing failures unrelated to your diff** → don't touch them. Surface them in the final report so the user knows the suite was already red, but don't fix them in this skill. Mixing unrelated fixes into a refactor PR is the thing this skill is meant to prevent.

If the test command isn't obvious, look at `package.json` `scripts.test` (or `Makefile`, `justfile`, `pyproject.toml`, etc.). Don't guess — running the wrong command and getting a passing exit code is worse than asking.

## Step 4 — Report

End with a short summary the user can scan in 10 seconds:

```
Refactored: <N> files, <M> hunks
KISS/DRY:
  - <one-line description of each meaningful structural change>
Comments: <N removed | n/a>
  - <one line per non-obvious removal, e.g. "stripped tombstone in foo.ts", "dropped redundant JSDoc on getUser">
TypeScript:
  - <errors fixed | warnings fixed | n/a>
Tests: <pass | fail with details>

Out of scope (not changed):
  - <anything you noticed that wasn't in the diff but might be worth a follow-up>
```

Keep the body terse. The user has the diff in front of them; they don't need a paraphrase. The point of the report is to flag anything that wasn't pure mechanics — a deviation from the rules and why, a behaviour change that fell out of the refactor, a pre-existing failing test.

## When NOT to use this skill

- The user is asking for a **review only**, not edits — that's `reviewing-code` or `pr-review-opinionated-thayto`.
- The user wants to refactor an entire codebase or a file outside the current diff — out of scope, this skill is branch-diff-bounded by design.
- The user wants to **add** a feature. This skill simplifies what's already there; it doesn't grow scope.
- There's no diff vs. base (clean tree, branch == main). Tell the user there's nothing to simplify rather than inventing work.

## Worked example

User: *"again, review our code implementations then let's improve our changes that we did, look for duplicated code, and learn more if is possible to improve and refactor using KISS and DRY principles. avoid classes, nested ifs, nested ternaries, prefer small functions. then fix the typescript errors and warnings. at the end, check tests. don't create a new branch."*

Flow:

1. `git diff $(git merge-base HEAD main)...HEAD` — confirm 4 files changed in `src/checkout/`.
2. Spot a class `OrderTotaliser` with two methods, both pure, used once each → flatten to two functions in `total.ts`. Spot a 3-deep ternary in `format-line.ts` → lift to a small `pickLabel(line)` helper. Spot the same `if (!user) return null` guard in three components → push into the `useUser()` hook.
3. Comment sweep over the changed lines: drop `// calculate total` above the now one-line `sum(lineItems)` call (paraphrase), drop the `// fixes CHK-481` next to a guard clause (ticket reference belongs in the PR description), drop the redundant `/** Returns the formatted label. */` above `pickLabel()`. Keep the comment in `tax.ts` that explains *why* the rounding step uses banker's rounding — that's load-bearing context a reader can't recover.
4. `pnpm tsc --noEmit` flags two errors caused by the function split (callsites not updated) and one pre-existing warning about an unused import in a touched file. Fix all three. The unused import surfaced because the refactor removed its only use — that's in scope.
5. `pnpm test` — 1 failing test relied on `OrderTotaliser` being a class instance. Update the import to call the new function. All green.
6. Report:
   ```
   Refactored: 4 files, 7 hunks
   KISS/DRY:
     - flattened OrderTotaliser class → 2 pure functions in total.ts
     - lifted nested ternary in format-line.ts to pickLabel()
     - moved repeated !user guard into useUser() hook
   Comments: 3 removed
     - dropped paraphrase comment above sum() in total.ts
     - dropped CHK-481 ticket reference in checkout.tsx
     - dropped redundant JSDoc on pickLabel() in format-line.ts
   TypeScript: 2 errors fixed (callsite updates), 1 unused-import warning removed
   Tests: pass (1 test updated to use the new function import)

   Out of scope:
     - format-line.ts still imports a deprecated date util — worth a follow-up.
   ```

That's the shape. No branch, no commit, the user reads the diff and ships it.
