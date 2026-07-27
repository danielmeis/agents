---
name: pull-request
description: "Write a pull request following team standards. Reviews actual git commits and saves the PR as a file. Use when asked to write a PR, pull request, or document code changes for a ticket."
argument-hint: "[ticket-number] — e.g. Prod-12345"
---

# Pull Request Guidelines

---

## ⚙️ Configuration

* **PR Save Path:** `~/Websites/agents/local/pull-requests/`
* **File Naming:** `Prod-NNNNN -- Short Description.md`
* **Example:** `Prod-12345 -- Homepagefeature added.md`

---

## ⚠️ MANDATORY: Two Required Actions — Do Both, Every Time

Writing a PR has two non-negotiable steps. Showing the content in chat is NOT sufficient on its own.

### Step 1 — Verify branch and review actual commits

**CRITICAL: Before reviewing commits, confirm the correct branch is checked out.**
- Run `git branch --show-current` to verify the active branch
- If the branch does not match the ticket, inform the user:
  > "Please check out the branch for [TICKET-NUMBER] before I begin. You can do this with: `git checkout <branch-name>`"
- Do not switch branches on behalf of the user.

```
git log --oneline <branch> ^main
git show --stat <each-commit-hash>
```

**CRITICAL: Always review actual git commits/diffs to verify what code changes are included. Do not assume or infer changes — only document what is actually in the commits.**

### Step 2 — Save the PR as a Markdown file

After writing the content, **always** create the file using the `create_file` tool at the configured save path (see ⚙️ Configuration above).

> The PR is not done until the file exists. Do not ask the user to save it — create it.

---

## Ticket Number Format

Ticket numbers are always in the format `PROD-12345` (case-insensitive on input, always rendered as `PROD-12345` in output). The file is always named with `Prod-NNNNN` casing (see ⚙️ Configuration above).

Examples of valid inputs:
- PROD-12345
- Prod-12345
- prod-12345

**CRITICAL: If no ticket number is provided, check the branch name for one (e.g., `feature/PROD-12345-description`). If none is found, ask the user before proceeding.**

---

## Pull Request (PR) Writing Best Practices

When writing Pull Requests, structure them to be clear for both technical and non-technical stakeholders (executives, product managers, etc.). Use plain language and focus on business value alongside technical details.

## Required Sections

1. **Overview** (# heading)
   - Brief summary of what changed at a high level (1-2 sentences)
   - Focus on the primary update or feature
   - Use plain language that non-developers can understand

2. **Problem** (# heading)
   - Clearly state the issue or limitation being addressed
   - Explain the impact on users or business
   - Describe what wasn't working or what was missing

3. **Product Notes** (# heading - OPTIONAL)
   - Include relevant context about product decisions
   - Document trade-offs or considerations
   - Explain why certain approaches were chosen
   - Note any stakeholder feedback or requirements

4. **Solution** (# heading)
   - Explain the approach taken to solve the problem
   - Highlight user-facing benefits and improvements
   - Keep technical jargon to a minimum or explain terms
   - Use bullet points with proper Markdown syntax (`-`)
   - Each bullet should have a clear label followed by description

5. **Testing & Validation** (# heading)
   - Provide specific instructions on how to test the work
   - Include step-by-step testing scenarios with expected results
   - List any test data, credentials, or setup required
   - Document edge cases that were tested
   - Include manual testing steps for reviewers
   - Reference automated test coverage if applicable
   - Note any environment-specific testing requirements

6. **Technical Improvements** (# heading) (Optional)
   - Optional section for code quality, performance, or architectural improvements
   - Detail refactoring, dependency updates, or migration work
   - Use subheadings (`##` heading) to group related improvements
   - Use bullet points with labels for each improvement
   - Mention test coverage changes, build improvements, or tooling updates
   - **EXCLUDE internal documentation changes** (like copilot-instructions updates) unless they directly impact developers' workflow

## Formatting Requirements

- **USE ACTUAL MARKDOWN SYNTAX** - output `#` for H1, `##` for H2, `-` for bullets
- Do NOT say "Use Markdown headings" - actually output the `#` symbols
- Primary sections use `#` (H1): Overview, Problem, Product Notes, Solution, Testing & Validation, Technical Improvements
- Subsections use `##` (H2): Enhanced User Experience, Code Quality & Testing, etc.
- Bullet points use `-` at the start of each line
- Each bullet should have format: `- Label: Description`
- Keep paragraphs short (2-4 sentences max)
- Bold key terms sparingly using `**term**`
- Include screenshots/videos for UI changes when applicable

## Example Structure

```markdown
# Overview
Brief description of the primary change.

# Problem
What wasn't working and why it mattered.

# Product Notes
(Optional) Context about product decisions or trade-offs.

# Solution
How we addressed the problem:

- Feature A: Description
- Feature B: Description

# Testing & Validation

## Manual Testing Steps
1. Navigate to [specific page/feature]
2. Perform [specific action]
3. Verify [expected result]

## Test Scenarios
* Scenario 1: Description and expected outcome
* Scenario 2: Description and expected outcome
* Edge Case: Description and how it was handled

## Automated Tests
* Unit tests: Added 15 tests covering new validation logic
* Integration tests: 3 new tests for API endpoints
* Coverage: Increased from 78% to 85%

# Technical Improvements
(Optional)

## Code Quality
* Refactoring: Extracted common logic into reusable service

## Performance
* Database: Added indexes on frequently queried columns
```

## Verification Steps Before Writing

1. Run `git log --oneline <branch> ^main` to see all branch commits
2. Run `git show --stat <commit-hash>` for each commit to see changed files
3. Review actual diffs to understand what code changed
4. Only document features/changes that exist in the actual commits
5. Exclude internal documentation updates unless they impact developer workflow
6. Verify technical details are actually in the final code

## Final Delivery — Save the File

**After writing the PR content, create the file immediately using the `create_file` tool at the configured save path (see ⚙️ Configuration above).**

* Do not present the content and wait — create the file as part of the same response.
* Do not ask the user to copy/paste or save it themselves.

## Tone

- Professional but conversational
- Focus on value and outcomes
- Avoid unnecessary technical depth in Overview/Problem/Solution
- Save technical details for Technical Improvements section
- Be factually accurate—don't embellish or assume changes

## PR Summary File Output

For research projects, multi-repository changes, or complex features, the standard PR file may include additional sections.

### When to Add Extra Sections

- Research projects with findings to implement later
- Changes spanning multiple repositories
- Complex features with detailed implementation checklists
- Work that will be merged to production at a later date

### Multi-Repo File Structure

In addition to the standard sections, add repository-specific sectionsand an **Implementation Checklist** after the standard PR sections as needed.:

```markdown
---

# Implementation Checklist

## Repository A
- [ ] Copy specific changes from research branch
- [ ] File: `src/components/Feature.tsx` lines 45-120
- [ ] Build/test steps

## Repository B
- [ ] Copy specific changes from feature branch
- [ ] File: `api/routes/feature.js` lines 23-67
- [ ] Build/test steps

## Post-Deployment
- [ ] Verify feature in staging
- [ ] Run smoke tests
- [ ] Monitor logs for errors
```

---

*Last Updated: 2026-05-01*
