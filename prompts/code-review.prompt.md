---
name: code-review
description: "Run a comprehensive code review on uncommitted work or specific commits. Use when asked to review code, check quality, security, maintainability, or functionality."
argument-hint: "Ticket number and small description of the work being reviewed (e.g., PROD-12345 - Mock Setup Feature Request [uncommitted | last N commits | startHash..endHash]). Add context as needed."
---

# Code Review Guidelines

## Requesting a Code Review

Code reviews should be comprehensive assessments of code quality, security, maintainability, and functionality. The agent needs clear context about what code to review.

**CRITICAL: Always specify whether the code is uncommitted or spans specific commits.**
**CRITICAL: Always review actual git commits/diffs to verify what code changes are included. Do not assume or infer changes—only document what is actually in the commits.**
**CRITICAL: Always verify the correct branch is checked out before reviewing.**
- Run `git branch --show-current` to confirm the active branch
- If the branch does not match the ticket, inform the user:
  > "Please check out the branch for [TICKET-NUMBER] before proceeding. You can do this with: `git checkout <branch-name>`"
- Do not switch branches on behalf of the user.
- Uncommitted work should be reviewed in the context of the current branch.
- **CRITICAL: If the review scope is not specified (uncommitted, last N commits, or commit range), ask the user before proceeding:**
> "Should I review uncommitted changes, or specific commits? If commits, how many or what range?"

---

## Input Requirements

**CRITICAL: If the ticket number is unknown**
- check the branch name for a ticket number (e.g., `feature/PROD-12345-new-feature` or `bugfix/PROD-12345-fix-issue`)
- If no ticket number is found, ask the user for the ticket number:
> "Please provide the ticket number for this review (e.g., PROD-12345) before I begin."

If the user provides extra context about what they built, use it to inform the review.
If no extra context is given, infer scope from the diff/commits alone.

## How to Request a Code Review

### Option 1: Uncommitted Work (Most Common)

Use this when you have local changes that haven't been committed yet:

```
Review all uncommitted work for [TICKET-NUMBER] - [Ticket Name]
Focus on: best practices, security, maintainability, and functionality
```

**Example:**
```
Review all uncommitted work for PROD-12345 - Mock Setup Feature Request
Focus on: best practices, security, maintainability, and functionality
```

### Option 2: Specific Commits

Use this when the work spans multiple commits:

```
Review the last [X] commits for [TICKET-NUMBER] - [Ticket Name]
Focus on: best practices, security, maintainability, and functionality
```

**Example:**
```
Review the last 3 commits for PROD-12345 - Mock Setup Feature Request
Focus on: best practices, security, maintainability, and functionality
```

### Option 3: Commit Range

Use this when you know the exact commit range:

```
Review commits [START_HASH]..[END_HASH] for [TICKET-NUMBER] - [Ticket Name]
Focus on: best practices, security, maintainability, and functionality
```

**Example:**
```
Review commits abc123..def456 for PROD-12345 - Mock Setup Feature Request
Focus on: best practices, security, maintainability, and functionality
```

## Code Review Process (Agent Workflow)

When performing a code review, the agent should:

### 1. Identify the Scope

- Use `git status` to see uncommitted changes
- Use `git log --oneline -X` to see recent commits
- Use `git diff` to view uncommitted changes
- Use `git show --stat <commit>` to see files changed in specific commits

**Note:** If reviewing a teammate's branch, commits may include upstream merges from main 
that are unrelated to the ticket. Focus only on commits authored for this ticket — 
use `git log --oneline --author` or inspect commit messages to filter by ticket number.

### 2. Review All Changed Files

- Read complete changed files (not just diffs when possible)
- Understand context and integration with existing code
- Check for patterns, consistency, and architecture decisions

### 3. Assessment Categories

Evaluate code across four dimensions:

#### Best Practices (Code Quality)

- Code organization and structure
- TypeScript/JavaScript patterns and idioms
- Error handling approach
- Documentation (JSDoc, comments)
- Modularity and reusability
- Naming conventions
- Code duplication

#### Security

- Input validation and sanitization
- Authentication and authorization
- Session management
- Rate limiting
- CORS and CSP configuration
- Sensitive data handling
- Environment variable usage
- Dependency vulnerabilities

#### Maintainability

- Code readability
- Documentation quality
- Test coverage
- Logging and monitoring
- Configuration management
- Upgrade path for dependencies
- Debugging capabilities

#### Functionality

- Feature completeness
- Edge case handling
- Error recovery
- Performance considerations
- Scalability approach
- Resource cleanup
- Integration with existing systems

#### Database Query Safety (Critical for Production Systems)

**CRITICAL:** Always verify that database queries are properly bounded to prevent performance issues in production environments with large datasets.

**Required Checks:**

1. **Query Result Limits**
   - ✅ All queries must specify a result limit (e.g., `posts_per_page`, `LIMIT`, max page caps)
   - ✅ Pagination must have maximum page boundaries
   - ❌ Never allow unbounded queries that could return millions of rows

2. **Date Range Constraints**
   - When querying time-series data, use date bounds (e.g., last 30 days, published after X)
   - Avoid open-ended date ranges on large tables

3. **Index-Friendly Queries**
   - Ensure queries use indexed columns (post_status, post_type, dates, term relationships)
   - Avoid queries that require full table scans

4. **Common Query Patterns to Verify**

   **WordPress WP_Query:**
   ```php
   // ✅ GOOD - Bounded query
   new \WP_Query([
     'posts_per_page' => 20,  // Always set a limit
     'paged' => $page,         // Bounded pagination
     'post_status' => 'publish',
     'date_query' => [         // Optional: date bounds
       'after' => '30 days ago'
     ],
   ]);

   // ❌ BAD - Unbounded query
   new \WP_Query([
     'posts_per_page' => -1,  // Returns ALL posts - dangerous!
     'post_status' => 'publish',
   ]);
   ```

   **Direct Database Queries:**
   ```php
   // ✅ GOOD - With LIMIT
   DB::table('posts')
     ->where('post_status', 'publish')
     ->limit(100)
     ->get();

   // ❌ BAD - No limit
   DB::table('posts')
     ->where('post_status', 'publish')
     ->get();  // Could return millions of rows
   ```

   **Elasticsearch Queries:**
   ```php
   // ✅ GOOD - Bounded size
   $args = [
     'size' => 20,
     'from' => $offset,
   ];

   // ❌ BAD - Unbounded or excessive size
   $args = [
     'size' => 10000,  // Too large
   ];
   ```

5. **Maximum Page Caps**
   - Implement maximum page boundaries for paginated endpoints
   - Example: Cap at page 50 or 100 to prevent deep pagination performance issues

6. **Batch Processing Safety**
   - When processing multiple IDs, validate input count and enforce limits
   - Example: `if (count($post_ids) > 100) { throw new Exception('Too many IDs'); }`

7. **Test with Production-Scale Data**
   - Consider query performance with millions of records
   - Verify indexes exist for query conditions
   - Use `EXPLAIN` to validate query plans

**Red Flags:**
- ⚠️ `posts_per_page => -1` (returns all posts)
- ⚠️ Queries without `LIMIT` clauses
- ⚠️ Unbounded pagination (no max page check)
- ⚠️ Queries spanning entire table without date constraints
- ⚠️ Missing `post_status` filters on post queries
- ⚠️ Batch operations without size limits

### 4. Provide Structured Feedback

Format feedback clearly with ratings and actionable recommendations:

```markdown
## Code Review: [TICKET-NUMBER] - [Ticket Name]

### ✅ Best Practices - [RATING]
- [Assessment summary]
- [Key strengths]
- [Areas for improvement if any]

### ✅ Security - [RATING]
**Implemented Protections:**
- ✅ [Security measure 1]
- ✅ [Security measure 2]

**⚠️ Recommendations:**
- [Specific security enhancement with code example]

### ✅ Maintainability - [RATING]
- [Assessment summary]

**💡 Enhancement Suggestions:**
1. [Optional improvement 1]
2. [Optional improvement 2]

### ✅ Functionality - [RATING]
- [Feature assessment]
- [Completeness check]

### 📋 Dependencies Added/Updated
- [List of new or updated dependencies]

### 🎯 Overall Assessment
- [Production readiness]
- [Key strengths]
- [Critical issues if any]
- **Rating: X/10** - [Overall verdict]
```

#### Rating Scale

- **EXCELLENT** - Exceeds standards, exemplary code
- **STRONG/GOOD** - Meets all standards, minor improvements optional
- **FAIR** - Meets basic standards, some improvements recommended
- **NEEDS WORK** - Below standards, improvements required

#### Overall Rating

- **9-10/10** - Production-ready, excellent quality
- **7-8/10** - Production-ready with minor improvements
- **5-6/10** - Needs improvements before production
- **Below 5/10** - Significant issues, major rework needed

### 5. Document Findings

- Create clear, actionable recommendations
- Include code examples for suggested improvements
- Reference specific files and line numbers using markdown links
- Highlight critical issues that block production deployment
- Note best practices that were exemplary

## Request Format Best Practices
Ticket numbers are always in the format `PROD-12345` (case-insensitive on input, always rendered as `PROD-12345` in output).

### Examples of valid inputs:
- PROD-12345
- Prod-12345
- prod-12345

✅ **Good requests include:**
- Specific scope (uncommitted, last N commits, or commit range)
- Ticket number and descriptive name
- Focus areas (or default to all four dimensions)

❌ **Avoid vague requests:**
- "Can you review my code?" - No scope specified
- "Review this" - No context provided
- Asking to review without git information

## Common Review Scenarios

### Scenario 1: Pre-Commit Review

- Developer has uncommitted work
- Wants to verify quality before committing
- Agent reviews using `git diff` and `git status`

### Scenario 2: Pre-Pull Request Review

- Developer has multiple commits on feature branch
- Wants comprehensive review before creating PR
- Agent reviews using `git log` and `git show` for each commit

### Scenario 3: Security-Focused Review

- Developer implemented authentication or authorization
- Needs security-specific assessment
- Agent reviews with emphasis on security dimension

### Scenario 4: Post-Implementation Validation

- Feature is complete and tested locally
- Final quality check before PR
- Agent performs comprehensive four-dimension review

## After the Review

Once the review is complete:

1. **Address Critical Issues**: Ask if we should fix any blocking security or functionality problems
2. **Consider Recommendations**: Evaluate optional improvements for effort vs. benefit
3. **Document Decisions**: If declining recommendations, document why in commit message or PR
4. **Request Re-review**: If significant changes made, request another review
5. **Proceed with Confidence**: Once approved, commit and create PR

---

*Last Updated: 2026-04-30*
