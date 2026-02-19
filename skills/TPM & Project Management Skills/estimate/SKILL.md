---
name: estimate
description: Estimates effort for tasks and tickets by analyzing code complexity, dependencies, scope, risk, and historical patterns. Use when the user says "how long will this take?", "estimate this", "how much effort?", "size this ticket", "story points", "can we do this in a sprint?", "level of effort", or "is this a big change?".
allowed-tools: Read, Grep, Glob, Bash
---

# Estimate Skill

When estimating work, follow this structured process. Good estimates are ranges, not single numbers. They account for complexity, risk, unknowns, and the reality that things always take longer than expected.

## 1. Understand What's Being Estimated

Before estimating, clarify the scope:

### Gather Requirements
- **What exactly needs to be built/changed?** — Get specific, not vague
- **What's the expected outcome?** — How will we know it's done?
- **What's explicitly out of scope?** — Prevent scope creep
- **Who is this for?** — End users, internal team, API consumers?
- **Are there design mocks?** — UI work without mocks adds uncertainty
- **Are there dependencies on other teams/services?** — External dependencies add risk

### Analyze the Codebase
```bash
# Find the files that would need to change
grep -rn "[relevant-keyword]" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | head -20

# Check complexity of affected area
wc -l [target-files]

# Check how many files import/depend on the affected code
grep -rn "import.*from.*[module]" --include="*.ts" --include="*.js" src/ 2>/dev/null | wc -l

# Check test coverage of affected area
find . -name "*[module]*test*" -o -name "*[module]*spec*" 2>/dev/null

# Check recent change frequency (hot spots = more risk)
git log --oneline -20 -- [target-path]

# Check how many people have worked on this area
git log --format='%aN' -- [target-path] | sort | uniq -c | sort -rn

# Check for existing TODOs/FIXMEs in the area
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.ts" --include="*.js" --include="*.py" [target-path] 2>/dev/null
```

## 2. Complexity Analysis

### Break Down the Work

Decompose every task into concrete subtasks:
```
Example: "Add OAuth2 login with Google"

Subtasks:
1. Research Google OAuth2 API and decide on library          [spike]
2. Create OAuth config and environment variables              [small]
3. Implement OAuth callback route and token exchange          [medium]
4. Create/update user record from OAuth profile               [medium]
5. Handle account linking (OAuth + existing email account)    [large]
6. Add OAuth login button to frontend                         [small]
7. Handle OAuth error states and edge cases                   [medium]
8. Write unit tests for OAuth service                         [medium]
9. Write integration tests for OAuth flow                     [medium]
10. Update documentation and environment setup guide          [small]
11. Test in staging with real Google credentials              [small]
12. Security review of token handling                         [small]
```

### Complexity Factors

Rate each factor on a scale of 1-5:

| Factor | 1 (Low) | 3 (Medium) | 5 (High) |
|--------|---------|------------|----------|
| **Code changes** | Single file, < 50 lines | 3-5 files, ~200 lines | 10+ files, 500+ lines |
| **Logic complexity** | Simple CRUD, straightforward | Conditional logic, state management | Complex algorithms, concurrency |
| **Dependencies** | No external dependencies | Uses existing libraries/APIs | New integrations, cross-team coordination |
| **Data changes** | No schema changes | New columns/indexes | New tables, data migration, backfill |
| **Testing effort** | Easy to test, few cases | Multiple scenarios, some mocking | Complex setup, E2E, hard to reproduce states |
| **Risk** | Well-understood area, safe change | Some unknowns, moderate impact | Touching critical path, payment/auth/data |
| **Frontend work** | No UI changes | Minor UI updates | New pages/flows, responsive, accessibility |
| **Domain knowledge** | Team has done similar work | Some research needed | New territory for the team |
| **Review/approval** | Standard PR review | Security review needed | Architecture review, stakeholder sign-off |
| **Deployment** | Normal deploy, no special steps | Feature flag, migration | Coordinated deploy, downtime window |

**Complexity Score** = Average of all factors
- **1.0 - 2.0** → Simple
- **2.1 - 3.0** → Medium
- **3.1 - 4.0** → Complex
- **4.1 - 5.0** → Very Complex

## 3. Estimation Methods

### Method A: T-Shirt Sizing

Quick, relative sizing — best for backlog grooming and sprint planning:

| Size | Description | Typical Duration | Story Points |
|------|-------------|-----------------|-------------|
| **XS** | Config change, copy update, one-liner fix | < 2 hours | 1 |
| **S** | Simple bug fix, small feature, well-defined scope | 2-4 hours | 2 |
| **M** | Feature with a few moving parts, some unknowns | 1-2 days | 3-5 |
| **L** | Multi-component feature, new integration, DB changes | 3-5 days | 8 |
| **XL** | Large feature spanning multiple systems, significant unknowns | 1-2 weeks | 13 |
| **XXL** | Epic-level work, needs decomposition before estimating | 2+ weeks | 21+ |

**Rule**: If it's XL or larger, break it into smaller tickets before estimating.

### Method B: Time Range Estimate

Provide optimistic, likely, and pessimistic estimates:
```
Estimate: Add OAuth2 Login with Google

Best case (everything goes smoothly):     3 days
Most likely (normal development pace):    5 days
Worst case (unexpected complications):    8 days

Recommended estimate:                     5 days
Buffer for unknowns (20%):               +1 day
Total with buffer:                        6 days
```

**Formula**: `Expected = (Best + 4×Likely + Worst) / 6`

This is the PERT estimation technique — it weights the most likely scenario while accounting for extremes.

### Method C: Story Points (Fibonacci)

Relative sizing using Fibonacci sequence: 1, 2, 3, 5, 8, 13, 21
```
Reference stories (calibrate with team):

1 point  → Fix a typo in the UI
2 points → Add a new field to an existing form
3 points → Create a new API endpoint with validation
5 points → Implement a new feature with frontend + backend + tests
8 points → New integration with external service + error handling
13 points → Large feature with DB schema changes + migration + multiple APIs
21 points → Epic-level, needs decomposition
```

**Rule**: If you can't agree on 5 vs 8, go with 8. Estimates should err on the side of caution.

### Method D: Task-Based Estimate

Most accurate for known work — sum up individual task estimates:
```
Task Breakdown: Add OAuth2 Login with Google

| # | Task | Estimate | Confidence |
|---|------|----------|-----------|
| 1 | Research & library selection | 2h | High |
| 2 | OAuth config & env vars | 1h | High |
| 3 | OAuth callback route | 3h | Medium |
| 4 | User record creation/linking | 4h | Medium |
| 5 | Account linking edge cases | 4h | Low |
| 6 | Frontend login button | 2h | High |
| 7 | Error handling | 3h | Medium |
| 8 | Unit tests | 3h | High |
| 9 | Integration tests | 4h | Medium |
| 10 | Documentation | 1h | High |
| 11 | Staging testing | 2h | Medium |
| 12 | Security review | 1h | High |
|---|------|----------|-----------|
| | **Subtotal** | **30h** | |
| | **Buffer (25% for medium/low confidence)** | **+8h** | |
| | **Total** | **38h (~5 days)** | |
```

## 4. Risk Assessment

### Identify Risks That Inflate Estimates

| Risk | Impact on Estimate | Mitigation |
|------|-------------------|------------|
| **First time doing this** | +50-100% | Do a time-boxed spike first |
| **Unclear requirements** | +30-50% | Clarify before estimating |
| **External API dependency** | +25-50% | Build with mock first, integrate later |
| **Cross-team coordination** | +25-50% | Align schedules early, identify blockers |
| **Database migration on large table** | +25% | Test migration time on production-size data |
| **Touching critical path** (auth, payments) | +25% | Extra testing, staged rollout |
| **No existing tests** | +30% | Write characterization tests first |
| **Legacy code / tech debt** | +25-50% | Budget time for understanding + cleanup |
| **Designer not available** | +25% | Use existing patterns, get async feedback |
| **New technology/library** | +30-50% | Prototype first, add learning time |
| **Regulatory/compliance requirements** | +25% | Involve compliance team early |
| **Multiple environments to consider** | +20% | Test matrix for browsers/devices |

### Risk-Adjusted Estimate
```
Base estimate:                          5 days
Risk: First time using OAuth2           +2 days (40%)
Risk: Account linking complexity        +1 day (20%)
Risk: No existing auth tests            +1 day (20%)

Risk-adjusted estimate:                 9 days
Recommended buffer:                     +1 day (10%)

Final estimate:                         10 days (2 weeks)
```

## 5. Confidence Levels

Always communicate confidence alongside the estimate:

| Confidence | When | Accuracy |
|-----------|------|----------|
| **High (±10%)** | Well-understood work, team has done similar, clear requirements | Estimate is very reliable |
| **Medium (±25%)** | Some unknowns, mostly understood, minor research needed | Estimate could vary |
| **Low (±50%)** | Significant unknowns, new technology, unclear requirements | Needs a spike before accurate estimate |
| **Very Low (±100%)** | Completely new territory, major unknowns | Don't estimate — do a spike first |
```
Example output:

Estimate: 5 days
Confidence: Medium (±25%)
Range: 4-7 days
Recommendation: Start with a 2-hour spike on OAuth library selection
                to increase confidence to High before committing to sprint
```

## 6. Historical Analysis

Use past work to calibrate estimates:
```bash
# How long did similar PRs take? (time between first commit and merge)
git log --merges --oneline --grep="auth\|login\|oauth" | head -10

# Average PR size for similar features
gh pr list --state merged --search "auth OR login" --json additions,deletions,createdAt,mergedAt --limit 10 2>/dev/null

# How long did the last similar feature take?
git log --oneline --after="2025-01-01" --grep="feat" | head -20

# Cycle time for recent PRs
gh pr list --state merged --limit 20 --json number,title,createdAt,mergedAt 2>/dev/null
```

### Compare with Similar Past Work
```
Similar completed work:

| Feature | Estimated | Actual | Ratio |
|---------|-----------|--------|-------|
| Password reset flow | 3 days | 4 days | 1.33x |
| Stripe integration | 5 days | 8 days | 1.60x |
| User profile page | 2 days | 2 days | 1.00x |
| Email notifications | 3 days | 5 days | 1.67x |

Average ratio: 1.40x (team typically takes 40% longer than estimated)

Applying to current estimate:
Raw estimate: 5 days
Historical adjustment (×1.40): 7 days
```

## 7. Sprint Fit Analysis

Determine if the work fits in the current sprint:
```
Sprint capacity analysis:

Sprint duration:                    10 days
Team members available:             4
Days off/meetings/overhead:         -2 days per person (20%)
Effective capacity per person:      8 days
Total team capacity:                32 person-days

Already committed:                  20 person-days
Remaining capacity:                 12 person-days

This task estimate:                 7 person-days
Buffer:                            +2 days

Verdict: ✅ Fits in current sprint (9 of 12 remaining days)
         Leaves 3 days buffer for other work/surprises
```

### Sprint Fit Rules
```
✅ FITS       → Task estimate ≤ 60% of remaining capacity
⚠️ TIGHT     → Task estimate is 60-80% of remaining capacity
❌ DOESN'T FIT → Task estimate > 80% of remaining capacity
🔪 SPLIT IT   → Task estimate > sprint capacity (needs decomposition)
```

## 8. Decomposition Recommendations

When a task is too large, suggest how to break it down:

### Splitting Strategies

**By Layer (Vertical Slice)**
```
Original: "Add user search feature"

Split into:
Sprint 1: Basic search API + simple UI (MVP)
Sprint 2: Advanced filters + pagination
Sprint 3: Search suggestions + analytics
```

**By Functionality (Horizontal Slice)**
```
Original: "Implement checkout flow"

Split into:
Ticket 1: Cart summary page (frontend only, mock data)
Ticket 2: Payment API integration (backend only)
Ticket 3: Connect frontend to backend
Ticket 4: Order confirmation + email notification
Ticket 5: Error handling + retry logic
```

**By Risk (Spike First)**
```
Original: "Migrate from Mailgun to SendGrid"

Split into:
Ticket 1: [Spike] Evaluate SendGrid API, test sending (time-boxed: 4h)
Ticket 2: Implement SendGrid wrapper matching current interface
Ticket 3: Update email templates for SendGrid format
Ticket 4: Migration + testing in staging
Ticket 5: Production cutover + monitoring
```

**By User Story**
```
Original: "OAuth login"

Split into:
Story 1: As a user, I can sign in with Google
Story 2: As a user, I can sign in with GitHub
Story 3: As an existing user, I can link my OAuth account
Story 4: As a user, I can unlink my OAuth account
```

## 9. Communication Templates

### For Sprint Planning
```
📋 Estimate: [Task Title]

Size: M (Medium)
Points: 5
Time estimate: 3-5 days (most likely 4)
Confidence: Medium (±25%)

Breakdown:
- Backend API changes — 1.5 days
- Frontend updates — 1 day
- Tests — 1 day
- Review + fixes — 0.5 days

Risks:
- Depends on design mock (not yet finalized) — could add 1-2 days
- First time using this API — may need extra research time

Dependencies:
- Needs design approval before frontend work starts
- Needs staging env access for integration testing

Sprint fit: ✅ Fits with 3 days buffer remaining
```

### For Stakeholder Communication
```
📊 Effort Estimate: [Feature Name]

Estimated delivery: 2-3 weeks
Confidence: Medium

What's included:
- [Capability 1]
- [Capability 2]
- [Capability 3]

What's NOT included (future work):
- [Out of scope item 1]
- [Out of scope item 2]

Key risks:
- [Risk 1] — mitigated by [approach]
- [Risk 2] — will know more after [milestone]

Assumptions:
- Design is finalized by [date]
- No other priority changes during this period
- [Team member] is available for the full duration
```

### For Quick Slack Estimates
```
Quick estimate for [task]:
- Size: S/M/L
- Time: X-Y days
- Confidence: High/Medium/Low
- Fits this sprint: Yes/No/Tight
- Risks: [one-liner]
```

## Output Format

Structure every estimate as:

### 1. Task Understanding
```
Task: [Clear description of what needs to be done]
Scope: [What's included and explicitly excluded]
```

### 2. Breakdown
```
| # | Subtask | Estimate | Confidence | Risk |
|---|---------|----------|------------|------|
| 1 | [task] | Xh | High/Med/Low | [risk if any] |
| ... | ... | ... | ... | ... |
```

### 3. Estimate Summary
```
Method: [T-shirt / Time Range / Story Points / Task-based]

Size: [XS/S/M/L/XL]
Story Points: [1/2/3/5/8/13/21]
Time Estimate:
  Best case:    X days
  Most likely:  Y days
  Worst case:   Z days
  
Recommended:    Y days
With buffer:    Y + buffer days
Confidence:     [High/Medium/Low] (±X%)
```

### 4. Risk Factors
```
| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| [risk] | +X days | High/Med/Low | [approach] |
```

### 5. Sprint Fit
```
Remaining sprint capacity: X days
This task (with buffer): Y days
Verdict: ✅ Fits / ⚠️ Tight / ❌ Doesn't fit / 🔪 Needs splitting
```

### 6. Recommendations
```
- [Recommendation 1 — e.g., do a spike first]
- [Recommendation 2 — e.g., split into 3 tickets]
- [Recommendation 3 — e.g., clarify requirements before committing]
```

## Adaptation Rules

- **Match the team's estimation method** — if the team uses story points, give story points; if hours, give hours
- **Use historical data** — look at past similar work to calibrate
- **Account for the individual** — senior vs junior, familiar vs unfamiliar with the area
- **Include non-coding work** — code review time, QA, documentation, deployment
- **Round up, not down** — when in doubt, estimate higher
- **Flag unknowns explicitly** — don't hide uncertainty in a single number
- **Recommend spikes** — when confidence is Low or Very Low, suggest a time-boxed investigation first
- **Consider context switching** — developers rarely get uninterrupted full days; account for meetings, Slack, reviews

## Common Estimation Mistakes to Avoid

- **Planning fallacy** — we always think things will go faster than they do
- **Anchoring** — don't let someone else's guess bias your estimate
- **Forgetting testing** — testing often takes as long as development
- **Forgetting reviews** — code review, QA, and stakeholder feedback take time
- **Ignoring deployment** — migrations, feature flags, monitoring setup
- **Happy path only** — estimate for error handling, edge cases, and rollback
- **Not including learning time** — new tools/APIs require ramp-up
- **Scope creep** — explicitly state what's NOT included

## Summary

End every estimate with:
1. **Quick answer** — size and time range in one line
2. **Confidence level** — how reliable this estimate is
3. **Biggest risk** — the single thing most likely to blow the estimate
4. **Recommendation** — should we commit, spike first, or decompose?
5. **Sprint fit** — does it fit in the current sprint?
