---
name: my-project
description: >
  Load [Project Name] context. Use when working in ~/Websites/[folder] on
  [what the project does] — [its actual framework/runtime/DB, e.g. "Next.js
  16 + PostgreSQL"]. Not to be confused with [similarly-named sibling
  project], which uses [different stack].
argument-hint: "[optional placeholder hint shown in chat when invoked as /my-skill]"
user-invocable: optional true/false            # defaults to true; false = hidden from / menu but still auto-loaded
disable-model-invocation: optional true/false  # default: false; true = manual /slash only, never auto-loaded
---

<!--
  DELETE THIS COMMENT BLOCK once the description above is filled in for real.
  Description-writing checklist, adapted for project (not technology)
  skills — full methodology at
  https://agentskills.io/skill-creation/optimizing-descriptions:
  - name+description are the ONLY signal used to decide whether to
    auto-load the skill (preloaded for every skill; body read once
    triggered) — keep the real description under 1024 chars.
  - The strongest trigger for a project skill is the file path itself:
    state "Also load when the active file/task is under ~/Websites/
    [folder]" rather than relying on the project name being mentioned.
  - If a similarly-named sibling exists (a "-prototype", "-v2", or
    otherwise easily-confused folder), name it and how it differs
    (different stack, different status) — the single biggest source of
    misapplied guidance is two folders that look related but run
    different frameworks/DBs/versions.
  - Don't restate generic tech-skill content here (Next.js/React/Redis/etc.
    best practices already live in their own skills) — this skill is for
    facts unique to THIS project. Cross-reference tech skills by their
    exact `name:` field instead of duplicating their content.
  - Keep tech-stack versions current — a stale version claim here is worse
    than none, since it can contradict what the paired tech skill actually
    verifies against package.json.
  - Most project-context skills in this repo default to
    `disable-model-invocation: true` (session-start style, explicitly
    invoked) since the workspace/active-file already gives a human strong
    context — only enable auto-invocation for ambient loading by
    agents/subagents operating without that context.
-->

# [Project Name]

**Location:** `~/Websites/[folder]`
**Status:** Active / Maintenance / Archived
**Not to be confused with:** [any similarly-named sibling project and how it differs, or delete this line if none]

## Overview
One paragraph: what this project does and who uses it.

## Tech Stack
- **Runtime / Language:**
- **Framework:**
- **Database:**
- **Other tools:**

## Architecture Patterns
Key architectural decisions, folder layout, and design patterns in use.

## Project-Specific Rules
Conventions or constraints unique to this project (naming, file structure, API patterns, etc.).

## Development Workflow
Common commands, local setup, how to run tests.

## Common Tasks
How to add a feature, deploy, run migrations, etc.

---

*For workspace-wide standards, see the global copilot-instructions.*
*For framework/language best practices, see the relevant tech skill (e.g. nextjs, react, typescript) instead of duplicating them here.*
*Last Updated: YYYY-MM-DD*
