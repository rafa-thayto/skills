---
name: write-issue-thayto
description: Use this skill whenever the user wants to scaffold a new issue, ticket, todo, or task file — phrases like "create an issue", "create a ticket for X", "add an issue", "new todo", "open a ticket", "file an issue", "create a linear issue", "linear ticket", "write up a task", or any equivalent. Creates a numbered markdown file under `docs/issues/` with YAML frontmatter (id, status, title) and the standard sections Description / Why / Acceptance / What you must do / Hints. Auto-increments the id by scanning existing files. Conducts a brief consolidated interview only when the user's request is too thin to write a useful ticket — otherwise scaffolds directly. Trigger this skill even when the user says "linear" or talks about a tracking system; the skill writes a local file regardless of which tracker they mention.
---

# write-issue-thayto

Scaffold a new issue file in `docs/issues/` so a future agent (or human) can pick it up and execute on it.

## Why this format exists

Each ticket is a self-contained handoff. Someone — possibly a Claude agent invoked tomorrow — should be able to open the file, understand the goal, know when it's done, and start work. That's why the sections aren't optional decoration: each one answers a specific question a worker will ask.

| Section | Question it answers |
|---|---|
| Description | What is this ticket about? |
| Why | Why are we doing this — what problem or goal does it serve? |
| Acceptance | How will I know I'm done? |
| What you must do | What concrete steps should I take? |
| Hints | What gotchas, files, or commands will save me time? |

If a section can't be filled with substance, the ticket isn't ready yet. That's the whole reason for the interview step below.

## Step 1 — Locate the issues folder

Find the project root (the nearest ancestor containing `.git`, `package.json`, `pyproject.toml`, or similar). Issues live at `<project-root>/docs/issues/`.

If the folder doesn't exist yet, create it. Don't ask permission for this — it's the obvious thing to do and the user expects it.

```bash
mkdir -p docs/issues
```

## Step 2 — Compute the next id and filename

Scan `docs/issues/` for existing tickets and take the highest numeric id + 1. Format as a 3-digit zero-padded number (`001`, `002`, …, `099`, `100`).

```bash
ls docs/issues/ 2>/dev/null | grep -E '^[0-9]{3}-' | sort | tail -1
```

If the directory is empty or doesn't exist yet, the next id is `001`.

**Filename:** `<id>-<slug>.md` where `<slug>` is the title:
- lowercased
- non-alphanumeric runs replaced by single `-`
- leading/trailing `-` stripped
- truncated to ~50 chars at a word boundary if very long

Example: title `"pomodoro status — show how long is left"` → slug `pomodoro-status-show-how-long-is-left` → file `001-pomodoro-status-show-how-long-is-left.md`.

## Step 3 — Decide: interview or scaffold directly?

Read the user's request and ask: **for each required section (Description, Why, Acceptance, What you must do), do I have enough material to write a non-trivial sentence without inventing facts?**

- If yes for all four → skip the interview. Write the file and show it to the user. They can edit afterward.
- If no for any → ask **one consolidated question** that covers only the gaps, then write the file.

Hints is always optional; omit the section entirely if you have nothing real to put there.

The interview is a tool for getting an executable ticket, not a checklist. If the user's intent is obvious (e.g. they paste a clear bug report), don't pester. If their intent is a vague one-liner like "create a ticket for the pomodoro thing", you have to ask.

### How to phrase the interview

Bundle the gaps into one message. Example, when everything is missing:

> Quick interview before I write the ticket:
> 1. **Acceptance** — what observable behavior tells us this is done? (e.g. command output, file appears, test passes)
> 2. **Implementation steps** — what do you want the worker to actually do, in order?
> 3. **Why** — why does this matter / what triggered it?
> 4. **Hints** (optional) — any files, commands, or gotchas worth flagging?

Adapt the questions to which gaps actually exist. If the user already gave you Acceptance, don't ask for it.

## Step 4 — Write the file

Use this exact structure. Keep prose terse — these tickets are read by people in a hurry.

```markdown
---
id: 001
status: todo
title: "<title in sentence case, with em-dash for sub-clauses if useful>"
---

## Description

<1–3 sentences: what this ticket is, the surface it touches, and the shape of the change.>

## Why

<1–3 sentences: the motivation. A bug? A user need? A blocker for another ticket? Be concrete — "users can't see remaining time" beats "improve UX".>

## Acceptance

- <Observable outcome 1 — phrased so a worker can check it off.>
- <Observable outcome 2.>
- <Edge case or failure mode the worker must handle.>

## What you must do

1. <Concrete step. Reference real file/function names if you know them.>
2. <Next step.>
3. <Verification step — usually "run the tests" or "manually check X".>

## Hints

- <Optional. Real, specific guidance — a function to reuse, a flag to set, a mistake easy to make. Omit the whole section if you have nothing real.>
```

### Frontmatter fields

- `id`: zero-padded 3-digit string, matching the filename prefix.
- `status`: one of `todo`, `in-progress`, `done`, `blocked`. Default to `todo` for new tickets.
- `title`: quoted string. Sentence case. Em-dash (`—`) is fine for "X — short clarifier" form, matching the existing style.

### Style guidance for the body

- **Be specific.** "Fix the pomodoro" is useless; "make `pomodoro status` print remaining minutes" is a ticket.
- **Acceptance items are checkable.** Each bullet should be testable by reading output, running a command, or inspecting a file. If you can't say how you'd verify it, it doesn't belong in Acceptance.
- **Numbered steps in "What you must do"** because order usually matters and they're easier to reference ("step 2 failed").
- **Use backticks for filenames, commands, and identifiers** — `pomodoro.py`, `python3 -m unittest`, `POMODORO_NOW`. Makes them grep-able and unambiguous.
- **Keep Hints honest.** Only include if it would genuinely save a worker time. Padding this section makes the format less trustworthy.

## Step 5 — Confirm and report

After writing, tell the user the file path and the assigned id in one short line, e.g.:

> Wrote `docs/issues/001-pomodoro-status-show-how-long-is-left.md` (id 001, status todo).

Don't paste the entire file back — they can open it. If they asked for changes, edit the file rather than rewriting from scratch.

## Worked example

User says: *"create a ticket: implement `pomodoro status` so it reads `.pomodoro` and prints how many minutes are left, defaulting to a 25 min pomodoro. should print 'no pomodoro running' if the file doesn't exist, and 'done' if expired."*

That request has Description, Acceptance, and the start of What-you-must-do. Why and Hints are missing — but Why is implied (the tool needs a status command to be useful), and Hints can be omitted. This is a case where the interview is **not** needed; scaffold directly:

```markdown
---
id: 001
status: todo
title: "pomodoro status — show how long is left"
---

## Description

Implement a `status` subcommand for the pomodoro CLI that reports the remaining time in the current pomodoro by reading the `.pomodoro` state file.

## Why

Right now the tool can start a pomodoro but the user has no way to ask "how much time is left?" without watching the clock. `status` is the smallest change that makes the tool actually useful day-to-day.

## Acceptance

- `pomodoro status` reads the `.pomodoro` file and prints how many minutes/seconds are left in the current pomodoro (default 25 minutes).
- If the file does not exist, prints `no pomodoro running` and exits non-zero.
- If the pomodoro has expired, prints `done`.

## What you must do

1. Implement the `status` command in `pomodoro.py`.
2. Add tests to `test_pomodoro.py` covering the three cases above. Subclass `PomodoroTestBase` and follow the pattern of `TestSeed.test_start_writes_iso_timestamp`.
3. Run `python3 -m unittest test_pomodoro` and make sure all tests pass — the new ones and the existing ones.

## Hints

- Use `pomodoro.now()` for the "current time" — it honours `POMODORO_NOW` so your tests can be deterministic.
```

That's a ticket a fresh agent could pick up and finish without asking questions.

## When NOT to use this skill

- The user wants to **edit** an existing issue (change status, add notes). This skill is for creating new files; for edits, just edit the file directly.
- The user wants to file an issue on GitHub/Linear/Jira. This skill writes a *local* file under `docs/issues/`. If they explicitly want the issue posted to a remote tracker, ask whether to also create a local file or to use a different tool (e.g. `gh issue create`).
- The user wants a one-line TODO inline in code. That's a `# TODO` comment, not a ticket file.
