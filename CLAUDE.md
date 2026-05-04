# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Personal Claude Code skills for `rafa-thayto`. Each top-level directory is one skill, containing at minimum a `SKILL.md` with YAML frontmatter (`name`, `description`) and a Markdown body. Optional `scripts/`, `references/`, `assets/` siblings are loaded on demand by the skill's body.

## Install / dev loop

Skills are not auto-discovered from this repo. Claude Code only loads personal skills from `~/.claude/skills/<name>/`. Symlink, don't copy, so edits here go live in new sessions immediately:

```bash
ln -s "$PWD/<skill-name>" ~/.claude/skills/<skill-name>
```

To pick up changes, just start a new Claude Code session — no reinstall.

## Naming: avoid collisions

The skill `name` in frontmatter must not collide with any installed plugin skill (e.g. `pr-review-toolkit:review-pr`, `code-review:code-review`). Plugins win over personal skills with the same effective name. When the obvious name is already taken upstream, suffix with `-thayto` (or another distinctive token). The directory name and the frontmatter `name` must match.

To list the names already in play:

```bash
ls ~/.claude/skills/                          # other personal skills
# plugin skills appear in the available-skills system reminder when a session starts
```

## Authoring conventions

These are the load-bearing rules for skills here, beyond the usual frontmatter format:

- **Description is the trigger.** It is the only thing Claude sees until the skill fires. List concrete signals (URL shapes, phrasings, file types) and bias toward "pushy" language — Claude under-triggers more often than it over-triggers.
- **Body under ~500 lines.** Split overflow into `references/<topic>.md` and point to it from the body. Don't inline what can be loaded on demand.
- **State the *why* behind hard rules.** Future Claude follows rules better when the reasoning is explicit. Reserve all-caps `MUST`/`NEVER` for absolutes; explain the rest.
- **Model the rules in the prose.** If the skill bans em dashes or effort estimates in output, don't use them in the SKILL.md text either — patterns leak.

## Skills in this repo

| Skill | What it does | Effect on the target repo |
|---|---|---|
| `pr-review-opinionated-thayto` | Reviews a GitHub PR and posts the review through `gh`. Triggered by PR URLs or "review this PR". | Read-only on the target. Posts only via `gh api`, never via local git. |
| `write-issue-thayto` | Scaffolds a numbered issue file under `docs/issues/` with frontmatter (id, status, title) and the sections Description / Why / Acceptance / What you must do / Hints. Triggered by "create an issue", "new ticket", "linear ticket", etc. | Writes one new file at `docs/issues/<NNN>-<slug>.md`. Creates the folder if missing. Does not edit existing tickets. |

When adding a new skill, append a row here so the inventory stays useful as a glance-sheet.

## Working in this repo

- Read-only filesystem rule: do not commit or push without an explicit instruction. The user pushes from their own session.
- Each skill should declare, in this CLAUDE.md row, what it does to the *target* repo (read-only, writes one file, posts via API, etc.). Surprises in scope are the most expensive bug in a skill.
