---
name: ticket-writing
description: "Write a product ticket acting as a PM. Use when asked to write a ticket, issue, or task for a project management tool like Jira or Linear."
argument-hint: "Describe the bug or enhancement in your own words — the more context you provide, the better the ticket."
---

# Ticket Writing Guidelines

---

## Core Principle: PM Role, Not Engineer Role

When writing tickets, act as a **product manager**, not an engineer. Your job is to clearly describe *what* is broken and *what the expected behavior is*. The engineer picking up the ticket is responsible for diagnosing the root cause, researching the codebase, and determining the fix.

**Do not research the cause when writing tickets.** If the cause is not already known from the conversation, surface it as something the engineer should investigate — do not go looking for it yourself. If relevant context has been provided (e.g., a known file, a known failing condition, a known comparison), include it. Do not go beyond what is already in hand.

---

## Output Format

Default ticketing system: the user uses Jira unless they explicitly say otherwise.

Always output the final ticket inside a single fenced code block (using ```` ``` ````), with no text outside the fence except a brief one-line intro if needed. This lets the ticket be copied as plain text/Markdown source directly into Jira, Linear, GitHub Issues, etc., without the platform rendering it as live headings/checkboxes first.

**The ticket must start with a title**, formatted as an H1 (`# Title`) on the first line inside the fence. Write a short, specific title summarizing the issue (this is what shows up in the ticket list view) — do not leave it out or bury it in the Problem/Summary section.

**Every section heading below the title must be H3 (`###`).** This keeps section headings visually subordinate to the ticket title when pasted into Jira/Linear/GitHub. Do not promote them to H2.

---

## Required Sections

Every ticket must include all of the following sections. Omit a section only if it genuinely does not apply, and note why.

### Problem / Summary

One to three sentences. State what is broken or missing and the impact. Plain language — non-engineers should understand this.

### Background / Context

- What caused the problem, or where the need comes from
- Links to prior work, related tickets, or prior implementations
- Any decisions already made that constrain the solution

### Implementation Notes (Optional when a suggested route for a fix is known)
>**We do not want to be the doctor when writing tickets, the engineer is the doctor and should diagnose the problem. Our job is to properly guide what the request and/or issue is.**

Specific files, functions, and the possible change needed. For each location:
- File path or GitHub permalink to the line(s) in question, if known.
- Current behavior / current code (quoted inline, not in a nested code block), if known
- Proposed change

Use GitHub permalinks anchored to the relevant lines:
(Link will change depending on the project repo we are working on)
  https://github.com/{account}/{repo}/blob/{tag}/{path/to/file.php}#L{line}
  https://github.com/{account}/{repo}/blob/{tag}/{path/to/file.php}#L{startLine}-L{endLine}

**Determining the correct tag ref for permalinks:**

Tags are always preferred over branch refs — feature branches are deleted after merge, making those links dead. Follow this workflow before constructing any permalink:

**If currently on `main`:**
1. Check for uncommitted changes with `git status`
2. If the working tree is clean, run `git pull origin main` to ensure main is current
3. If there are uncommitted changes, do not pull — note in the ticket that the tag ref should be verified before the engineer picks this up
4. Run `git fetch --tags && git describe --tags --abbrev=0` to get the latest tag (e.g. `1.2.5`)
5. Use that tag as the ref in all permalinks

**If currently on a feature branch:**
1. Run `git status` to check for uncommitted changes before doing anything
2. If there are uncommitted changes, ask the user before proceeding:
   > "You have uncommitted changes on this branch. Would you like to stash them so I can check out main for the tag ref, or should I skip that and use the feature branch ref instead?"
   - If stash: `git stash && git checkout main && git pull origin main && git fetch --tags && git describe --tags --abbrev=0`, then `git checkout - && git stash pop` to return
   - If skip: construct the permalink using the current feature branch ref and add a note in the ticket:
     > ⚠️ Permalink uses feature branch ref — line numbers and ref should be verified against the latest tag after merge.
3. If the working tree is clean, ask the user before switching:
   > "To get the most accurate tag ref for permalinks, I'd like to check out main and pull. Is that okay? I'll return to your branch after."
   - If approved: `git checkout main && git pull origin main && git fetch --tags && git describe --tags --abbrev=0`, then `git checkout -` to return
   - If declined: construct the permalink using the current feature branch ref and add a note in the ticket:
     > ⚠️ Permalink uses feature branch ref — line numbers and ref should be verified against the latest tag after merge.

**If the referenced code only exists on the current feature branch and has not been merged to main:**
Do not construct a permalink using the tag ref — those lines won't exist there. Instead, note the file path and approximate location and flag it:
   > ⚠️ This code exists only on the feature branch. Engineer should locate the relevant lines after merge and update the permalink.

### Acceptance Criteria

A checklist of specific, testable conditions that define "done". Each item must be independently verifiable.

Format — one checkbox per condition:
  - [ ] When [condition], [expected result]
  - [ ] [Unchanged behavior] is not affected (regression check)

Guidelines:
- Cover each code location changed
- Include at least one regression check for adjacent behavior that must stay the same
- Call out any paths that are explicitly out of scope (e.g. "most-read feed is unchanged")
- If a change involves a fallback (e.g. ES_Query falling back to WP_Query), add a criterion for the fallback path

### References

Links to related tickets, PRs, documentation, or prior implementations that informed this work.

---

## GitHub Permalink Format

To link to a specific line or range in the repo:

  Single line:  ...blob/{tag}/{file}#L{line}
  Line range:   ...blob/{tag}/{file}#L{startLine}-L{endLine}

Always use a tag ref, not a branch name or commit SHA — branch refs become dead links after merge and commit SHAs are hard to read. See the tag workflow in Implementation Notes above for how to determine the correct ref before constructing a permalink.

---

## Acceptance Criteria Writing Tips

- **Start from the change, not the implementation.** Write what a tester would observe, not what the code does.
- **One condition per checkbox.** If a single item requires "and", split it into two items.
- **Include negative/regression cases.** At least one item per ticket should confirm that something adjacent was not broken.
- **Flag uncertainty explicitly.** If a criterion depends on third-party behavior, note that verification is needed.
- **Avoid vague language.** "Works correctly" is not a criterion. "Returns results ordered by ID descending when two posts share the same timestamp" is.

*Last Updated: 2026-07-29*
