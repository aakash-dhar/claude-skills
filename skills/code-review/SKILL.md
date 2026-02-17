---
name: code-review
description: Reviews code for bugs, security issues, performance problems, and adherence to best practices. Use when the user asks to "review this code", "check my code", "is this code good?", or before submitting a PR.
allowed-tools: Read, Grep, Glob, Bash
---

# Code Review Skill

When reviewing code, follow this structured process:

## 1. Understand the Context
- What does this code do? Summarize its purpose in 1-2 sentences
- What files were changed and why?
- If reviewing a diff, understand both the before and after

## 2. Correctness
- Are there any logic bugs?
- Are edge cases handled (null, empty, zero, negative, boundary values)?
- Are error paths handled properly with meaningful error messages?
- Are return types and values correct?
- Are async operations handled properly (missing await, race conditions)?

## 3. Security
- SQL injection or NoSQL injection risks
- XSS vulnerabilities (unsanitized user input rendered in HTML)
- Hardcoded secrets, API keys, or credentials
- Insecure use of eval(), innerHTML, or dynamic code execution
- Missing authentication or authorization checks
- Sensitive data exposure in logs or error messages

## 4. Performance
- Unnecessary loops or O(n²) operations
- Missing database indexes for frequent queries
- N+1 query problems
- Large objects held in memory unnecessarily
- Missing pagination on list endpoints
- Expensive operations inside loops that could be batched

## 5. Readability & Maintainability
- Are variable and function names clear and descriptive?
- Are functions small and focused (single responsibility)?
- Is there duplicated code that should be extracted?
- Are magic numbers or strings replaced with named constants?
- Is complex logic commented or self-documenting?

## 6. Testing
- Are there tests for the new/changed code?
- Do tests cover happy path AND error cases?
- Are tests testing behavior, not implementation details?
- Are mocks used appropriately (not over-mocked)?

## 7. Project Standards
- Does the code follow the project's existing patterns and conventions?
- Are imports organized consistently?
- Does it match the linting and formatting rules?
- Are types properly defined (no unnecessary `any` in TypeScript)?

## Output Format

For each issue found, report it as:

**[SEVERITY] Category — File:Line**
Description of the issue.

Suggested fix:
```
// corrected code here
```

Severity levels:
- 🔴 **CRITICAL** — Bugs, security vulnerabilities, data loss risks. Must fix.
- 🟡 **WARNING** — Performance issues, missing error handling, potential problems. Should fix.
- 🟢 **SUGGESTION** — Readability, style, minor improvements. Nice to have.

## Summary

End every review with:
1. **Overall assessment** — Is this safe to merge? (Yes / Yes with changes / No)
2. **Critical issues count** — How many must-fix items
3. **Top 3 things done well** — Always highlight positives
4. **Top 3 improvements** — Most impactful changes to make