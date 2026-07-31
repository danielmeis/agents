# Copilot Coding Agent Onboarding Instructions

**Scope:** These instructions apply to all projects in this workspace. See project-specific sections for details. Workspace-wide practices are listed first, followed by details for individual projects.

## Workspace-Wide Practices

- **General Purpose**: This workspace contains multiple web projects, primarily WordPress-based, with mixed technologies (PHP, JS/TS, CSS, Bash, YAML).
- **Global Setup**: Always ensure Node.js, PHP, Composer, and npm are installed and up-to-date for all projects. Follow project-specific setup for each folder.

### Code Style (All Projects)
- **Indentation:** 2 spaces (never tabs)
- **Line length:** 120 characters maximum
- **Variables:** Use `const` over `let`, never use `var`
- **Functions:** Use arrow functions for callbacks and simple functions; use regular functions for class methods and when `this` binding is needed
- **Newlines:** Always insert final newline in files
- **Whitespace:** Trim trailing whitespace, maintain consistent spacing
- **NO TRAILING SPACES**: Never include trailing spaces at the end of lines. This is enforced by ESLint (`no-trailing-spaces`).
- **Blank Lines**: No multiple consecutive blank lines (max 1 blank line between code blocks).
- **Line Endings**: Files must end with exactly one newline character.
- **JSDoc:** Required for public APIs and complex logic
- **Trailing Commas:** Always use trailing (hanging) commas in multi-line arrays, objects, and function parameters/arguments. This is enforced by Prettier and ESLint (`comma-dangle`).
- **Security:** Sanitize inputs, use HTTPS, implement authentication/authorization, handle sensitive data securely, keep dependencies updated.
- **Documentation:** Maintain clear, up-to-date documentation for all projects, including setup instructions, code comments, and API references.
- **CSS/HTML Class Naming:** Use BEM (Block Element Modifier) convention for all new features and components. Prefer double underscore for elements (e.g., block__element) and double hyphen for modifiers (e.g., block__element--modifier).
- **Validation:** Include security checks, comprehensive error messages, handle edge cases, enforce code quality, follow project standards, maintain consistency.
- **Accessibility:** WCAG AA compliance, semantic HTML, keyboard navigation, screen reader support, color contrast, alt text, focus management, ARIA labels.
- **Performance:** Optimize for load times, minimize bundle size, use efficient algorithms, lazy load where appropriate, monitor performance metrics.
- **Commenting:** Use clear, concise comments to explain complex logic, decisions. Add important details when needed. Avoid obvious comments that restate code.

### Docblocks & Comments

The goal is clarity, not completeness. A reader should understand the *why* and any non-obvious behavior — not have the code described back to them in prose.

- **Lead with what the caller needs to know.** The function name already says what it does; the docblock expands on contracts, side effects, or non-obvious behavior. Never restate the signature.
- **Default to brief.** A single summary line is enough when the name and parameters are self-explanatory. Add more only when it earns its place.
- **Scale detail to complexity.** Simple getters and slice helpers need one line. Functions with branching paths, caching behavior, or external side effects justify fuller explanation.
- **Don't duplicate constant or type docblocks.** If a constant documents its value or behavior, use `{@see ConstantName}` in the function docblock — don't copy the prose.
- **Eligibility rules and invariants belong at the source.** Document constraints at the function that enforces them, not at every caller.
- **Inline comments explain decisions, not mechanics.** `// Stale: serve retained snapshot` is useful. `// Check if $building is truthy` is not.

---

### TypeScript Best Practices

> For detailed patterns and examples, load the `/typescript` skill.
- Enable strict mode in tsconfig.json (`"strict": true`)
- Avoid `any` type - use `unknown` when type is truly unknown, then use type guards
- Define interfaces for all public APIs and data structures
- Use type guards for runtime type checking (`typeof`, `instanceof`, custom predicates)
- Prefer `interface` over `type` for object shapes (better for extension)
- Prefer `const` objects with `as const` over enums for fixed sets of values; derive union types with `typeof X[keyof typeof X]`
- Document complex types with JSDoc comments
- Use utility types: `Partial<T>`, `Required<T>`, `Pick<T>`, `Omit<T>`, `Record<K, T>`

### Error Handling
- Use custom error classes for different error types (extends Error)
- Always include context in error messages (userId, operation, resource)
- Log errors with appropriate severity levels (error, warn, info, debug)
- Handle async errors with try/catch or .catch()
- Never swallow errors silently - at minimum log them
- Validate inputs early and fail fast with descriptive messages
- Use error boundaries in React applications
- Return proper HTTP status codes in API responses

### Logging Standards
- Use structured logging (JSON format for production)
- Include context: userId, requestId, operation, resource
- **Levels:**
  - `error`: Requires immediate action, something failed
  - `warn`: Review needed, degraded functionality
  - `info`: Business events, important state changes
  - `debug`: Development only, verbose details
- Never log sensitive data (passwords, tokens, credit cards, PII)
- Log at decision points and error boundaries
- Include timing information for performance monitoring

### API Design Standards
- Use proper HTTP methods: GET (read), POST (create), PUT/PATCH (update), DELETE (remove)
- Use appropriate HTTP status codes:
  - 200 (OK), 201 (Created), 204 (No Content)
  - 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found)
  - 422 (Unprocessable Entity), 429 (Rate Limited)
  - 500 (Internal Server Error), 503 (Service Unavailable)
- Consistent error response format: `{ error: { code, message, details } }`
- Version APIs in URL (`/api/v1/`) or header (`Accept: application/vnd.api.v1+json`)
- Use pagination for list endpoints (limit, offset, or cursor-based)
- Include rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- Support filtering, sorting, and field selection for list endpoints
- Use consistent naming: plural nouns for collections (`/users`), IDs for specific resources (`/users/123`)
- Document APIs with OpenAPI/Swagger specs

### Testing Standards
- Write descriptive test names that explain behavior: `should return 401 when token is expired`
- Group related tests logically using `describe` blocks
- Always include edge cases: null, undefined, empty, boundary values
- Maintain or improve test coverage with changes (aim for >80%)
- **Testing frameworks:** Jest + React Testing Library for React components, PHPUnit for PHP code
- Test behavior, not implementation details
- **CRITICAL: Failing tests are production signals first:** When a test fails, do not only patch the test or loosen assertions. First determine whether the failure exposes a real production bug, race condition, integration problem, stale cache issue, or broken user workflow. Fix the production issue when one exists, then update or strengthen tests to lock in the corrected behavior.
- Use test fixtures and factories for complex data setup
- Mock external dependencies (APIs, databases) in unit tests

### Root Cause Communication
When a bug's cause was unknown at the start of a session, once it's located, state it directly in chat before or alongside implementing the fix — don't let it stay buried in code comments or diffs.
- **What caused it, in plain English:** Explain the mechanism for a human, not just the code — write it so it can be dropped into a code review or handed to QA as-is, not just understood by another engineer reading the diff.
- **How to reproduce it:** Give concrete, actionable repro steps — which conditions trigger it, which endpoint/page/action to exercise, what to look for — so QA can independently confirm the failure exists pre-fix and is resolved post-fix.
- **Scope of impact:** State whether the bug is narrow (one specific record/config) or systemic (any input matching a certain condition), so QA knows how broadly to test.
This does not apply to trivial or already-understood fixes — only when the root cause had to be genuinely diagnosed during the conversation.

### Version Control & AI Agent Workflow
- **Git Commits:** AI/Copilot agents should NEVER create git commits automatically. All work done by AI must be verified and approved by the developer before committing.
- **Code Review:** All AI-generated code changes must be reviewed, tested, and validated before being committed to version control.
- **Developer Approval:** The developer maintains full control over what gets committed and when. AI agents assist with implementation but do not perform version control operations.

---

### Dependency Management
- Review security advisories before adding dependencies (npm audit, Snyk)
- Prefer maintained packages (recent commits, active issues, responsive maintainers)
- **Pin exact versions in package.json — no `^` or `~` prefixes.** Use the exact installed version (e.g. `"axios": "1.16.1"`, not `"axios": "^1.16.1"`). This prevents unexpected upgrades from pulling in malicious or breaking releases.
- Lock dependencies with package-lock.json or yarn.lock
- Audit regularly: `npm audit`, `yarn audit`
- Update dependencies deliberately and one at a time — never allow auto-upgrades
- Document why dependencies were chosen (package.json comments or PR)
- Check bundle size impact of new dependencies

### Multi-Project Workspace
- Changes affecting multiple projects require testing in all affected projects
- Shared code should be extracted to common packages/libraries
- Document cross-project dependencies clearly
- Use consistent versions of shared tools (ESLint, TypeScript, Prettier)
- When fixing issues, check if similar code exists in other projects

### Directory Scanning & Exploration
When exploring or scanning the workspace, **always skip `node_modules/` and `vendor/` directories by default.** These dependency trees contain thousands of files that are not part of the codebase we have built and will waste time, inflate context, and obscure relevant code.

- **Default behavior:** Exclude `node_modules/`, `vendor/`, and any nested copies of these directories from all scans, searches, and directory listings unless explicitly necessary.
- **When a specific dependency must be inspected** (e.g., to understand an API, trace a bug, or verify a behavior): look up only that package directly — do not traverse the full dependency tree or read unrelated packages.
- **Sub-dependencies** (packages required by packages) should never be read speculatively. Only inspect them if there is a concrete reason tied to the current task.
- Apply the same principle to other large generated or third-party directories: `dist/`, `build/`, `.git/`, `Pods/`, `__pycache__/`, etc. — prefer source over build artifacts.

---

### WordPress-Specific Validation
- **PHP:** Follow WordPress Coding Standards (use phpcs if available)
- **Gutenberg blocks:** Test in block editor before committing
- **Hooks/Filters:** Verify no conflicts with WordPress core or other plugins

---

## Special Instructions

### File Versioning (WordPress)
WordPress themes use file modification time for cache busting:
```php
function get_file_mtime( $filename ) {
  $filepath = get_template_directory() . '/' . $filename;
  if ( file_exists( $filepath ) ) {
    return filemtime( $filepath );
  }
  return null;
}
```
**Impact:** File timestamps are used for CSS/JS versioning, so no manual version bumps needed

---

## Pull Request Guidelines

Use the `/pull-request` prompt in Copilot Chat for complete PR writing standards.

**Critical:** Always verify actual git commits/diffs before documenting changes — only include what's actually in the code.

---

## Code Review Guidelines

Use the `/code-review` prompt in Copilot Chat for the complete code review process and criteria.

**Quick Request Format:**
```
/code-review
Review all uncommitted work for [TICKET-NUMBER] - [Ticket Name]
Focus on: best practices, security, maintainability, and functionality
```

---

## Ticket Writing Guidelines

Use the `/ticket-writing` prompt in Copilot Chat for complete ticket writing standards.

**Required sections:** Problem/Summary, Background/Context, Implementation Notes (with GitHub permalinks), Acceptance Criteria, References.

---

## Critical Thinking & Honest Assessment

**Do not simply agree with a request because the developer suggested it.** The goal of every interaction is to leave the codebase better and stronger than before — not to validate ideas or produce quick output.

### Core Principle: Raise Concerns Before Executing
If a request has a problem, say so clearly and explain why *before* writing any code. Do not implement a flawed approach and mention the issue as a footnote.

### When to Push Back
- **Architectural risk:** The requested change conflicts with an established pattern, introduces tight coupling, or will create tech debt that undermines future work on this project.
- **Best practice violation:** The approach goes against the standards in these instructions (security, testing, error handling, code style, etc.).
- **Breaking changes:** The change could break existing functionality, APIs, or integrations — even if not immediately obvious.
- **Scope creep risk:** The request will work but is the wrong layer to solve the problem (e.g., fixing a symptom instead of the root cause).
- **Project context conflict:** The suggestion doesn't account for how this specific project is structured, deployed, or consumed by other systems.

### How to Push Back
- Be direct and specific: name exactly what the problem is and which part of the request causes it.
- Provide an alternative: don't just say no — offer the better path and explain why it's stronger.
- If the request is sound but needs a small correction, implement the corrected version and note the deviation.
- If there are tradeoffs with no clear winner, present them objectively so the developer can decide with full information.

### The Quality Standard
Every change — no matter how small — should leave the relevant code in a better state than it was found. "Working" is the floor, not the ceiling. Consider:
- Is this maintainable by someone unfamiliar with the context?
- Does it handle failure cases and edge cases?
- Does it align with the existing architecture of this project?
- Will it hold up as the project grows?

If the answer to any of these is no, that gap should be addressed or flagged — not silently shipped.

---

## When to Deviate from These Guidelines

These guidelines are strong defaults, but there are situations where deviation is appropriate:

### Acceptable Deviations
- **Hot fixes for production incidents:** Speed matters more than perfection. Document shortcuts in commit message.
- **Prototypes and proof-of-concepts:** Mark clearly with comments. Don't merge to production without refactoring.
- **Legacy code maintenance:** Match existing patterns in legacy files to maintain consistency, but document tech debt.
- **External library constraints:** Some libraries require specific patterns (e.g., class components for certain HOCs).
- **Performance critical code:** Micro-optimizations may require less idiomatic code. Document and benchmark.
- **Framework conventions:** Follow framework-specific patterns even if they differ from our standards (e.g., Next.js, WordPress).

### Documentation Requirements
- Always document why you deviated in PR description or commit message
- Link to relevant issue, discussion, or benchmark
- Create follow-up ticket for tech debt if temporary deviation
- Add inline comments explaining non-standard patterns

### When in Doubt
- Follow the guideline by default
- Discuss with team if significant deviation needed
- Prioritize: Security > Correctness > Readability > Performance > Brevity

---

## Project-Specific Instructions

Load a project skill at the start of a session (e.g., `/myprojectskill`). Project skills live in `~/Websites/agents/local/skills/` and are machine-specific — add the relevant ones for your environment. See `README.md` for setup instructions.

---

*Last Updated: 2026-07-31*
*Version: 3.2.0*
