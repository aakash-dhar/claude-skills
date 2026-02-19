---
name: risk-register
description: Creates and maintains a living project risk register by analyzing the codebase, dependencies, team structure, timeline, and technical decisions. Identifies risks, scores them by likelihood and impact, assigns owners, tracks mitigations, and flags risks that have changed since last assessment. Saves output to project-decisions/ folder. Use when the user says "risk register", "project risks", "what could go wrong", "risk assessment", "identify risks", "update risks", "risk review", "what are our risks", or "flag risks for the project".
allowed-tools: Read, Grep, Glob, Bash
---

# Risk Register Skill

When creating or updating a risk register, follow this structured process. The goal is to maintain a living document that surfaces project risks early enough to act on them — before they become incidents, missed deadlines, or scope explosions.

**IMPORTANT**: Always save the output as a markdown file in the `project-decisions/` directory at the project root. Create the directory if it doesn't exist.

**PRINCIPLE**: A good risk register is not a one-time document. It should be reviewed and updated every sprint. Risks change — new ones appear, old ones are mitigated, some become reality.

## 0. Output Setup
```bash
mkdir -p project-decisions

# File naming:
# First time:  project-decisions/YYYY-MM-DD-risk-register.md
# Updates:     Edit the existing file, add to the changelog at the bottom
# If no existing register exists, create a new one
# If one exists, update it

ls project-decisions/*risk-register* 2>/dev/null
```

## 1. Risk Discovery

### 1a. Codebase & Technical Risks
```bash
# Complexity hotspots (high complexity = high risk of bugs)
find . -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -path '*/node_modules/*' ! -path '*/dist/*' -exec wc -l {} + 2>/dev/null | sort -rn | head -15

# Files with highest churn (most changes = most fragile)
git log --name-only --since="3 months ago" --format="" -- src/ app/ 2>/dev/null | sort | uniq -c | sort -rn | head -15

# Files with most bug fixes (where problems live)
git log --name-only --since="6 months ago" --grep="fix\|bug\|hotfix" --format="" -- src/ app/ 2>/dev/null | sort | uniq -c | sort -rn | head -10

# Dependency vulnerabilities
npm audit --json 2>/dev/null | head -50
pip audit 2>/dev/null | head -20

# Outdated dependencies
npm outdated 2>/dev/null | head -20
pip list --outdated 2>/dev/null | head -20

# TODO/FIXME/HACK count (unaddressed known issues)
echo "TODO: $(grep -rn 'TODO' --include='*.ts' --include='*.js' --include='*.py' src/ app/ 2>/dev/null | grep -v 'node_modules' | wc -l)"
echo "FIXME: $(grep -rn 'FIXME' --include='*.ts' --include='*.js' --include='*.py' src/ app/ 2>/dev/null | grep -v 'node_modules' | wc -l)"
echo "HACK: $(grep -rn 'HACK' --include='*.ts' --include='*.js' --include='*.py' src/ app/ 2>/dev/null | grep -v 'node_modules' | wc -l)"

# Test coverage gaps (untested code = risk)
find src/ app/ -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -name "*.test.*" ! -name "*.spec.*" ! -name "test_*" ! -name "*.d.ts" ! -name "index.*" ! -path '*/node_modules/*' ! -path '*/dist/*' 2>/dev/null | while read f; do
  base=$(basename "$f" | sed 's/\.\(ts\|tsx\|js\|jsx\|py\)$//')
  if ! find . \( -name "${base}.test.*" -o -name "${base}.spec.*" -o -name "test_${base}.*" \) ! -path '*/node_modules/*' 2>/dev/null | grep -q .; then
    echo "UNTESTED: $f"
  fi
done | head -20

# Single points of failure (bus factor)
for f in $(git log --name-only --since="12 months ago" --format="" -- src/ 2>/dev/null | sort -u | head -30); do
  authors=$(git log --format='%aN' --since="12 months ago" -- "$f" 2>/dev/null | sort -u | wc -l)
  if [ "$authors" -eq 1 ]; then
    echo "BUS FACTOR 1: $f ($(git log --format='%aN' -1 -- "$f" 2>/dev/null))"
  fi
done | head -15

# Missing error handling in critical paths
grep -rn "catch\|except\|rescue" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | wc -l
grep -rn "async function\|async def\|async (" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | wc -l

# Infrastructure configuration
cat docker-compose.yml Dockerfile 2>/dev/null | head -40
cat .github/workflows/*.yml 2>/dev/null | head -40

# Check for health checks and monitoring
grep -rn "health\|readiness\|liveness\|monitor\|sentry\|datadog\|prometheus" --include="*.ts" --include="*.js" --include="*.py" --include="*.yaml" --include="*.yml" . 2>/dev/null | grep -v "node_modules" | head -10

# Check for secrets management
grep -rn "process\.env\|os\.environ\|os\.Getenv" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" src/ app/ 2>/dev/null | grep -v "node_modules\|test\|spec" | wc -l
ls .env .env.local .env.production 2>/dev/null
```

### 1b. Project & Delivery Risks

Evaluate from context, PRDs, recent activity:
```bash
# Recent velocity (commits per week)
for week in 4 3 2 1 0; do
  start=$(date -d "$((week+1)) weeks ago" +%Y-%m-%d 2>/dev/null || date -v-$((week+1))w +%Y-%m-%d 2>/dev/null)
  end=$(date -d "$week weeks ago" +%Y-%m-%d 2>/dev/null || date -v-${week}w +%Y-%m-%d 2>/dev/null)
  count=$(git log --oneline --after="$start" --before="$end" 2>/dev/null | wc -l)
  echo "Week -$week: $count commits"
done

# PR cycle time (how long PRs stay open)
gh pr list --state merged --limit 10 --json number,title,createdAt,mergedAt 2>/dev/null | head -40

# Open PRs (work in progress)
gh pr list --state open --json number,title,createdAt,author 2>/dev/null | head -20

# Pending issues
gh issue list --state open --limit 20 --json number,title,labels,createdAt 2>/dev/null | head -40

# Recent incidents
ls project-decisions/*incident* 2>/dev/null

# Recent scope changes or decision records
ls project-decisions/ 2>/dev/null | tail -10

# Check for deadline references
grep -rn "deadline\|due date\|launch\|go-live\|ship by\|target date" --include="*.md" . 2>/dev/null | grep -v "node_modules\|\.git" | head -10
```

## 2. Risk Categories

### Technical Risks

| ID | Risk Category | What to Look For |
|----|--------------|-----------------|
| T1 | **Architecture** | Single points of failure, monolith pain points, scaling bottlenecks, circular dependencies |
| T2 | **Code Quality** | High complexity files, low test coverage, excessive tech debt, code smells |
| T3 | **Dependencies** | Vulnerable packages, outdated major versions, unmaintained libraries, license issues |
| T4 | **Security** | Exposed secrets, injection vulnerabilities, auth gaps, data exposure |
| T5 | **Performance** | Slow queries, memory leaks, missing caching, N+1 problems |
| T6 | **Data** | Missing backups, no migration rollback, data integrity gaps, missing validation |
| T7 | **Infrastructure** | No redundancy, manual deployments, missing monitoring, no auto-scaling |
| T8 | **Integration** | Flaky third-party APIs, missing circuit breakers, undocumented API contracts |

### Delivery Risks

| ID | Risk Category | What to Look For |
|----|--------------|-----------------|
| D1 | **Timeline** | Unrealistic deadlines, scope creep, incomplete requirements, blocked tasks |
| D2 | **Resources** | Team capacity constraints, key person dependency, skill gaps, competing priorities |
| D3 | **Scope** | Vague requirements, missing acceptance criteria, unbounded features, no MVP definition |
| D4 | **Dependencies** | Cross-team blockers, external vendor timelines, design deliverables, stakeholder approvals |
| D5 | **Communication** | Unclear ownership, missing documentation, no stakeholder alignment, siloed knowledge |

### Operational Risks

| ID | Risk Category | What to Look For |
|----|--------------|-----------------|
| O1 | **Availability** | No SLA defined, missing health checks, no incident response plan, no runbooks |
| O2 | **Disaster Recovery** | No backup strategy, untested recovery, missing failover, no RTO/RPO targets |
| O3 | **Compliance** | GDPR gaps, missing audit logging, data retention policy unclear, security certifications pending |
| O4 | **Support** | No on-call rotation, missing runbooks, no escalation path, knowledge silos |

### Business Risks

| ID | Risk Category | What to Look For |
|----|--------------|-----------------|
| B1 | **Market** | Competitive pressure, changing requirements, pivoting product direction |
| B2 | **Vendor** | Vendor lock-in, pricing changes, vendor stability, contract expiry |
| B3 | **Revenue** | Payment system reliability, billing accuracy, churn risk from outages |
| B4 | **Reputation** | Data breach risk, public-facing outage risk, user trust |

## 3. Risk Scoring

### Likelihood Scale

| Score | Level | Definition | Probability |
|-------|-------|-----------|------------|
| 1 | **Rare** | Could happen but very unlikely in the next 3 months | < 10% |
| 2 | **Unlikely** | Possible but not expected | 10-30% |
| 3 | **Possible** | Could go either way | 30-60% |
| 4 | **Likely** | More likely than not | 60-85% |
| 5 | **Almost Certain** | Will very likely happen | > 85% |

### Impact Scale

| Score | Level | Definition | Examples |
|-------|-------|-----------|---------|
| 1 | **Negligible** | Minor inconvenience, no user impact | Cosmetic bug, minor delay |
| 2 | **Minor** | Small user impact, easy to fix | Edge case bug, 1-2 day delay |
| 3 | **Moderate** | Noticeable impact, workaround exists | Feature degraded, 1 week delay |
| 4 | **Major** | Significant impact, hard to work around | Core feature broken, 2+ week delay, partial data loss |
| 5 | **Severe** | Critical failure, no workaround | Full outage, data breach, project cancelled, regulatory fine |

### Risk Score Matrix
```
                        IMPACT
                 1     2     3     4     5
            ┌─────┬─────┬─────┬─────┬─────┐
     5      │  5  │ 10  │ 15  │ 20  │ 25  │
            │ 🟡  │ 🟠  │ 🔴  │ 🔴  │ 🔴  │
L    ──────┼─────┼─────┼─────┼─────┼─────┤
I    4      │  4  │  8  │ 12  │ 16  │ 20  │
K           │ 🟢  │ 🟡  │ 🟠  │ 🔴  │ 🔴  │
E    ──────┼─────┼─────┼─────┼─────┼─────┤
L    3      │  3  │  6  │  9  │ 12  │ 15  │
I           │ 🟢  │ 🟡  │ 🟡  │ 🟠  │ 🔴  │
H    ──────┼─────┼─────┼─────┼─────┼─────┤
O    2      │  2  │  4  │  6  │  8  │ 10  │
O           │ 🟢  │ 🟢  │ 🟡  │ 🟡  │ 🟠  │
D    ──────┼─────┼─────┼─────┼─────┼─────┤
     1      │  1  │  2  │  3  │  4  │  5  │
            │ 🟢  │ 🟢  │ 🟢  │ 🟢  │ 🟡  │
            └─────┴─────┴─────┴─────┴─────┘

Score ranges:
🟢 Low (1-4):       Accept — monitor, no immediate action
🟡 Medium (5-9):    Mitigate — plan mitigation, review regularly
🟠 High (10-15):    Act — active mitigation required, escalate
🔴 Critical (16-25): Urgent — immediate action, executive visibility
```

### Risk Score Calculation
```
Risk Score = Likelihood × Impact

Example:
  Risk: "Key developer leaves before project completion"
  Likelihood: 3 (Possible)
  Impact: 4 (Major — critical knowledge loss, 2+ week delay)
  Score: 3 × 4 = 12 (🟠 High)
```

## 4. Risk Response Strategies

For each identified risk, choose a response strategy:

| Strategy | When to Use | Example |
|----------|------------|---------|
| **Avoid** | Eliminate the risk entirely by changing approach | Don't use the unproven technology; use the established one instead |
| **Mitigate** | Reduce likelihood or impact | Add tests, create documentation, build redundancy |
| **Transfer** | Shift risk to a third party | Use managed service instead of self-hosting; buy insurance |
| **Accept** | Risk is low enough or unavoidable | Known minor UI bug that doesn't affect core functionality |
| **Contingency** | Prepare a plan B if the risk materializes | Rollback plan, backup vendor, alternative approach ready |

## 5. Risk Register Entry Format

Each risk should include:
```
### RISK-[ID]: [Title]

| Field | Value |
|-------|-------|
| **Category** | [Technical / Delivery / Operational / Business] |
| **Subcategory** | [T1-T8 / D1-D5 / O1-O4 / B1-B4] |
| **Description** | [What could happen and why] |
| **Trigger** | [What event or condition would cause this risk to materialize] |
| **Likelihood** | [1-5] [Rare/Unlikely/Possible/Likely/Almost Certain] |
| **Impact** | [1-5] [Negligible/Minor/Moderate/Major/Severe] |
| **Score** | [L × I] [🟢/🟡/🟠/🔴] |
| **Response** | [Avoid / Mitigate / Transfer / Accept / Contingency] |
| **Mitigation** | [Specific actions to reduce likelihood or impact] |
| **Contingency** | [What to do if the risk materializes] |
| **Owner** | [Person responsible for monitoring and acting] |
| **Status** | [Open / Mitigating / Mitigated / Accepted / Realized / Closed] |
| **Due Date** | [When mitigation should be complete] |
| **Evidence** | [Data from codebase scan, metrics, or observations] |
| **Linked Items** | [Related tickets, incidents, decisions] |
| **Last Reviewed** | [Date] |
```

## 6. Automated Risk Detection Rules

### Auto-Flag as 🔴 Critical
```
IF any of these are true, auto-flag as critical risk:

- Dependency with known critical CVE (CVSS ≥ 9.0)
- Secrets/credentials committed to git
- Production database has no backup configured
- Zero test coverage on authentication or payment code
- Single point of failure in production architecture
- No rollback strategy for upcoming deployment
- Key person dependency on critical path with no documentation
- Deadline is < 2 weeks and > 30% of scope is incomplete
```

### Auto-Flag as 🟠 High
```
IF any of these are true, auto-flag as high risk:

- Dependency with known high CVE (CVSS ≥ 7.0)
- Test coverage < 30% on modified files
- Files with > 500 lines and no tests
- Bus factor of 1 on > 5 critical files
- More than 20 unresolved TODOs/FIXMEs in critical paths
- No monitoring/alerting on production service
- Third-party API with no circuit breaker or fallback
- Sprint velocity declining for 3+ consecutive sprints
- PR cycle time > 5 days average
```

### Auto-Flag as 🟡 Medium
```
IF any of these are true, auto-flag as medium risk:

- Dependencies > 6 months outdated
- No API documentation for public endpoints
- Missing .env.example or setup documentation
- No runbook for common failure scenarios
- Inconsistent error handling patterns
- Code duplication detected across > 3 files
```

## 7. Risk Trends

Track how risks change over time:
```
### Risk Trend: [Risk Title]

| Date | Likelihood | Impact | Score | Change | Notes |
|------|-----------|--------|-------|--------|-------|
| 2026-01-15 | 3 | 4 | 12 🟠 | — | Initial assessment |
| 2026-01-29 | 3 | 4 | 12 🟠 | → | No change, mitigation in progress |
| 2026-02-12 | 2 | 4 | 8 🟡 | ↓ | Tests added, documentation improved |
| 2026-02-19 | 2 | 3 | 6 🟡 | ↓ | Second engineer onboarded to module |

Trend: ↓ Improving
```

Trend symbols:
```
↑ Worsening (score increased)
→ Stable (no change)
↓ Improving (score decreased)
⚡ Realized (risk became an actual issue)
✅ Closed (risk eliminated or accepted and documented)
```

## 8. Review Cadence
```
Recommended review schedule:

| Review Type | Frequency | Who | Focus |
|------------|-----------|-----|-------|
| Quick scan | Every sprint | TPM | New risks, status updates, score changes |
| Full review | Monthly | TPM + Tech Lead | All risks, trends, mitigation effectiveness |
| Deep dive | Quarterly | Full team | Architecture risks, strategic risks, historical trends |
| Ad-hoc | As needed | TPM | After incidents, major scope changes, team changes |
```

## Output Document Template

Save to `project-decisions/YYYY-MM-DD-risk-register.md`:
```markdown
# Project Risk Register

**Project:** [Project Name]
**Last Updated:** YYYY-MM-DD
**Updated By:** [Name]
**Next Review:** YYYY-MM-DD
**Overall Risk Level:** [🟢 Low / 🟡 Medium / 🟠 High / 🔴 Critical]

---

## Risk Summary

| Severity | Count | Trend |
|----------|-------|-------|
| 🔴 Critical | X | [↑/→/↓] |
| 🟠 High | X | [↑/→/↓] |
| 🟡 Medium | X | [↑/→/↓] |
| 🟢 Low | X | [↑/→/↓] |
| **Total Open** | **X** | |
| Mitigated this period | X | |
| New this period | X | |
| Realized (became issues) | X | |

---

## Risk Heat Map
```
                        IMPACT
                 1     2     3     4     5
            ┌─────┬─────┬─────┬─────┬─────┐
     5      │     │     │     │ R03 │     │
L    ──────┼─────┼─────┼─────┼─────┼─────┤
I    4      │     │     │ R07 │ R01 │     │
K    ──────┼─────┼─────┼─────┼─────┼─────┤
E    3      │     │ R09 │ R04 │ R02 │     │
L    ──────┼─────┼─────┼─────┼─────┼─────┤
I    2      │ R10 │ R08 │ R06 │     │     │
H    ──────┼─────┼─────┼─────┼─────┼─────┤
O    1      │     │ R11 │ R05 │     │     │
O           └─────┴─────┴─────┴─────┴─────┘
D
```

---

## Top Risks Requiring Action

| Rank | ID | Risk | Score | Owner | Status | Due |
|------|----|------|-------|-------|--------|-----|
| 1 | R01 | [Title] | 16 🔴 | [Name] | [Status] | [Date] |
| 2 | R02 | [Title] | 12 🟠 | [Name] | [Status] | [Date] |
| 3 | R03 | [Title] | 20 🔴 | [Name] | [Status] | [Date] |

---

## Full Risk Register

### 🔴 Critical Risks

#### RISK-001: [Title]

| Field | Value |
|-------|-------|
| **Category** | [Category] |
| **Description** | [What could happen] |
| **Trigger** | [What would cause this] |
| **Likelihood** | [X] — [Level] |
| **Impact** | [X] — [Level] |
| **Score** | [XX] 🔴 |
| **Response** | [Strategy] |
| **Mitigation** | [Actions] |
| **Contingency** | [Plan B] |
| **Owner** | [Name] |
| **Status** | [Status] |
| **Due Date** | [Date] |
| **Evidence** | [Codebase findings] |
| **Last Reviewed** | [Date] |

**Trend:**
| Date | L | I | Score | Change | Notes |
|------|---|---|-------|--------|-------|
| [Date] | X | X | XX | — | [Notes] |

---

[Repeat for each risk...]

---

### 🟠 High Risks

[Same format...]

### 🟡 Medium Risks

[Same format...]

### 🟢 Low Risks

[Same format...]

---

## Realized Risks (became actual issues)

| ID | Risk | Realized Date | Impact | Incident Link |
|----|------|--------------|--------|--------------|
| R05 | [Title] | YYYY-MM-DD | [Actual impact] | [Link to incident report] |

---

## Closed Risks

| ID | Risk | Closed Date | Reason |
|----|------|------------|--------|
| R12 | [Title] | YYYY-MM-DD | [Mitigated / Accepted / No longer relevant] |

---

## Risk Metrics

| Metric | Current | Previous | Trend |
|--------|---------|----------|-------|
| Total open risks | X | X | [↑/→/↓] |
| Average risk score | X.X | X.X | [↑/→/↓] |
| Critical + High risks | X | X | [↑/→/↓] |
| Risks mitigated this period | X | X | |
| Risks realized this period | X | X | |
| Mean time to mitigate | X days | X days | [↑/→/↓] |
| Overdue mitigations | X | X | [↑/→/↓] |

---

## Upcoming Mitigation Actions

| Risk ID | Action | Owner | Due | Status |
|---------|--------|-------|-----|--------|
| R01 | [Specific action] | [Name] | [Date] | ⬜ TODO |
| R02 | [Specific action] | [Name] | [Date] | 🔄 In Progress |
| R03 | [Specific action] | [Name] | [Date] | ⬜ TODO |

---

## Review Log

| Date | Type | Reviewer | Changes Made |
|------|------|----------|-------------|
| YYYY-MM-DD | Initial creation | [Name] | Created register with X risks |
| YYYY-MM-DD | Sprint review | [Name] | Updated R01, added R15, closed R05 |
| YYYY-MM-DD | Monthly review | [Name] | Full review, re-scored 3 risks |
```

After saving, update the project-decisions index:
```bash
echo "# Project Decisions\n" > project-decisions/README.md
echo "| Date | Decision | Type | Status |" >> project-decisions/README.md
echo "|------|----------|------|--------|" >> project-decisions/README.md

for f in project-decisions/2*.md; do
  date=$(basename "$f" | cut -d'-' -f1-3)
  title=$(head -1 "$f" | sed 's/^# //')
  type="Other"
  echo "$f" | grep -q "risk-register" && type="Risk Register"
  echo "$f" | grep -q "build-vs-buy" && type="Build vs Buy"
  echo "$f" | grep -q "incident" && type="Incident Report"
  echo "$f" | grep -q "scope" && type="Scope Check"
  echo "$f" | grep -q "impact" && type="Impact Analysis"
  echo "$f" | grep -q "tech-debt" && type="Tech Debt Report"
  echo "$f" | grep -q "pentest" && type="Pentest Report"
  echo "$f" | grep -qv "risk-register\|build-vs-buy\|incident\|scope\|impact\|tech-debt\|pentest" && type="Tech Decision"
  status=$(grep "^**Status:\|^**Overall Risk Level:\|^**Last Updated:" "$f" | head -1 | sed 's/.*: //' | sed 's/\*//g')
  echo "| $date | [$title](./$(basename $f)) | $type | $status |" >> project-decisions/README.md
done
```

## Adaptation Rules

- **Always save to file** — every risk register gets persisted in `project-decisions/`
- **Scan the codebase** — don't guess at technical risks, find them with grep, git log, npm audit
- **Be specific** — "authService.ts has 0% test coverage and handles password hashing" not "some code is untested"
- **Include evidence** — every technical risk should reference actual files, metrics, or scan results
- **Score consistently** — use the same likelihood and impact scales every time
- **Track trends** — show whether each risk is improving, stable, or worsening
- **Update, don't recreate** — if a risk register already exists, update it rather than starting from scratch
- **Link to other documents** — connect realized risks to incident reports, mitigations to tech decisions
- **Assign owners** — unowned risks don't get mitigated
- **Flag overdue mitigations** — a mitigation plan that's past due is itself a risk
- **Scale to project** — small project gets 5-10 risks, large project gets 20-30
- **Distinguish symptoms from risks** — "slow API" is a symptom, "no caching strategy for growing dataset" is the risk

## Summary

End every risk register with:
1. **Overall risk level** — 🟢/🟡/🟠/🔴 based on highest open risk
2. **Risk count** — total open, by severity
3. **Top 3 risks** — requiring immediate attention
4. **New risks** — added since last review
5. **Trend** — overall trajectory (improving / stable / worsening)
6. **Overdue actions** — mitigations past their due date
7. **Next review date** — when this should be updated
8. **File saved** — confirm the document location
