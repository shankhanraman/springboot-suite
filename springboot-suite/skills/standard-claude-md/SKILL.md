---
name: standard-claude-md
description: >
  Scaffold or refresh the team-standard CLAUDE.md for a Spring Boot backend repo.
  Use when a project has no CLAUDE.md, when the user asks to "init CLAUDE.md",
  "set up project memory", "add the standard CLAUDE.md", "bootstrap team conventions",
  or right after scaffolding a new Spring Boot service. Ships a curated template
  (project pins, skill routing, spec-driven + Superpowers workflow, house conventions,
  hook notes, validation gate) and merges it into the target repo rather than blindly
  overwriting an existing file.
metadata:
  version: 1.0.0
  category: project-setup
  tags:
    - claude-md
    - project-memory
    - spring-boot
    - conventions
    - scaffolding
---

# Standard CLAUDE.md

This skill installs the team-standard `CLAUDE.md` — the living, git-checked project memory
file for a Spring Boot backend — into the current repository.

The canonical template lives at `assets/CLAUDE.md.template` in this skill. Treat it as the
source of truth for structure and wording; adapt only the marked `<!-- ← EDIT -->` spots to
the actual project.

## When to use

- A Spring Boot repo has **no** `CLAUDE.md` yet.
- The user just scaffolded a new service and wants project memory set up.
- The user explicitly asks to init / refresh / standardize the `CLAUDE.md`.

## How to apply

1. **Check for an existing `CLAUDE.md` at the repo root.**
   - **None present** → copy `assets/CLAUDE.md.template` to `./CLAUDE.md` verbatim, then do step 2.
   - **Already present** → do **not** overwrite. Read it, then merge: add any missing standard
     sections (Project, Skill, Workflow, Working principles, House conventions, Enforced by hooks,
     Validation gate, Run locally, Learnings) while preserving the team's existing custom lines
     and any accumulated `## Learnings`. Show the diff before writing.

2. **Fill the `<!-- ← EDIT -->` placeholders** using what you can verify from the repo — do not ask
   the user for things already pinned by the scaffold (Java version, build tool, DB). Specifically:
   - The one-line service description under **Project**.
   - The Java / Spring Boot / build / DB line — read `build.gradle`/`pom.xml` and `flyway`/JPA config
     rather than guessing; only adjust if the repo differs from the template defaults.
   - The base package in **House conventions** (`com.arogya.cafe.*`) — replace with the repo's actual
     root package (read it from the main `@SpringBootApplication` class).

3. **Keep it under ~100 lines.** Don't inline skill references or expand sections; the file is an
   index and a set of rules, not documentation.

4. **Don't duplicate hooks as advisory rules.** The "Enforced by hooks" section only *names* what the
   hooks do; the actual enforcement is in `.claude/settings.json` / `hooks/` (see the
   `springboot-suite` plugin). If those hooks aren't installed yet, tell the user, don't silently
   convert them into prose rules.

## Guardrails

- Never overwrite an existing `CLAUDE.md` without showing the merge first.
- Never invent project facts — if the base package or DB can't be determined from the repo, leave the
  template's placeholder and flag it for the user to edit.
- Preserve the `## Learnings` section's existing lines on any refresh — that history is the point of
  the file.
