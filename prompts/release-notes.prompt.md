---
name: release-notes
description: "Format GitHub-generated release notes into the standard release format with Jira/atlassian ticket links. Use when asked to format or write release notes for a new version or tag. Expects a subdomain to be provided for the Jira ticketing base URL."
argument-hint: "The argument after the command name is the Jira subdomain (or a full Atlassian URL from which only the subdomain will be extracted)."
---

# Release Notes Guidelines

## Purpose

Release notes document the changes included in a new release/tag build. They provide clear communication to stakeholders about enhancements, bug fixes, and other updates included in the deployment.

## Input Requirements

- **Jira Subdomain**: The base subdomain for the Jira instance (e.g., `myJiraSubdomain` for `https://myJiraSubdomain.atlassian.net`). This is required to generate clickable links to Jira tickets in the release notes.
- **GitHub Release Notes**: The user must paste the GitHub-generated release notes into this prompt.
- **Note:** User must provide this subdomain when prompted, may provide a full Jira URL as well, but only the subdomain will be used in the output.
- In the event the user does not provide a subdomain, the prompt will ask for it before proceeding.

## How the user should interact with this prompt: Generate Release Notes

1. Go to GitHub repository
2. Navigate to the "Releases" section
3. Select "Generate release notes" button
4. GitHub will automatically generate a changelog with all merged PRs since the last release
5. Copy the generated output and format it according to the guidelines below
6. Paste the GitHub-generated release notes into this prompt and provide the Jira ticketing base URL (e.g., `https://{subdomain}.atlassian.net/browse/PROD-`)

--- 

## Formatting Release Notes

### Structure Overview

Release notes should follow this structure:

```
## Enhancement
[[ Prod - 12345 ]](https://{subdomain}.atlassian.net/browse/PROD-12345) -- [Description] by @[engineer] in https://github.com/[path]
[[ Prod - 12346 ]](https://{subdomain}.atlassian.net/browse/PROD-12346) -- [Description] by @[engineer] in https://github.com/[path]

## Bug Fix
[[ Prod - 12347 ]](https://{subdomain}.atlassian.net/browse/PROD-12347) -- [Description] by @[engineer] in https://github.com/[path]

## Dev / Build Tooling
[[ Prod - 12348 ]](https://{subdomain}.atlassian.net/browse/PROD-12348) -- [Description] (automated) in https://github.com/[path]

**Full Changelog**: [URL provided by GitHub]
```

### Step-by-Step Formatting Process

#### Step 1: Replace Heading

- **Original:** `## What's Changed`
- **Replace with:** `## Enhancement`

#### Step 2: Format Ticket Numbers

Each item should begin with a formatted ticket number. GitHub generates items in these formats:

- `PROD-12419 WordPress: Fix...`
- `[PROD-12419] - WordPress: Fix...`
- `Prod 12330 Add a new feature...`

**Format all variations to:** `[ Prod - 12345 ]`

Examples:
- `PROD-12419` → `[ Prod - 12419 ]`
- `[PROD-12419]` → `[ Prod - 12419 ]`
- `Prod 12330` → `[ Prod - 12330 ]`

#### Step 3: Add Ticket URL

Each formatted ticket number should be a clickable link to the Jira ticket.

**URL Format:** `https://{subdomain}.atlassian.net/browse/PROD-[TICKET_ID]`

**Example:**
```
[[ Prod - 12419 ]](https://{subdomain}.atlassian.net/browse/PROD-12419)
```

#### Step 4: Add Separator Between Ticket and Description

Use ` -- ` (space, two dashes, space) to separate the ticket number from the work description.

#### Step 5: Keep GitHub PR Information with Engineer Attribution

Keep the engineer name and GitHub PR link at the end of each item.

**Original GitHub format:**
```
* PROD-12419 WordPress: Fix live card gallery image IDs... by @[engineer] in https://github.com/[path]
```

**Formatted for release notes:**
```
* [[ Prod - 12419 ]](https://{subdomain}.atlassian.net/browse/PROD-12419) -- WordPress: Fix live card gallery... by @[engineer] in https://github.com/[path]
```

#### Step 6: Remove Bullet Points

- **Remove:** `* ` from the beginning of each line
- Items should be displayed as plain text lines

#### Step 7: Add Bug Fix and Dev / Build Tooling Sections

Before the **Full Changelog** URL, add the following sections:

```
## Bug Fix

## Dev / Build Tooling
```

#### Attempt to place items into the appropriate sections
Attempt to place items into the appropriate sections based on their description keywords. Place items containing "Fix", "Fixes", or "Patch" in Bug Fix. Place dependency bumps or automated items in Dev / Build Tooling. If unsure, leave them in the Enhancement section for manual review.

**Dev / Build Tooling** is for dependency upgrades, automated bumps (e.g. Dependabot), and removal of dev tooling — changes that were not authored directly by an engineer. For these items, replace the `by @[engineer]` attribution with `(automated)` since no engineer directly authored the change.

**Example:**
```
[[ Prod - 12567 ]](https://{subdomain}.atlassian.net/browse/PROD-12567) -- Bump handlebars from 4.7.8 to 4.7.9 in /wp-content/themes/st_refresh (automated) in https://github.com/[path]
```

#### Step 8: Preserve Full Changelog URL

Keep the **Full Changelog** URL unchanged at the end of the release notes. Do not modify the compare link.

**Example:**
```
**Full Changelog**: https://github.com/[path]/compare/[1.0...1.1]
```

### Complete Example

**Original GitHub output:**
```
## What's Changed
* PROD-12419 WordPress: Fix live card gallery image... by @[engineer] in https://github.com/[path]
* PROD-12420 WordPress: Fix video thumbnail... by @[engineer] in https://github.com/[path]
* [PROD-12421] - WordPress: Add a new feature... by @[engineer] in https://github.com/[path]

**Full Changelog**: https://github.com/{account}/{repo}/compare/1.0.0...1.1.0
```

**Formatted release notes:**
```
## Enhancement
[[ Prod - 12421 ]](https://{subdomain}.atlassian.net/browse/PROD-12421) -- WordPress: Add new feature... by @[engineer] in https://github.com/[path]

## Bug Fix
[[ Prod - 12419 ]](https://{subdomain}.atlassian.net/browse/PROD-12419) -- WordPress: Fix live card gallery... by @[engineer] in https://github.com/[path]
[[ Prod - 12420 ]](https://{subdomain}.atlassian.net/browse/PROD-12420) -- WordPress: Fix video thumbnail... by @[engineer] in https://github.com/[path]

## Dev / Build Tooling
[[ Prod - 12422 ]](https://{subdomain}.atlassian.net/browse/PROD-12422) -- Bump some-package from 1.0.0 to 1.0.1 (automated) in https://github.com/[path]

**Full Changelog**: https://github.com/{account}/{repo}/compare/1.0.0...1.1.0
```

### Output Delivery

- Present the formatted release notes in a **single fenced code block** so they can be copied directly
- **Do not** append or save the release notes to this prompt file.

---

## Common Variations to Handle

| Variation | Format to Use |
|-----------|---------------|
| `PROD-12419` | `[ Prod - 12419 ]` |
| `[PROD-12419]` | `[ Prod - 12419 ]` |
| `Prod 12330` | `[ Prod - 12330 ]` |
| `prod 12330` (lowercase) | `[ Prod - 12330 ]` |

## Notes

- Ticket IDs in the Jira URL are always uppercase: `PROD-12345`
- The Jira instance is always: `https://{subdomain}.atlassian.net`
- Reordering bug fixes is done manually after initial formatting
- The Full Changelog URL should never be modified
- Each item should be on its own line
