---
name: scope-check
description: Analyzes feature specs, tickets, PRDs, and proposals for hidden complexity, scope creep risks, missing requirements, ambiguous language, unstated assumptions, and unrealistic timelines. Produces a structured scope assessment with flags and recommendations. Saves output to project-decisions/ folder. Use when the user says "scope check", "is this bigger than it sounds?", "check this spec", "review this ticket", "hidden complexity", "scope creep", "what are we missing", "sanity check this", "is this realistic?", "review this PRD", or "what could go wrong with this plan?".
allowed-tools: Read, Grep, Glob, Bash
---

# Scope Check Skill

When checking scope, follow this structured process. The goal is to catch the things that make a "2-week project" turn into a "3-month nightmare" — before the team commits.

**IMPORTANT**: Always save the output as a markdown file in the `project-decisions/` directory at the project root. Create the directory if it doesn't exist.

## 0. Output Setup
```bash
# Create project-decisions directory if it doesn't exist
mkdir -p project-decisions

# File will be saved as:
# project-decisions/YYYY-MM-DD-scope-[kebab-case-topic].md
```

## 1. Parse the Spec / Proposal

### Extract Key Information

From the spec, ticket, PRD, or conversation, identify:

- **What's being built?** — the core feature or change
- **Who is it for?** — target users/personas
- **What's the stated timeline?** — any deadlines or sprint targets
- **What's the stated estimate?** — story points, days, t-shirt size
- **Who wrote this?** — product, engineering, design, stakeholder?
- **What stage is this?** — idea, spec, approved, in-progress?

### Scan the Codebase for Context
```bash
# Check how the current system works in the affected area
grep -rn "[feature-keyword]" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules\|\.git\|test\|spec" | head -30

# Check existing models/schemas in the area
grep -rn "interface\|type\|class\|model\|schema" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -i "[keyword]"

# Check existing API endpoints in the area
grep -rn "router\.\|app\.\(get\|post\|put\|delete\)\|@app\.route\|@GetMapping" --include="*.ts" --include="*.js" --include="*.py" --include="*.java" src/ 2>/dev/null | grep -i "[keyword]"

# Check existing tests in the area
find . -name "*.test.*" -o -name "*.spec.*" -o -name "test_*" | xargs grep -l "[keyword]" 2>/dev/null

# Check for similar past features (how long did they take?)
git log --oneline --all --grep="[keyword]" | head -10

# Check for related TODOs/FIXMEs
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -i "[keyword]"

# Check for tech debt in the area
wc -l $(grep -rln "[keyword]" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | head -10) 2>/dev/null
```

## 2. Scope Creep Red Flags

Scan the spec for language and patterns that signal hidden complexity:

### 🔴 Critical Red Flags — Almost Certainly Bigger Than It Sounds

#### Vague Requirements
```
Flag these phrases:
- "Simple" / "Just" / "Quickly" / "Easy"
  → The word "just" in a spec is the #1 predictor of scope creep
  → "Just add a button that..." hides all the backend logic

- "Similar to [complex product]"
  → "Like Slack but for X" is 5 years of engineering
  → Always ask: which specific feature of that product?

- "And everything that goes with it"
  → Unbounded scope — what does "everything" mean?

- "Should work like it does now, but also..."
  → Backward compatibility + new behavior = double the work

- "Users should be able to..."  (without specifying which users)
  → All users? Admins? Free tier? Paid? Each is different scope

- "Support for [open-ended list]"
  → "Support for multiple payment providers" = how many? Which ones? Each is a separate integration
```

#### Missing Edge Cases
```
Flag when the spec doesn't mention:
- What happens on error?
- What happens with empty/null data?
- What happens with very large data sets?
- What happens when the user is offline?
- What happens on timeout?
- What happens concurrently?
- What happens when the user goes back/refreshes?
- What about existing data? (migration needed?)
- What about internationalization?
- What about accessibility?
- What about mobile/responsive?
```

#### Hidden "Icebergs"
```
The spec says:              But it actually requires:
─────────────────────────   ────────────────────────────────────
"Add search"                Full-text search engine, indexing,
                            ranking, filters, pagination, 
                            highlighting, typo tolerance

"Add notifications"         Notification service, delivery 
                            channels (email, SMS, push),
                            preferences, templates, scheduling,
                            unsubscribe, rate limiting

"Add user roles"            Permission system, role management UI,
                            migration of existing users, audit 
                            trail, testing every endpoint

"Add file upload"           Storage service, virus scanning,
                            file type validation, size limits,
                            thumbnails, CDN, cleanup/deletion

"Add payments"              Payment provider integration, PCI
                            compliance, webhooks, retry logic,
                            refunds, invoicing, tax calculation

"Add real-time updates"     WebSocket infrastructure, connection
                            management, reconnection, state sync,
                            horizontal scaling with pub/sub

"Add an admin panel"        CRUD for every entity, audit logging,
                            role-based access, bulk operations,
                            export, dashboard, search/filter

"Add multi-tenancy"         Data isolation, tenant-aware queries
                            everywhere, tenant management, billing
                            per tenant, cross-tenant prevention

"Add SSO/SAML"              SAML/OIDC integration, identity 
                            provider config UI, JIT provisioning,
                            session management, logout propagation

"Make it work offline"      Local database, sync engine, conflict
                            resolution, queue for pending actions,
                            retry logic, cache invalidation
```

### 🟡 Warning Flags — Likely Bigger Than Estimated

#### Underspecified Behavior
```
Flag when the spec says:
- "The usual flow" → Which flow? Document it
- "Handle errors appropriately" → What specifically? Show user? Retry? Log?
- "Responsive design" → Mobile? Tablet? Which breakpoints? Different layouts?
- "Good performance" → What's the target? <200ms? <1s? Under what load?
- "Secure" → Against what? XSS? CSRF? Injection? Auth bypass?
- "Nice UI" → Where are the mocks? No mocks = design work not included
- "Integration with X" → Which API? Which version? Is there documentation?
- "Export functionality" → What format? CSV? PDF? Excel? All of them?
- "Audit trail" → What events? How long to retain? Who can view?
- "Configurable" → By whom? Admin UI? Config file? Environment variable?
```

#### Cross-Cutting Concerns Not Mentioned
```
Flag if the spec doesn't address:
- Authentication → Who can access this?
- Authorization → What permissions are needed?
- Logging → What should be logged for debugging?
- Monitoring → What alerts should fire if this fails?
- Rate limiting → Can this be abused?
- Caching → Does this need caching for performance?
- Data validation → What input validation is needed?
- Error handling → What's the error UX?
- Migration → What happens to existing data/users?
- Documentation → API docs? User docs? Internal docs?
- Testing strategy → Unit? Integration? E2E? Manual?
- Deployment plan → Feature flag? Gradual rollout?
- Rollback plan → What if this needs to be reverted?
```

#### Dependencies Not Called Out
```
Flag when the feature requires:
- Design work that hasn't started
- API from another team that doesn't exist yet
- Third-party service evaluation or contract
- Infrastructure provisioning (new database, queue, etc.)
- Data migration or backfill
- Legal/compliance review
- External partner coordination
- New environment variables or secrets
- CI/CD pipeline changes
```

### 🟢 Minor Flags — Worth Noting
```
- Copy/text not finalized (changes = rework)
- No success metrics defined (how do we know it's working?)
- No acceptance criteria (how do we know it's done?)
- No out-of-scope section (invitation for scope creep)
- No priority ranking of requirements (everything feels P0)
- No user research backing the feature (building the wrong thing risk)
```

## 3. Complexity Analysis

### Decompose Into Work Streams

Break the spec into concrete work streams and flag complexity:
```
| Work Stream | Stated Scope | Hidden Scope | Complexity |
|-------------|-------------|-------------|------------|
| Backend API | 2 new endpoints | + validation, error handling, rate limiting, auth checks, logging | Medium → Large |
| Database | 1 new table | + migration, indexes, existing data backfill, relationships to 3 other tables | Small → Medium |
| Frontend UI | 1 new page | + form validation, loading states, error states, empty states, mobile responsive, accessibility | Medium → Large |
| Integration | Connect to Stripe | + webhook handling, retry logic, idempotency, test mode, error mapping, PCI compliance review | Medium → Very Large |
| Testing | "Add tests" | + unit, integration, e2e, mocking external services, test data setup | Medium |
| Deployment | Not mentioned | Feature flag, migration script, env vars, monitoring dashboard, rollback plan | Small → Medium |
| Documentation | Not mentioned | API docs, README update, runbook, user guide | Small |
```

### Effort Multipliers

Apply multipliers based on what's missing from the spec:
```
Base estimate from spec:                    5 days

Multipliers:
× 1.0   Clear, detailed spec with mocks         (rare)
× 1.3   Spec exists but missing edge cases       (common)
× 1.5   Vague spec, "you know what I mean"       (common)
× 2.0   One-liner ticket, no spec at all         (too common)
× 1.2   Team has done similar work before
× 1.5   Team has partial experience
× 2.0   Completely new territory for the team
× 1.0   No external dependencies
× 1.3   One external dependency
× 1.5   Multiple external dependencies
× 2.0   External dependency with no documentation
× 1.0   No database changes
× 1.3   Additive database changes
× 1.5   Database changes with migration
× 2.0   Database changes with large data backfill
× 1.0   No breaking changes
× 1.5   Breaking changes with known consumers
× 2.0   Breaking changes with unknown consumers

Adjusted estimate: base × (product of applicable multipliers)
```

### Complexity Score

Rate the overall complexity:
```
| Factor | Score (1-5) | Notes |
|--------|-------------|-------|
| Requirements clarity | X | [how clear is the spec?] |
| Technical complexity | X | [how hard is the implementation?] |
| Integration complexity | X | [how many external touchpoints?] |
| Data complexity | X | [schema changes, migrations, data quality?] |
| UX complexity | X | [how many states, flows, edge cases?] |
| Testing complexity | X | [how hard to verify?] |
| Deployment complexity | X | [special rollout needs?] |
| Dependency risk | X | [external blockers?] |
| Team familiarity | X | [has team done this before?] |
| Stakeholder alignment | X | [is everyone on the same page?] |

Average: X.X / 5

1.0-2.0: Straightforward — estimate is probably right
2.1-3.0: Moderate — add 30% buffer
3.1-4.0: Complex — add 50-100% buffer, consider decomposition
4.1-5.0: Very Complex — needs spike, decomposition, or scope reduction before committing
```

## 4. Assumption Audit

### Stated Assumptions
List assumptions explicitly stated in the spec:
```
1. [Assumption from the spec]
2. [Assumption from the spec]
```

### Unstated Assumptions
List assumptions the spec DOESN'T state but relies on:
```
1. [Assumed X is already in place — is it?]
2. [Assumed team has capacity — do they?]
3. [Assumed API is stable — is it?]
4. [Assumed design is finalized — is it?]
5. [Assumed data is clean — is it?]
6. [Assumed no breaking changes — really?]
7. [Assumed same behavior on mobile — confirmed?]
8. [Assumed existing auth covers this — does it?]
```

### Assumption Risk
```
| Assumption | If Wrong, Impact | Likelihood Wrong | Action |
|-----------|-----------------|------------------|--------|
| [Assumption 1] | +X days | High/Med/Low | [Validate how] |
| [Assumption 2] | +X days | High/Med/Low | [Validate how] |
| [Assumption 3] | Blocker | High/Med/Low | [Validate how] |
```

## 5. Missing Requirements Checklist

### Functional Requirements
- [ ] Happy path clearly defined
- [ ] Error paths defined (what happens when things fail)
- [ ] Edge cases addressed (empty, null, max, concurrent)
- [ ] User permissions defined (who can do what)
- [ ] Data validation rules specified
- [ ] Existing data migration plan (if applicable)
- [ ] Backward compatibility addressed (if changing existing behavior)
- [ ] Multi-device behavior defined (desktop, mobile, tablet)
- [ ] Offline behavior defined (if applicable)
- [ ] Internationalization requirements (languages, RTL, date formats)

### Non-Functional Requirements
- [ ] Performance targets (response time, throughput)
- [ ] Scalability requirements (concurrent users, data volume)
- [ ] Availability requirements (uptime SLA)
- [ ] Security requirements (auth, encryption, compliance)
- [ ] Accessibility requirements (WCAG level)
- [ ] Browser/device support matrix
- [ ] Data retention/deletion policy
- [ ] Audit/compliance logging

### Design & UX
- [ ] Design mocks provided and approved
- [ ] All UI states covered (loading, empty, error, success)
- [ ] Mobile/responsive design specified
- [ ] Accessibility reviewed
- [ ] Copy/text finalized
- [ ] Animation/transition behavior defined
- [ ] Dark mode (if applicable)

### Operational
- [ ] Monitoring and alerting plan
- [ ] Logging requirements
- [ ] Feature flag strategy
- [ ] Rollout plan (% rollout, canary, blue-green)
- [ ] Rollback procedure
- [ ] Runbook for on-call
- [ ] Support team briefed

### Delivery
- [ ] Acceptance criteria defined and testable
- [ ] Testing strategy defined (unit, integration, e2e, manual)
- [ ] Definition of done agreed upon
- [ ] Out of scope explicitly stated
- [ ] Dependencies on other teams identified
- [ ] Success metrics defined
- [ ] Launch criteria defined
- [ ] Post-launch monitoring plan

## 6. Scope Reduction Recommendations

If the scope is too large, suggest how to cut it down:

### MoSCoW Prioritization
```
MUST HAVE (P0) — Ship doesn't sail without these
- [Requirement 1] — [why it's essential]
- [Requirement 2] — [why it's essential]

SHOULD HAVE (P1) — Important but can ship without
- [Requirement 3] — [can follow in v1.1]
- [Requirement 4] — [can follow in v1.1]

COULD HAVE (P2) — Nice to have
- [Requirement 5] — [future enhancement]
- [Requirement 6] — [future enhancement]

WON'T HAVE (this time) — Explicitly out of scope
- [Requirement 7] — [why it's cut, when to revisit]
- [Requirement 8] — [why it's cut, when to revisit]
```

### Phased Delivery
```
Phase 1: MVP (Sprint 1-2)
- [Core functionality only]
- [Happy path only]
- [Single use case]
- Estimated: X days

Phase 2: Hardening (Sprint 3)
- [Error handling]
- [Edge cases]
- [Performance optimization]
- Estimated: X days

Phase 3: Polish (Sprint 4)
- [Additional use cases]
- [Nice-to-have features]
- [Advanced features]
- Estimated: X days
```

### Cut Suggestions
```
| Requirement | Cut? | Savings | Trade-off |
|-------------|------|---------|-----------|
| [Feature A] | Defer to Phase 2 | 3 days | Users use workaround for now |
| [Feature B] | Simplify | 2 days | Manual instead of automated |
| [Feature C] | Remove | 5 days | Not needed for MVP validation |
| [Feature D] | Keep | 0 | Core requirement, can't cut |
```

## 7. Questions to Ask Before Committing

Generate a list of questions that should be answered before the team commits to this scope:
```
## Questions to Resolve

### Requirements Questions
1. [Specific question about unclear requirement]
2. [Specific question about missing behavior]
3. [Specific question about edge case]

### Technical Questions
4. [Question about feasibility]
5. [Question about integration]
6. [Question about data migration]

### Design Questions
7. [Question about missing mock/flow]
8. [Question about mobile behavior]
9. [Question about error states]

### Business Questions
10. [Question about priority]
11. [Question about timeline flexibility]
12. [Question about success metrics]

### Dependency Questions
13. [Question about external team/service]
14. [Question about blocked work]
```

## Output Document Template

Save to `project-decisions/YYYY-MM-DD-scope-[topic].md`:
```markdown
# Scope Check: [Feature/Ticket Title]

**Date:** YYYY-MM-DD
**Spec reviewed:** [Link to spec/ticket/PRD]
**Reviewed by:** [Name]
**Stated estimate:** [Original estimate if any]
**Adjusted estimate:** [After scope check]

---

## Verdict

**Scope assessment: [🟢 Realistic / 🟡 Needs Refinement / 🟠 Significantly Underscoped / 🔴 Unrealistic]**

**Key finding:** [One sentence summary — e.g., "The spec covers the happy path well but misses 
authentication, error handling, and mobile — which roughly doubles the effort."]

---

## Red Flags Found

### 🔴 Critical (will definitely increase scope)
1. [Red flag with explanation and impact]
2. [Red flag with explanation and impact]

### 🟡 Warning (likely to increase scope)
3. [Warning with explanation and impact]
4. [Warning with explanation and impact]

### 🟢 Minor (worth noting)
5. [Minor flag]
6. [Minor flag]

## Hidden Complexity

| Area | What the Spec Says | What It Actually Requires | Extra Effort |
|------|-------------------|--------------------------|-------------|
| [Area 1] | [Stated scope] | [Real scope] | +X days |
| [Area 2] | [Stated scope] | [Real scope] | +X days |

## Missing Requirements

[List of requirements not in the spec but needed for a complete implementation]

## Unstated Assumptions

[List of assumptions the spec relies on that haven't been validated]

## Questions to Resolve

[Numbered list of questions that must be answered before committing]

## Estimate Comparison

| | Stated | Adjusted | Notes |
|--|--------|----------|-------|
| **Total effort** | X days | Y days | [why it's different] |
| **Complexity** | [Low/Med/High] | [Low/Med/High] | [what changed] |
| **Confidence** | — | [High/Med/Low] | [why this confidence level] |
| **Sprint fit** | [Y sprints] | [Z sprints] | [with buffer] |

## Scope Reduction Recommendations

### If we need to fit the original timeline:
[What to cut or defer]

### Phased delivery suggestion:
[Phase 1 MVP → Phase 2 Hardening → Phase 3 Polish]

## Recommendation

**[Commit as-is / Refine spec first / Reduce scope / Spike first / Reject timeline]**

[Reasoning — 2-3 sentences]

## Next Steps

1. [ ] [Action item — e.g., "Answer questions 1-5 before sprint planning"]
2. [ ] [Action item — e.g., "Get design mocks for mobile states"]
3. [ ] [Action item — e.g., "Validate assumption about API availability"]

---

## Decision Log

| Date | Event | By |
|------|-------|----|
| YYYY-MM-DD | Scope check requested | [Name] |
| YYYY-MM-DD | Analysis completed | [Name] |
| YYYY-MM-DD | Questions resolved | [Name] |
| YYYY-MM-DD | Scope approved / refined | [Name] |
```

After saving, update the project-decisions index:
```bash
# Update README.md index
echo "# Project Decisions\n" > project-decisions/README.md
echo "| Date | Decision | Type | Status |" >> project-decisions/README.md
echo "|------|----------|------|--------|" >> project-decisions/README.md

for f in project-decisions/2*.md; do
  date=$(basename "$f" | cut -d'-' -f1-3)
  title=$(head -1 "$f" | sed 's/^# //' | sed 's/^Scope Check: //' | sed 's/^Impact Analysis: //' | sed 's/^Tech Decision: //')
  type="Other"
  echo "$f" | grep -q "scope" && type="Scope Check"
  echo "$f" | grep -q "impact" && type="Impact Analysis"
  echo "$f" | grep -q "tech-decision\|decision" && type="Tech Decision"
  status=$(grep "^**Status:**\|^**Scope assessment:**\|^**Proceed:**" "$f" | head -1 | sed 's/.*: //' | sed 's/\*//g')
  echo "| $date | [$title](./$(basename $f)) | $type | $status |" >> project-decisions/README.md
done
```

## Adaptation Rules

- **Always save to file** — every scope check gets persisted in `project-decisions/`
- **Read the actual codebase** — don't just analyze the spec in isolation, check what exists
- **Flag specific lines** — "line 12 says 'simple search' but the codebase has no search infrastructure" is better than "search might be complex"
- **Quantify impact** — "+3 days for error handling" not "error handling will add time"
- **Be constructive** — don't just say "this is underscoped", suggest how to fix it
- **Offer cut options** — always provide scope reduction recommendations
- **Ask questions** — generate specific questions, not vague "needs more detail"
- **Compare to past work** — check git history for similar features and how long they took
- **Consider the team** — same scope is different effort for different teams
- **Respect the timeline** — if the deadline is fixed, focus on what to cut, not what to add

## Summary

End every scope check with:
1. **Verdict** — 🟢 Realistic / 🟡 Needs Refinement / 🟠 Underscoped / 🔴 Unrealistic
2. **Red flag count** — X critical, Y warning, Z minor
3. **Estimate adjustment** — stated vs adjusted estimate
4. **Top 3 risks** — biggest scope creep threats
5. **Questions count** — X questions must be resolved before committing
6. **Recommendation** — commit / refine / reduce / spike / reject
7. **File saved** — confirm the document location
