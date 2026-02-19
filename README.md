# 🛠️ Claude Code Skills for Dev Teams

A comprehensive collection of 23 Claude Code skills designed to boost developer and TPM productivity across the entire software development lifecycle — from planning and estimating to writing code, shipping it, and managing what happens after.

These skills work as `/slash-commands` in Claude Code and can also be auto-invoked by Claude when your task matches the skill's description.

---

## 📋 Table of Contents

- [Quick Start](#-quick-start)
- [Skills Overview](#-skills-overview)
- [Installation](#-installation)
- [Skills Reference — Development](#-skills-reference--development)
  - [explain-code](#1--explain-code)
  - [code-review](#2--code-review)
  - [optimize](#3--optimize)
  - [security-audit](#4--security-audit)
  - [test-coverage](#5--test-coverage)
  - [write-tests](#6--write-tests)
  - [refactor](#7--refactor)
  - [document-code](#8--document-code)
  - [explain-codebase](#9--explain-codebase)
  - [setup-guide](#10--setup-guide)
  - [create-pr](#11--create-pr)
  - [changelog](#12--changelog)
  - [create-ticket](#13--create-ticket)
- [Skills Reference — TPM & Project Management](#-skills-reference--tpm--project-management)
  - [estimate](#14--estimate)
  - [tech-decision](#15--tech-decision)
  - [impact-analysis](#16--impact-analysis)
  - [scope-check](#17--scope-check)
  - [build-vs-buy](#18--build-vs-buy)
  - [incident-report](#19--incident-report)
  - [tech-debt-report](#20--tech-debt-report)
  - [pentest-report](#21--pentest-report)
  - [risk-register](#22--risk-register)
  - [vulnerability-report](#23--vulnerability-report)
- [Supported Stacks](#-supported-stacks)
- [Workflow Recipes](#-workflow-recipes)
- [Customization](#-customization)
- [FAQ](#-faq)
- [Contributing](#-contributing)

---

## 🚀 Quick Start

**1. Clone into your project:**

```bash
# Copy the skills into your project's .claude directory
cp -r skills/ .claude/skills/
```

**2. Open Claude Code in your project:**

```bash
claude
```

**3. Use any skill:**

```bash
/code-review src/api/routes/users.ts
/write-tests src/services/orderService.ts
/security-audit
```

Or just ask naturally — Claude auto-detects when a skill is relevant:

```
> "This function is really slow, can you help?"        → triggers /optimize
> "I'm new to this project, walk me through it"        → triggers /explain-codebase
> "Write tests for the auth module"                    → triggers /write-tests
> "Should we build auth or use Clerk?"                 → triggers /build-vs-buy
> "Is this ticket bigger than it sounds?"              → triggers /scope-check
> "We had a production outage, write a postmortem"     → triggers /incident-report
> "What are the project risks?"                        → triggers /risk-register
> "Scan our packages for vulnerabilities"              → triggers /vulnerability-report
```

---

## 📦 Skills Overview

### Development Skills

| # | Skill | Purpose | Trigger Examples |
|---|-------|---------|-----------------|
| 1 | `/explain-code` | Understand code with analogies and diagrams | "how does this work?", "explain this" |
| 2 | `/code-review` | Quality check before merging | "review this code", "is this good?" |
| 3 | `/optimize` | Find and fix performance issues | "this is slow", "make this faster" |
| 4 | `/security-audit` | Scan for vulnerabilities | "is this secure?", "security review" |
| 5 | `/test-coverage` | Find missing tests and prioritize | "what's not tested?", "check coverage" |
| 6 | `/write-tests` | Generate comprehensive tests | "write tests for this", "add tests" |
| 7 | `/refactor` | Detect code smells and restructure | "clean up this code", "refactor this" |
| 8 | `/document-code` | Generate docs, docstrings, READMEs | "document this", "add docs" |
| 9 | `/explain-codebase` | Full project overview for onboarding | "explain this project", "I'm new here" |
| 10 | `/setup-guide` | Dev environment setup with checks | "how do I run this?", "setup guide" |
| 11 | `/create-pr` | Prepare and submit pull requests | "create a PR", "submit for review" |
| 12 | `/changelog` | Generate release notes | "what changed?", "release notes" |
| 13 | `/create-ticket` | Turn ideas into structured tickets | "log a bug", "create a ticket" |

### TPM & Project Management Skills

| # | Skill | Purpose | Trigger Examples |
|---|-------|---------|-----------------|
| 14 | `/estimate` | Effort estimation with breakdown | "how long will this take?", "size this ticket" |
| 15 | `/tech-decision` | Evaluate "should we do X?" proposals | "should we switch to", "compare X vs Y" |
| 16 | `/impact-analysis` | Map blast radius of proposed changes | "what would break if", "who is impacted" |
| 17 | `/scope-check` | Detect hidden complexity and scope creep | "is this bigger than it sounds?", "check this spec" |
| 18 | `/build-vs-buy` | Build in-house vs adopt vendor analysis | "should we build our own", "roll our own or use" |
| 19 | `/incident-report` | Blameless postmortem generation | "we had an outage", "write a postmortem" |
| 20 | `/tech-debt-report` | Quantify and prioritize tech debt | "how much tech debt?", "codebase health check" |
| 21 | `/pentest-report` | OWASP-based penetration testing report | "OWASP audit", "pentest report" |
| 22 | `/risk-register` | Living project risk tracking and assessment | "what could go wrong", "project risks" |
| 23 | `/vulnerability-report` | Three-tier CVE dependency scan reports | "vulnerable packages", "CVE report" |

---

## 📥 Installation

### Option A: Project-Level (Recommended for Teams)

Install into your project so the entire team gets the skills automatically via Git:

```bash
# From your project root
mkdir -p .claude/skills
cp -r skills/* .claude/skills/

# Commit for your team
git add .claude/skills/
git commit -m "feat(skills): add Claude Code skills for dev team"
git push
```

Now anyone who clones the repo has all the skills available.

### Option B: Global (Personal Use Across All Projects)

Install globally so skills are available in every project:

```bash
# Copy to your global Claude config
cp -r skills/* ~/.claude/skills/
```

### Option C: Cherry-Pick Individual Skills

Only want a few skills? Copy just the ones you need:

```bash
# Example: just code-review and write-tests
mkdir -p .claude/skills
cp -r skills/code-review .claude/skills/
cp -r skills/write-tests .claude/skills/
```

### Verify Installation

Open Claude Code and type `/` — you should see your skills in the autocomplete list. You can also run:

```
/help
```

to see all available commands including your installed skills.

---

## 📖 Skills Reference — Development

---

### 1. 💡 explain-code

**What it does:** Explains code using everyday analogies, ASCII diagrams, step-by-step walkthroughs, and highlights common gotchas.

**Best for:** Understanding unfamiliar code, learning a new codebase, code walkthroughs with teammates.

**How to use:**

```bash
# Point it at a specific file
/explain-code src/services/paymentService.ts

# Point it at a function
/explain-code src/utils/parseToken.ts

# Or just ask naturally
> "How does the authentication flow work?"
> "What does this function do?"
```

**What you get:**

```
1. Analogy     → "Think of this like a restaurant order system..."
2. Diagram     → ASCII art showing the flow and relationships
3. Walkthrough → Step-by-step explanation of what happens
4. Gotcha      → Common mistakes and misconceptions
```

---

### 2. 🔍 code-review

**What it does:** Reviews code for bugs, security issues, performance problems, readability, test coverage, and project standards. Provides a merge readiness assessment.

**Best for:** Pre-merge reviews, catching issues early, maintaining code quality standards.

**How to use:**

```bash
# Review a specific file
/code-review src/api/routes/users.ts

# Review multiple files
/code-review src/services/

# Review staged changes
/code-review

# Or just ask naturally
> "Review my changes before I submit the PR"
> "Is this code good enough to merge?"
```

**What you get:**

Each issue is reported with severity:

```
🔴 CRITICAL  → Must fix before merge (bugs, security vulnerabilities)
🟡 WARNING   → Should fix (performance, missing error handling)
🟢 SUGGESTION → Nice to have (readability, style improvements)
```

Plus a summary:
- Overall merge readiness (Yes / Yes with changes / No)
- Critical issues count
- Top 3 things done well
- Top 3 improvements needed

---

### 3. ⚡ optimize

**What it does:** Analyzes code for performance issues including slow algorithms, memory leaks, database query problems, frontend bottlenecks, and API inefficiencies.

**Best for:** Debugging slow endpoints, reducing load times, fixing memory issues, optimizing database queries.

**How to use:**

```bash
# Analyze a specific file
/optimize src/api/routes/products.ts

# Analyze a module
/optimize src/services/search/

# Or just ask naturally
> "The /api/orders endpoint takes 3 seconds, why?"
> "Our bundle size is 2MB, help me reduce it"
> "This page re-renders too often"
```

**What you get:**

Each issue reported with impact level:

```
🔴 HIGH   → Noticeable slowdown or crashes at scale
🟡 MEDIUM → Degrades under load
🟢 LOW    → Minor inefficiency
```

Includes before/after code examples, complexity analysis (O notation), and estimated improvement.

**Covers:**
- Algorithm & data structure analysis
- Database queries (N+1, missing indexes, over-fetching)
- Memory leaks and unbounded caches
- Frontend (re-renders, bundle size, lazy loading, virtualization)
- API & network (caching, batching, parallelization)
- Concurrency and async patterns

---

### 4. 🔒 security-audit

**What it does:** Scans code for security vulnerabilities including injection attacks, exposed secrets, authentication flaws, insecure dependencies, and data exposure.

**Best for:** Pre-deployment security checks, compliance reviews, identifying vulnerabilities before they're exploited.

**How to use:**

```bash
# Audit the entire project
/security-audit

# Audit specific area
/security-audit src/api/

# Audit a single file
/security-audit src/auth/login.ts

# Or just ask naturally
> "Is our auth flow secure?"
> "Check for exposed secrets in the codebase"
> "We're deploying Friday, run a security check"
```

**What you get:**

Each vulnerability reported with severity:

```
🔴 CRITICAL → Actively exploitable (data breach, RCE risk)
🟠 HIGH     → Exploitable with some effort
🟡 MEDIUM   → Requires specific conditions
🟢 LOW      → Defense-in-depth improvement
```

Each finding includes the vulnerability description, risk assessment, proof of concept, fix with code, and CWE/OWASP reference.

**Covers:**
- Secrets & credentials exposure
- Injection attacks (SQL, NoSQL, XSS, command, path traversal, template)
- Authentication & authorization flaws (IDOR, privilege escalation)
- Data exposure (PII in logs, verbose errors, CORS misconfiguration)
- Input validation gaps
- Dependency vulnerabilities (CVEs, outdated packages)
- HTTP security headers
- Cryptography issues
- Infrastructure & configuration

**Stack-specific checks for:** Node.js/Express, Python/Django, Python/Flask, React/Next.js, Vue/Nuxt, Ruby on Rails, Go, Java/Spring Boot, PHP/Laravel, Mobile (React Native/Flutter), Payment security, AWS/Cloud, Docker/Containers.

---

### 5. 📊 test-coverage

**What it does:** Analyzes test coverage gaps, identifies untested code paths, and prioritizes which tests to write based on risk level.

**Best for:** Understanding what's tested and what's not, planning test improvement work, ensuring critical code is covered.

**How to use:**

```bash
# Analyze entire project
/test-coverage

# Analyze specific module
/test-coverage src/services/

# Analyze a single file
/test-coverage src/auth/login.ts

# Or just ask naturally
> "What tests are we missing?"
> "Which tests should I write first?"
> "Check our test coverage"
```

**What you get:**

Coverage report with gaps prioritized by risk:

```
🔴 Critical → Auth, payments, data mutation — test first
🟠 High     → Error handling, DB queries, integrations — test next
🟡 Medium   → UI components, utilities, middleware — test when possible
🟢 Low      → Static components, types, constants — test if time permits
```

Plus:
- Overall coverage metrics (line, branch, function)
- Files with 0% coverage
- Test quality analysis (weak assertions, flaky tests, over-mocking)
- Prioritized improvement plan with test scaffolding

---

### 6. ✅ write-tests

**What it does:** Generates comprehensive, runnable test files including unit tests, integration tests, edge cases, error paths, and proper mocking.

**Best for:** Adding test coverage quickly, learning testing patterns, covering new features with tests.

**How to use:**

```bash
# Generate tests for a file
/write-tests src/services/orderService.ts

# Generate tests for a component
/write-tests src/components/LoginForm.tsx

# Generate tests for an API route
/write-tests src/api/routes/users.ts

# Or just ask naturally
> "Write tests for the payment service"
> "Add integration tests for the orders API"
> "Cover the auth module with tests"
```

**What you get:**

Complete, runnable test files that match your project's conventions:

- **Test plan** — list of all test cases organized by category
- **Full test file** — ready to save and run
- **Test factories** — reusable test data builders
- **Setup instructions** — any dependencies to install

**Test categories covered:**
- Happy path (normal successful operation)
- Edge cases (boundary values, empty inputs, limits)
- Error cases (invalid inputs, failures, exceptions)
- Integration points (database, API, file system)
- State transitions (before/after)
- Security (malicious inputs)

**Full examples provided for:** Jest, Vitest, React Testing Library, Pytest, Go testing, RSpec, PHPUnit, Supertest (API tests), FastAPI/httpx.

---

### 7. ♻️ refactor

**What it does:** Detects code smells, suggests design pattern improvements, and provides step-by-step restructuring guidance while preserving existing behavior.

**Best for:** Cleaning up messy code, reducing complexity, improving maintainability, paying down tech debt.

**How to use:**

```bash
# Refactor a specific file
/refactor src/services/userService.ts

# Refactor a module
/refactor src/api/routes/

# Or just ask naturally
> "This file is a mess, clean it up"
> "This function is 200 lines long, break it up"
> "Find code smells in the services folder"
> "Make the auth flow simpler"
```

**What you get:**

Each smell reported with severity and fix:

```
🔴 Critical  → Long functions, god classes, deep nesting, duplication
🟡 Warning   → Feature envy, long parameter lists, switch chains
🟢 Minor     → Inconsistent naming, comments that should be code
```

Plus:
- Before/after code for each refactoring
- Prioritized refactoring plan (quick wins → medium effort → large refactors)
- Complexity metrics (before and after)
- Step-by-step safe refactoring workflow

**15+ refactoring techniques covered:**
Extract Function, Extract Class, Replace Conditional with Polymorphism, Introduce Parameter Object, Guard Clauses, Decompose Conditional, Replace Temp with Query, Move Function, and more.

**Stack-specific patterns for:** TypeScript/JavaScript, Python, Go, React/Next.js, Vue/Nuxt, Java/Spring Boot, Ruby/Rails, PHP/Laravel, Database/SQL.

---

### 8. 📝 document-code

**What it does:** Generates documentation including function docstrings, module headers, README files, API documentation, and architecture overviews.

**Best for:** Adding documentation to undocumented code, generating READMEs for new modules, creating API docs.

**How to use:**

```bash
# Document a file
/document-code src/services/orderService.ts

# Document a module
/document-code src/api/

# Generate a README
/document-code --readme src/payments/

# Or just ask naturally
> "Add JSDoc to all functions in this file"
> "Generate a README for the payments service"
> "Document our API endpoints"
> "This module has no documentation, fix that"
```

**What you get:**

- **Inline comments** — for complex logic, workarounds, regex, magic numbers
- **Function docstrings** — params, returns, errors, examples, side effects
- **Module headers** — purpose, dependencies, key decisions
- **README** — overview, architecture, API reference, setup, configuration
- **API documentation** — endpoints, auth, request/response, examples, errors

**Docstring formats for 10+ languages:**
TypeScript/JSDoc, Python/Google Style, Go/Godoc, Java/Javadoc, Ruby/YARD, PHP/PHPDoc, Rust/Rustdoc, Swift, Kotlin/KDoc.

---

### 9. 🗺️ explain-codebase

**What it does:** Generates a comprehensive overview of an entire codebase including architecture diagrams, directory maps, domain models, critical paths, data flow, and tribal knowledge.

**Best for:** Onboarding new team members, architecture reviews, knowledge transfer, understanding inherited codebases.

**How to use:**

```bash
# Explain the entire project
/explain-codebase

# Explain a specific area
/explain-codebase src/

# Or just ask naturally
> "I just joined the team, walk me through this project"
> "Give me an architecture overview"
> "How is this codebase organized?"
> "Map out the codebase for me"
```

**What you get:**

1. **30-Second Overview** — what it is, who it's for, tech stack
2. **Architecture Diagram** — ASCII visual of all major components
3. **Directory Map** — what each folder does and why it's organized that way
4. **Domain Model** — core entities and relationships with diagrams
5. **Critical Paths** — 2-3 key user flows traced through the code
6. **Data Flow** — where data lives (DB, cache, client, S3) and how it moves
7. **External Dependencies** — third-party services and integrations
8. **Configuration** — environment variables and environment differences
9. **Development Workflow** — git workflow, common commands, CI/CD
10. **Gotchas & Tribal Knowledge** — non-obvious things that aren't in the code
11. **"Where to Look When..."** — quick reference for common tasks

---

### 10. 🏗️ setup-guide

**What it does:** Auto-detects the project stack, checks prerequisites, generates step-by-step setup instructions, and verifies everything works.

**Best for:** New team member onboarding, setting up after a fresh clone, documenting the setup process.

**How to use:**

```bash
# Generate setup guide for current project
/setup-guide

# Or just ask naturally
> "I just cloned this repo, help me get it running"
> "How do I set up this project locally?"
> "Walk me through the dev environment setup"
```

**What you get:**

1. **Prerequisites table** — tools needed with version check results (✅/❌)
2. **Install missing tools** — OS-specific commands for anything not found
3. **Step-by-step setup** — from clone to running, customized to your stack
4. **Verification checklist** — confirm infrastructure, app, and dev tools work
5. **IDE setup** — recommended extensions and settings for VS Code, JetBrains
6. **Useful commands** — daily reference table
7. **Troubleshooting** — common issues and fixes
8. **Credentials & access** — dev accounts, seed users, external service keys
9. **Next steps** — first file to read, suggested first task

**Auto-detects:** Node.js, Python, Go, Ruby, Java, Rust, PHP, Flutter; npm/yarn/pnpm/bun; Docker, databases, and framework.

---

### 11. 🚀 create-pr

**What it does:** Analyzes your changes, runs pre-PR checks (lint, tests, build), generates a conventional title and descriptive body, and creates the PR via CLI.

**Best for:** Submitting polished PRs quickly, ensuring nothing is forgotten, maintaining consistent PR quality across the team.

**How to use:**

```bash
# Create a PR for current branch
/create-pr

# Create as draft
/create-pr --draft

# Or just ask naturally
> "I'm done with my changes, create a PR"
> "Prepare a pull request for this branch"
> "Submit this for review"
```

**What you get:**

1. **Pre-check results** — lint, tests, build, type check, debug statement scan
2. **Conventional title** — `feat(auth): add OAuth2 login with Google and GitHub`
3. **Structured body** — what, why, how, changes, testing instructions, checklist
4. **Auto-detected labels** — feature, bug, size/small, security, etc.
5. **Suggested reviewers** — based on file ownership and CODEOWNERS
6. **PR creation** — via `gh` (GitHub), `glab` (GitLab), or `az` (Azure DevOps)

**Adapts body format for:** bug fixes, features, refactors, dependency updates, and database migrations.

---

### 12. 📋 changelog

**What it does:** Generates release notes by analyzing git history, commits, merged PRs, and tags between versions.

**Best for:** Preparing releases, communicating changes to users/stakeholders, maintaining a CHANGELOG.md.

**How to use:**

```bash
# Generate changelog since last tag
/changelog

# Generate between specific versions
/changelog v2.3.0..v2.4.0

# Or just ask naturally
> "What changed since the last release?"
> "Generate release notes for v3.0"
> "Write release notes for our users"
```

**What you get:**

- **Categorized changes** — features, fixes, performance, security, breaking, dependencies
- **Version bump suggestion** — MAJOR/MINOR/PATCH with reasoning
- **Multiple output formats:**
  - **Keep a Changelog** — for libraries and open source
  - **User-facing release notes** — for products and apps
  - **Developer release notes** — for internal engineering teams
  - **Compact format** — for rapid release cycles
- **Deployment checklist** — migrations, env vars, rollback plan
- **GitHub/GitLab release creation** — via CLI

---

### 13. 🎫 create-ticket

**What it does:** Turns bug reports, feature ideas, tech debt, and improvement requests into well-structured tickets with acceptance criteria, estimates, and subtasks.

**Best for:** Quickly logging bugs, writing feature specs, breaking down work, maintaining consistent ticket quality.

**How to use:**

```bash
# Create a ticket interactively
/create-ticket

# Or just describe the issue naturally
> "Create a ticket: users can't reset their password if they signed up with OAuth"
> "Write a feature ticket for adding dark mode support"
> "Log a tech debt ticket for the messy order service"
```

**What you get:**

Structured ticket based on type:

| Type | Includes |
|------|----------|
| 🐛 Bug | Steps to reproduce, expected vs actual, severity, root cause, fix suggestion |
| ✨ Feature | User story, requirements, user flow, API changes, DB changes, acceptance criteria |
| 🏗️ Tech Debt | Problem description, impact, approach, effort estimate, risk |
| 🔍 Spike | Research questions, time-box, expected output |
| 🔒 Security | Vulnerability description, risk, remediation steps |

All tickets include: acceptance criteria, subtasks, dependencies, estimates, and labels.

---

## 📖 Skills Reference — TPM & Project Management

These skills are designed for Technical Project Managers and team leads. They produce structured analysis documents saved to `project-decisions/` in your project root, building a searchable decision history over time.

---

### 14. 🎯 estimate

**What it does:** Estimates effort for tasks and tickets by analyzing code complexity, dependencies, scope, risk, and historical patterns. Supports T-shirt sizing, story points, time ranges, and task-based estimates.

**Best for:** Sprint planning, backlog grooming, timeline forecasting, answering "can we fit this in the sprint?"

**How to use:**

```bash
# Estimate a feature
/estimate "Add OAuth2 login with Google and GitHub"

# Estimate from a ticket
/estimate "Migrate from REST to GraphQL for the user service"

# Or just ask naturally
> "How long would it take to add dark mode?"
> "Size this ticket for me"
> "Can we fit this in the current sprint?"
```

**What you get:**

- **Task breakdown** — decomposed subtasks with individual estimates
- **Multiple estimation methods** — T-shirt, story points (Fibonacci), time ranges (PERT), task-based
- **Risk assessment** — factors that could inflate the estimate with multipliers
- **Confidence level** — High (±10%), Medium (±25%), Low (±50%) with range
- **Historical comparison** — past similar work and how long it actually took
- **Sprint fit analysis** — does it fit in remaining capacity?
- **Decomposition recommendations** — how to split if it's too large

---

### 15. 🧭 tech-decision

**What it does:** Evaluates technical proposals and "should we do X instead of Y?" questions. Analyzes feasibility, compares options with structured pros/cons, estimates effort and risk, and provides a clear recommendation.

**Best for:** Evaluating team proposals, technical direction decisions, tool/approach comparisons.

**Output:** Saved to `project-decisions/YYYY-MM-DD-[topic].md`

**How to use:**

```bash
# Evaluate a proposal
/tech-decision "Should we migrate from REST to GraphQL for our user service?"

# Evaluate a team suggestion
/tech-decision "Dan thinks we should hook up an agent to BigQuery instead of Looker Studio"

# Or just ask naturally
> "Should we switch from Webpack to Vite?"
> "Is it worth migrating to Next.js 15?"
> "Compare PostgreSQL vs DynamoDB for our events table"
```

**What you get:**

- **Decision framing** — clear statement of what's being decided and constraints
- **3+ options compared** — always includes status quo and at least one alternative
- **Feasibility analysis** — can we actually do this?
- **Effort & timeline** — phased breakdown for each option
- **Cost analysis** — one-time and ongoing costs compared
- **Risk assessment** — per-option risk register with mitigations
- **Weighted comparison matrix** — scored across 7 criteria
- **Clear recommendation** — Go / Spike First / Defer / Don't Do with reasoning
- **Next steps** — actionable plan based on the recommendation

---

### 16. 💥 impact-analysis

**What it does:** Maps the full blast radius of a proposed change — which code, APIs, databases, services, infrastructure, teams, and workflows are affected. Detects breaking changes and produces a rollout strategy.

**Best for:** Pre-implementation planning, change management, preventing surprises in production.

**Output:** Saved to `project-decisions/YYYY-MM-DD-impact-[topic].md`

**How to use:**

```bash
# Analyze a proposed change
/impact-analysis "Renaming the users table to accounts"

# Analyze a migration
/impact-analysis "Replacing our custom auth with Clerk"

# Or just ask naturally
> "What would break if we change the order status from string to enum?"
> "What's the blast radius of upgrading to Next.js 15?"
> "Who is impacted if we remove the v1 API?"
```

**What you get:**

- **Blast radius summary** — files, endpoints, tables, services, teams affected (quantified)
- **Dependency graph** — ASCII diagram showing direct and transitive dependencies
- **Breaking change detection** — API, database, and library breaking changes flagged
- **Team & people impact** — who needs to know and what they need to do
- **Communication plan** — when and how to notify affected parties
- **Migration & rollout strategy** — phased plan for breaking changes
- **Rollback plan** — how to revert if things go wrong
- **Risk matrix** — impact vs likelihood with mitigation strategies
- **Change-type checklist** — API, database, service, infrastructure, or library specific

---

### 17. 🔎 scope-check

**What it does:** Analyzes feature specs, tickets, and PRDs for hidden complexity, scope creep risks, missing requirements, ambiguous language, unstated assumptions, and unrealistic timelines.

**Best for:** Pre-sprint commitment review, catching "2-week projects" that are really 3-month projects, improving spec quality.

**Output:** Saved to `project-decisions/YYYY-MM-DD-scope-[topic].md`

**How to use:**

```bash
# Check a feature spec
/scope-check "Add real-time notifications — should be a simple WebSocket integration, probably 3-4 days"

# Check a ticket
/scope-check "JIRA-456: search products by name, category, and price range. Estimate: 5 points"

# Or just ask naturally
> "Is this ticket bigger than it sounds?"
> "Sanity check this PRD before we commit"
> "What are we missing in this feature spec?"
> "Can we really do this in 2 weeks?"
```

**What you get:**

- **Verdict** — 🟢 Realistic / 🟡 Needs Refinement / 🟠 Underscoped / 🔴 Unrealistic
- **Red flags** — critical, warning, and minor flags with specific explanations
- **Hidden complexity** — what the spec says vs what it actually requires (with effort delta)
- **Hidden icebergs** — common features that are 5x harder than they sound
- **Missing requirements checklist** — functional, non-functional, design, operational, delivery
- **Unstated assumptions** — assumptions the spec relies on that haven't been validated
- **Effort multipliers** — spec clarity, team experience, dependencies, data changes
- **Scope reduction recommendations** — MoSCoW prioritization, phased delivery, cut suggestions
- **Questions to resolve** — specific questions that must be answered before committing

---

### 18. ⚖️ build-vs-buy

**What it does:** Compares building a solution internally versus buying/adopting a third-party tool or service. Analyzes total cost of ownership, effort, risk, vendor lock-in, feature fit, and strategic alignment.

**Best for:** Evaluating "should we roll our own or use a service?", vendor selection, make-or-buy decisions.

**Output:** Saved to `project-decisions/YYYY-MM-DD-build-vs-buy-[topic].md`

**How to use:**

```bash
# Compare build vs buy
/build-vs-buy "Should we build our own auth or use Clerk/Auth0?"

# Evaluate vendor options
/build-vs-buy "We need full-text search — Elasticsearch self-hosted vs Algolia?"

# Or just ask naturally
> "Should we roll our own email sending or use SendGrid?"
> "Build or buy for our analytics dashboard?"
> "Should we self-host Sentry or use their cloud?"
```

**What you get:**

- **Problem framing** — requirements, constraints, and success criteria
- **3+ options evaluated** — Build, Buy (Vendor A), Buy (Vendor B), OSS, Hybrid
- **Feature fit matrix** — must-have and nice-to-have requirements per option
- **3-year TCO comparison** — development, infrastructure, maintenance, hidden costs
- **Time to value** — how fast each option delivers a working solution
- **Risk assessment** — build risks, buy risks, OSS risks with mitigations
- **Vendor lock-in score** — data portability, API standards, contract terms
- **Strategic alignment** — is this a differentiator or a commodity?
- **Vendor health check** — funding, customer count, certifications, SLA, support quality
- **Weighted comparison matrix** — scored across 8 criteria
- **Implementation plan** — phased plan for the recommended option
- **Exit strategy** — how to switch if the decision needs to be reversed

---

### 19. 🚨 incident-report

**What it does:** Generates blameless postmortem reports by analyzing git history, recent deployments, code changes, and error patterns. Produces a structured report with timeline, root cause analysis, impact assessment, and action items.

**Best for:** Production incidents, outage postmortems, compliance-required incident documentation.

**Output:** Saved to `project-decisions/YYYY-MM-DD-incident-[topic].md`

**How to use:**

```bash
# Generate an incident report
/incident-report "Payment processing went down for 2 hours yesterday"

# Analyze a specific issue
/incident-report "Users reporting 500 errors on login since this morning's deploy"

# Or just ask naturally
> "Write a postmortem for the database outage"
> "We need an RCA for the webhook failures"
> "Document what happened with the payment processing incident"
```

**What you get:**

- **Executive summary** — what happened, impact, root cause, what we're doing about it
- **Severity classification** — SEV-1 through SEV-4 with category
- **Precise timeline** — minute-by-minute events from trigger to resolution
- **Key metrics** — time to detect, respond, mitigate, and resolve
- **Root cause analysis** — 5 Whys technique drilling to systemic causes
- **Contributing factors** — all factors that combined to cause the incident
- **Impact assessment** — users affected, requests failed, revenue impact, SLA impact
- **Response evaluation** — what went well, what didn't, where we got lucky
- **Action items** — prioritized by urgency (immediate, short-term, medium-term) with owners
- **Lessons learned** — key takeaways and how to apply them
- **Recurring incident check** — is this a pattern? Were previous actions completed?

---

### 20. 🏥 tech-debt-report

**What it does:** Quantifies technical debt across the codebase by scanning for code complexity, outdated dependencies, missing tests, TODOs, dead code, architectural issues, and change hotspots. Produces a prioritized remediation roadmap.

**Best for:** Sprint planning (20% debt budget), leadership reporting, quarterly health checks, making the case for debt paydown.

**Output:** Saved to `project-decisions/YYYY-MM-DD-tech-debt-report.md`

**How to use:**

```bash
# Full codebase health check
/tech-debt-report

# Check a specific area
/tech-debt-report src/services/

# Or just ask naturally
> "How much tech debt do we have?"
> "Give me a health check on the codebase"
> "What's the state of our codebase?"
> "Generate a tech debt report for the leadership review"
```

**What you get:**

- **Overall debt score** — X/10 with health rating (🟢 Healthy → 🔴 Critical)
- **9 category breakdown** — complexity, TODOs, dependencies, tests, docs, dead code, config, architecture, hotspots
- **Specific findings** — file names, line counts, exact numbers (not vague descriptions)
- **Trend analysis** — is debt increasing or decreasing over the last 6 months?
- **Bus factor report** — files only one person has ever changed
- **Change hotspot analysis** — most-changed files that also have the most bugs
- **Remediation roadmap** — quick wins (< 1 day), sprint-sized (1-5 days), large efforts (1-4 weeks)
- **Sprint allocation plan** — recommended 20% debt budget with specific items per sprint
- **Projected improvement** — debt score reduction after 1, 3, and 6 sprints

---

### 21. 🛡️ pentest-report

**What it does:** Generates a structured penetration testing report based on OWASP standards. Scans code for vulnerabilities, maps findings to OWASP Top 10 (2021), checks ASVS compliance, assigns CVSS v3.1 scores, and produces a professional security assessment.

**Best for:** Compliance reviews, pre-audit preparation, stakeholder security reporting, formal security assessments.

**Output:** Saved to `project-decisions/YYYY-MM-DD-pentest-[topic].md`

**How to use:**

```bash
# Full OWASP assessment
/pentest-report

# Assess a specific area
/pentest-report src/api/

# Or just ask naturally
> "Run an OWASP security assessment"
> "Generate a penetration testing report"
> "Check OWASP Top 10 compliance"
> "OWASP ASVS Level 2 compliance check"
```

**What you get:**

- **Executive summary** — overall risk rating, finding count by severity, key findings table
- **OWASP Top 10 coverage** — each category tested with specific checks and findings
- **CVSS v3.1 scoring** — every finding scored with attack vector, complexity, and impact
- **ASVS compliance matrix** — Level 1/2/3 verification requirements with pass/fail status
- **Detailed findings** — description, evidence (code), impact, proof of concept, remediation (code), CWE reference
- **Remediation roadmap** — immediate (0-48h), short-term (1-2 weeks), medium-term (1 month), long-term
- **Positive findings** — security controls that ARE properly implemented
- **Recommendations** — quick wins, process improvements, architecture improvements

**Methodology:** OWASP WSTG v4.2, OWASP Top 10 (2021), OWASP ASVS v4.0, CVSS v3.1

**When to use which security skill:**

| Skill | Use When |
|-------|----------|
| `/security-audit` | Quick security check during development or before a PR |
| `/pentest-report` | Formal assessment for compliance, audits, or stakeholder reporting |

---

### 22. ⚠️ risk-register

**What it does:** Creates and maintains a living project risk register by scanning the codebase, dependencies, team activity, and project history. Identifies technical, delivery, operational, and business risks, scores them by likelihood and impact, assigns owners, and tracks mitigations over time.

**Best for:** Sprint planning risk reviews, project kickoffs, stakeholder reporting, quarterly health checks, pre-launch risk assessment.

**Output:** Saved to `project-decisions/YYYY-MM-DD-risk-register.md` (updated in-place on subsequent runs)

**How to use:**

```bash
# Create or update the risk register
/risk-register

# Focus on a specific concern
/risk-register "We're 3 weeks from launch and just lost a senior developer"

# Or just ask naturally
> "What are the risks in this project?"
> "Create a risk register for our Q2 release"
> "Update our risk register — we added a new third-party integration"
> "What could go wrong before we launch?"
```

**What you get:**

- **Risk summary dashboard** — count by severity with trend indicators (↑ worsening / → stable / ↓ improving)
- **Risk heat map** — visual likelihood × impact matrix showing where all risks sit
- **Automated risk detection** — scans codebase for bus factor, untested code, vulnerable deps, missing monitoring, secrets exposure
- **Four risk categories** — Technical (T1-T8), Delivery (D1-D5), Operational (O1-O4), Business (B1-B4)
- **Scored entries** — each risk rated on 1-5 likelihood × 1-5 impact with color coding (🟢🟡🟠🔴)
- **Response strategy per risk** — Avoid, Mitigate, Transfer, Accept, or Contingency
- **Mitigation actions** — specific actions with owners and due dates
- **Trend tracking** — how each risk has changed over time across reviews
- **Realized risks** — risks that became actual incidents, linked to incident reports
- **Risk metrics** — total open, average score, mean time to mitigate, overdue count
- **Review cadence** — recommended sprint/monthly/quarterly review schedule

**Auto-flags as 🔴 Critical:**
- Dependencies with critical CVEs
- Secrets committed to git
- Zero test coverage on auth/payment code
- Key person dependency on critical path
- Deadline < 2 weeks with > 30% scope incomplete

---

### 23. 🛡️ vulnerability-report

**What it does:** Scans ALL project dependencies across every package ecosystem (npm, pip, Go, Ruby, PHP, Rust, Docker) for known CVEs, and generates three separate severity-tiered reports with full remediation guidance.

**Best for:** Security reviews, compliance audits, sprint planning (prioritizing fixes), dependency update cycles, CI/CD integration.

**Output:** Generates THREE files:
- `project-decisions/YYYY-MM-DD-vulnerabilities-high.md` — 🔴 Critical & High (fix within 48 hours)
- `project-decisions/YYYY-MM-DD-vulnerabilities-medium.md` — 🟡 Medium (fix within 2 weeks)
- `project-decisions/YYYY-MM-DD-vulnerabilities-low.md` — 🟢 Low (fix when convenient)

**How to use:**

```bash
# Full dependency vulnerability scan
/vulnerability-report

# Or just ask naturally
> "Scan our dependencies for vulnerabilities"
> "Generate a CVE report for our packages"
> "What vulnerable packages do we have?"
> "Run a dependency security scan"
```

**What you get per report:**

- **Executive summary** — vulnerability count, auto-fixable count, direct vs transitive breakdown
- **Quick fix commands** — copy-paste `npm audit fix`, `pip install --upgrade`, etc.
- **Detailed findings** — each vulnerability with:
  - Package name, installed version, fixed version
  - CVE ID, CVSS score and vector, CWE classification
  - Direct vs transitive dependency chain
  - Reachability analysis (is the vulnerable code actually called?)
  - Exploit availability (PoC exists? actively exploited?)
  - Remediation with exact commands and verification steps
- **Dependency health metrics** — outdated packages, deprecated packages, unpinned versions, lockfile status
- **Remediation priority table** — ranked by CVSS score, fix availability, and reachability
- **Cross-report links** — each report links to the other two severity tiers
- **Automation recommendations** — GitHub Actions / GitLab CI config for automated scanning, Dependabot/Renovate setup

**Ecosystems scanned:**

| Ecosystem | Scanner | Lockfile |
|-----------|---------|----------|
| Node.js (npm/yarn/pnpm) | `npm audit` | package-lock.json, yarn.lock, pnpm-lock.yaml |
| Python (pip/poetry) | `pip-audit`, `safety` | requirements.txt, poetry.lock |
| Go | `govulncheck` | go.sum |
| Ruby | `bundler-audit` | Gemfile.lock |
| PHP | `composer audit` | composer.lock |
| Rust | `cargo audit` | Cargo.lock |
| Docker | Base image version check | Dockerfile |

**When to use which security skill:**

| Skill | Scope | Use When |
|-------|-------|----------|
| `/security-audit` | Source code vulnerabilities | Quick check during development or before a PR |
| `/pentest-report` | Full OWASP application assessment | Formal compliance audits, stakeholder reporting |
| `/vulnerability-report` | Dependency/package CVEs | Dependency update cycles, CI/CD security gates |
| `/risk-register` | All project risks (including security) | Sprint planning, project-level risk management |

---

## 🔧 Supported Stacks

These skills include stack-specific checks and patterns for:

| Category | Technologies |
|----------|-------------|
| **Languages** | TypeScript, JavaScript, Python, Go, Java, Ruby, PHP, Rust, Swift, Kotlin |
| **Backend Frameworks** | Express, Fastify, NestJS, Django, Flask, FastAPI, Spring Boot, Rails, Laravel, Gin, Echo, Fiber |
| **Frontend Frameworks** | React, Next.js, Vue, Nuxt, Angular, Svelte |
| **Testing** | Jest, Vitest, Pytest, Go testing, RSpec, PHPUnit, JUnit, Testing Library |
| **Databases** | PostgreSQL, MySQL, MongoDB, Redis, Elasticsearch |
| **Mobile** | React Native, Flutter |
| **Infrastructure** | Docker, Kubernetes, AWS, Terraform, GitHub Actions |
| **Git Platforms** | GitHub, GitLab, Bitbucket, Azure DevOps |

---

## 🔄 Workflow Recipes

### Before Every PR

```bash
/code-review                    # Check code quality
/security-audit src/api/        # Scan for vulnerabilities  
/create-pr                      # Generate and submit the PR
```

### Before Every Release

```bash
/test-coverage                  # Check for coverage gaps
/security-audit                 # Full security scan
/changelog                      # Generate release notes
```

### Onboarding a New Team Member

```bash
/explain-codebase               # Architecture overview
/setup-guide                    # Get their environment running
```

### Paying Down Tech Debt

```bash
/tech-debt-report                # Quantify what needs fixing
/refactor src/services/         # Find code smells
/optimize src/api/              # Find performance issues
/test-coverage src/services/    # Find missing tests
/write-tests src/services/auth  # Generate the missing tests
```

### Evaluating a Technical Proposal

```bash
/scope-check "the feature spec"              # Is it bigger than it sounds?
/tech-decision "should we use X or Y?"       # Evaluate the options
/impact-analysis "implementing option X"     # Map the blast radius
/estimate "implement the chosen approach"    # Estimate the effort
/build-vs-buy "build or use a vendor?"       # If relevant
```

### After a Production Incident

```bash
/incident-report "description of what happened"   # Blameless postmortem
/tech-decision "should we add circuit breakers?"   # Evaluate the fix
/impact-analysis "implementing the fix"            # Map what's affected
```

### Security & Compliance Review

```bash
/security-audit                 # Quick scan for source code vulnerabilities
/vulnerability-report           # Three-tier dependency CVE reports
/pentest-report                 # Formal OWASP-based assessment
/risk-register                  # Update project risk register with findings
```

### Creating a Pre-Deploy Command

Create `.claude/commands/pre-deploy.md`:

```markdown
---
description: Full pre-deployment review — tests, security, performance, and code quality
---

Run a complete pre-deployment review:

1. Analyze test coverage and flag critical untested code
2. Run a security audit on all changed files since the last release
3. Check for performance issues
4. Do a final code review

Combine all findings into a single report grouped by file, with critical issues first.
```

Now `/pre-deploy` chains multiple skills into one command.

---

## ⚙️ Customization

### Adding Team-Specific Rules

Edit any `SKILL.md` to add your team's conventions. For example, in `code-review/SKILL.md`:

```markdown
## Team-Specific Rules
- All API endpoints must have rate limiting
- All database queries must go through the repository layer
- No direct DOM manipulation — use React state
- All env vars must be accessed through the config module
```

### Adjusting for Your Stack

If your team only uses Python/Django, you can simplify skills by keeping only the relevant stack-specific sections and removing others.

### Creating Combo Commands

Create `.claude/commands/` files that chain skills together:

```markdown
---
description: Full code quality check
---
Run /code-review, /optimize, and /security-audit on the changed files.
Produce a unified report.
```

### Listing Your Skills in CLAUDE.md

Add to your project's `CLAUDE.md` to improve auto-invocation:

```markdown
## Available Skills — Development
- `/explain-code` - Explains code with diagrams and analogies
- `/code-review` - Code quality review with severity ratings
- `/optimize` - Performance analysis and fix suggestions
- `/security-audit` - Vulnerability scanning
- `/test-coverage` - Coverage analysis and prioritization
- `/write-tests` - Generate comprehensive test files
- `/refactor` - Code smell detection and restructuring
- `/document-code` - Generate documentation and docstrings
- `/explain-codebase` - Full project overview for onboarding
- `/setup-guide` - Dev environment setup and verification
- `/create-pr` - Pull request preparation and submission
- `/changelog` - Release notes generation
- `/create-ticket` - Structured ticket creation

## Available Skills — TPM & Project Management
- `/estimate` - Effort estimation with breakdown and sprint fit
- `/tech-decision` - Evaluate technical proposals and options
- `/impact-analysis` - Map blast radius of proposed changes
- `/scope-check` - Detect hidden complexity and scope creep
- `/build-vs-buy` - Build in-house vs adopt vendor analysis
- `/incident-report` - Blameless postmortem generation
- `/tech-debt-report` - Quantify and prioritize tech debt
- `/pentest-report` - OWASP-based penetration testing report
- `/risk-register` - Living project risk tracking and assessment
- `/vulnerability-report` - Three-tier dependency CVE scan reports
```

---

## ❓ FAQ

**Q: Do I need all 23 skills?**
No. Start with the 3-4 you'd use most. For dev teams, `/code-review`, `/write-tests`, `/create-pr`, and `/security-audit` give the most immediate value. For TPMs, start with `/estimate`, `/scope-check`, and `/tech-decision`. For security, start with `/vulnerability-report` and `/security-audit`.

**Q: What's the `project-decisions/` folder?**
The TPM and security skills (14-23) save their output as markdown files in `project-decisions/` at your project root. Over time this becomes a searchable history of every technical decision, estimate, incident, risk assessment, and vulnerability scan your team has made. Commit it to Git so the whole team has access.

**Q: The vulnerability report generates three files — why?**
Each severity tier has a different audience and action timeline. The High report goes to security leads and needs fixing within 48 hours. The Medium report is for sprint planning. The Low report is for batching with regular dependency updates. Splitting them prevents alert fatigue and lets each audience focus on what matters to them.

**Q: Will these slow down Claude Code?**
Skill descriptions are loaded into context so Claude knows what's available. If you have many skills, they may use context budget. Run `/context` to check. You can set `SLASH_COMMAND_TOOL_CHAR_BUDGET` to adjust the limit.

**Q: Can Claude auto-invoke these without me typing the command?**
Yes — Claude reads the `description` field in each SKILL.md and can auto-invoke when your task matches. For example, saying "is this code secure?" may trigger `/security-audit`. Auto-invocation works best when descriptions are specific.

**Q: Do these work outside Claude Code?**
Skills follow the Agent Skills open standard. They work in Claude Code, Claude.ai, and Claude Desktop.

**Q: Can I share skills across multiple repos?**
Yes. Either install them globally (`~/.claude/skills/`) or package them as a Claude Code plugin that team members install.

**Q: How do I update a skill?**
Edit the `SKILL.md` file directly. If installed via Git in your project, commit and push — your team gets the update automatically.

**Q: Can I combine skills into workflows?**
Yes. Create `.claude/commands/*.md` files that reference multiple skills. See the [Workflow Recipes](#-workflow-recipes) section.

---

## 🤝 Contributing

To add or improve a skill:

1. Create a new directory in `.claude/skills/[skill-name]/`
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`) and markdown instructions
3. Test the skill in Claude Code
4. Submit a PR with examples of the skill in action

### Skill Quality Checklist

- [ ] Clear, specific `description` for auto-invocation
- [ ] Structured process with numbered steps
- [ ] Stack-specific sections where relevant
- [ ] Before/after code examples
- [ ] Output format defined with severity levels
- [ ] Summary section at the end
- [ ] Adaptation rules for different project sizes

---

## 📁 Directory Structure

```
.claude/skills/
│
│── Development Skills
├── explain-code/
│   └── SKILL.md
├── code-review/
│   └── SKILL.md
├── optimize/
│   └── SKILL.md
├── security-audit/
│   └── SKILL.md
├── test-coverage/
│   └── SKILL.md
├── write-tests/
│   └── SKILL.md
├── refactor/
│   └── SKILL.md
├── document-code/
│   └── SKILL.md
├── explain-codebase/
│   └── SKILL.md
├── setup-guide/
│   └── SKILL.md
├── create-pr/
│   └── SKILL.md
├── changelog/
│   └── SKILL.md
├── create-ticket/
│   └── SKILL.md
│
│── TPM & Project Management Skills
├── estimate/
│   └── SKILL.md
├── tech-decision/
│   └── SKILL.md
├── impact-analysis/
│   └── SKILL.md
├── scope-check/
│   └── SKILL.md
├── build-vs-buy/
│   └── SKILL.md
├── incident-report/
│   └── SKILL.md
├── tech-debt-report/
│   └── SKILL.md
├── pentest-report/
│   └── SKILL.md
├── risk-register/
│   └── SKILL.md
└── vulnerability-report/
    └── SKILL.md

project-decisions/          ← Auto-generated by TPM & security skills
├── README.md               ← Auto-updated index of all decisions
├── 2026-02-19-bigquery-vs-looker.md
├── 2026-02-19-scope-notifications.md
├── 2026-02-19-incident-payment-outage.md
├── 2026-02-19-risk-register.md
├── 2026-02-19-vulnerabilities-high.md
├── 2026-02-19-vulnerabilities-medium.md
├── 2026-02-19-vulnerabilities-low.md
└── ...
```

---

## 📄 License

MIT

---

**Built with ❤️ for dev teams and TPMs who want to ship better code, faster — and make smarter, safer decisions along the way.**