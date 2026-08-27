---
name: my-skill
description: >
  Use this skill when [specific user scenario], including when the user
  doesn't name the tech explicitly (e.g. "[casual example prompt]") — check
  [package.json dependency / config file] before assuming it applies.
  Targets [version/stack] specifically; verify the installed version before
  applying version-pinned guidance. NOT for [near-miss adjacent tech] — see
  the [other-skill-name] skill instead.
argument-hint: "[optional placeholder hint shown in chat when invoked as /my-skill]"
user-invocable: optional true/false            # defaults to true; false = hidden from / menu but still auto-loaded
disable-model-invocation: optional true/false  # default: false; true = manual /slash only, never auto-loaded
---

<!--
  DELETE THIS COMMENT BLOCK once the description above is filled in for real.
  Description-writing checklist — full methodology at
  https://agentskills.io/skill-creation/optimizing-descriptions:
  - name+description are the ONLY signal used to decide whether to
    auto-load the skill (they're preloaded for every skill; the body is
    only read once triggered) — keep the real description under 1024 chars.
  - Use imperative phrasing: "Use this skill when X," not a topic label.
  - Cover the FULL scope of the body, not just the headline topic — a topic
    buried in the body but missing from the description will never trigger.
  - Name the specific version/stack targeted, and tell the agent to verify
    the installed version/dependency before applying version-pinned facts
    — don't assume every project in a multi-project workspace matches
    (e.g. "MyDashboard" vs "MyDashboard-prototype" can be different stacks).
  - Add an explicit "NOT for X" clause for near-miss adjacent tech — shared
    keywords are the most common source of false triggers, not unrelated
    topics.
  - Add an ambient-trigger clause for unnamed-but-implied usage, gated on
    an actual project signal (package.json dependency, config file present)
    — not on vibes.
  - Cross-reference other skills by their EXACT `name:` field, not a guess.
  - If "why this skill, when to use it, and what NOT to confuse it with"
    doesn't fit in 1024 chars, the scope is probably too broad — split it.
  - Ask first: is this genuinely non-obvious/version-pinned knowledge the
    model would otherwise get wrong (worth auto-loading), or common
    knowledge it already has (candidate for `disable-model-invocation:
    true`, or no skill at all)?
-->

# Skill Title

> State the version(s)/stack this skill targets, and how to verify a given
> project is actually on that version/stack before applying version-pinned
> guidance.

## Core Principles
- Key rule or constraint

## Patterns & Conventions
Describe the approach, with code examples where helpful.

## Common Pitfalls
What to avoid and why.

## Reference Files (optional, for large skills)
If the topic is large, keep this main file to the highest-value summary and
split deep-dive material into `references/*.md`, loaded only when the task
goes deeper than the summary — cross-reference them by relative path:
- **`references/topic-a.md`** — what it covers, when to load it

*Last Updated: YYYY-MM-DD*
