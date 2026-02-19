---
name: incident-report
description: Generates structured incident postmortem reports by analyzing git history, recent deployments, code changes, logs, and error patterns. Produces a blameless postmortem with timeline, root cause analysis, impact assessment, remediation actions, and prevention measures. Saves output to project-decisions/ folder. Use when the user says "incident report", "postmortem", "what went wrong", "outage report", "root cause analysis", "RCA", "write a post-mortem", "incident review", "we had an incident", "production issue", or "site went down".
allowed-tools: Read, Grep, Glob, Bash
---

# Incident Report Skill

When generating an incident report, follow this structured process. The goal is to produce a blameless postmortem that helps the team learn from what happened and prevents recurrence — not to assign blame.

**IMPORTANT**: Always save the output as a markdown file in the `project-decisions/` directory at the project root. Create the directory if it doesn't exist.

**PRINCIPLE**: This is a BLAMELESS postmortem. Focus on systems, processes, and conditions — never on individuals. Replace "Person X did wrong" with "The system allowed X to happen without safeguards."

## 0. Output Setup
```bash
# Create project-decisions directory if it doesn't exist
mkdir -p project-decisions

# File will be saved as:
# project-decisions/YYYY-MM-DD-incident-[kebab-case-topic].md
# Example: project-decisions/2026-02-19-incident-payment-processing-outage.md
```

## 1. Incident Discovery — Gather the Facts

### Recent Deployments
```bash
# Recent deployments / merges to main
git log --oneline --merges --since="7 days ago" main 2>/dev/null | head -20
git log --oneline --since="7 days ago" main 2>/dev/null | head -30

# Tags / releases in the last week
git tag --sort=-creatordate | head -10

# What was deployed most recently?
git log --oneline -1 main
git log --format="%H %ai %s" -1 main

# Who deployed and when?
git log --format="%ai — %an — %s" --since="3 days ago" main | head -20

# Diff between last two deployments
LATEST=$(git log --oneline -1 main | cut -d' ' -f1)
PREVIOUS=$(git log --oneline -2 main | tail -1 | cut -d' ' -f1)
git diff --stat $PREVIOUS $LATEST 2>/dev/null
git diff --name-only $PREVIOUS $LATEST 2>/dev/null
```

### Recent Code Changes
```bash
# Files changed in the last 3 days
git log --name-only --since="3 days ago" --format="" main 2>/dev/null | sort | uniq -c | sort -rn | head -20

# Most changed files recently (hot spots)
git log --name-only --since="7 days ago" --format="" main 2>/dev/null | sort | uniq -c | sort -rn | head -10

# Changes to critical areas (auth, payments, database)
git log --oneline --since="7 days ago" -- src/auth/ src/payment/ src/db/ 2>/dev/null | head -10

# Changes to configuration
git log --oneline --since="7 days ago" -- "*.config.*" "*.env*" "docker-compose*" "*.yaml" "*.yml" 2>/dev/null | head -10

# Changes to infrastructure
git log --oneline --since="7 days ago" -- Dockerfile docker-compose* k8s/ terraform/ .github/workflows/ 2>/dev/null | head -10

# Changes to dependencies
git log --oneline --since="7 days ago" -- package.json package-lock.json yarn.lock requirements.txt Gemfile.lock 2>/dev/null | head -10
```

### Error Patterns in Code
```bash
# Check for error handling in recently changed files
for file in $(git diff --name-only $PREVIOUS $LATEST 2>/dev/null); do
  echo "=== $file ==="
  grep -n "catch\|except\|rescue\|error\|throw\|panic\|fatal" "$file" 2>/dev/null | head -5
done

# Check for recent TODO/FIXME/HACK that might be relevant
grep -rn "TODO\|FIXME\|HACK\|XXX\|WORKAROUND\|TEMPORARY" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -i "[incident-keyword]"

# Check for missing error handling in affected area
grep -rn "[incident-area-keyword]" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | head -20

# Check for known issues in the area
grep -rn "KNOWN.*ISSUE\|BUG\|REGRESSION" --include="*.ts" --include="*.js" --include="*.py" --include="*.md" . 2>/dev/null | head -10
```

### Environment & Configuration
```bash
# Check environment configuration
cat .env.example 2>/dev/null | head -30

# Check for recent config changes
git log --oneline --since="7 days ago" -- "*.config.*" "*.env*" config/ 2>/dev/null

# Check Docker/infrastructure config
cat docker-compose.yml 2>/dev/null | head -50
cat Dockerfile 2>/dev/null | head -30

# Check for health checks
grep -rn "health\|readiness\|liveness\|heartbeat" --include="*.ts" --include="*.js" --include="*.py" --include="*.yaml" --include="*.yml" . 2>/dev/null | head -10

# Check monitoring configuration
grep -rn "sentry\|datadog\|newrelic\|prometheus\|grafana\|pagerduty\|opsgenie" --include="*.ts" --include="*.js" --include="*.py" --include="*.yaml" --include="*.json" . 2>/dev/null | head -10
```

### Database State
```bash
# Recent migrations
ls -la src/db/migrations/ db/migrations/ migrations/ prisma/migrations/ 2>/dev/null | tail -10

# Recent migration changes
git log --oneline --since="7 days ago" -- "**/migrations/**" "prisma/" 2>/dev/null | head -10

# Check for schema changes
git diff --name-only $PREVIOUS $LATEST 2>/dev/null | grep -i "migration\|schema\|prisma"
```

## 2. Incident Classification

### Severity Levels

| Level | Name | Definition | Examples |
|-------|------|-----------|---------|
| **SEV-1** | Critical | Complete service outage affecting all users, data loss, or security breach | Site down, database corruption, credential leak |
| **SEV-2** | Major | Significant degradation affecting many users, core functionality broken | Payment processing failing, auth broken for 50%+ users |
| **SEV-3** | Minor | Partial degradation affecting some users, workaround available | Search not working, slow page loads, one API endpoint failing |
| **SEV-4** | Low | Minimal impact, cosmetic issues, edge case bugs in production | UI glitch, typo, non-critical feature broken for small user segment |

### Incident Categories

| Category | Description |
|----------|------------|
| **Availability** | Service down or unreachable |
| **Performance** | Service slow or degraded |
| **Data** | Data loss, corruption, or inconsistency |
| **Security** | Unauthorized access, data breach, vulnerability exploited |
| **Functionality** | Feature broken or behaving incorrectly |
| **Integration** | Third-party service failure or miscommunication |
| **Infrastructure** | Server, network, DNS, or cloud provider issue |
| **Deployment** | Bad deploy, failed rollback, configuration error |
| **Capacity** | Resource exhaustion (disk, memory, connections, rate limits) |
| **Dependency** | Upstream service failure cascading to our system |

## 3. Timeline Construction

Build a precise timeline of events:
```
## Timeline

All times in [timezone — e.g., UTC]

| Time | Event | Source |
|------|-------|--------|
| YYYY-MM-DD HH:MM | [Normal state — last known good] | [monitoring/logs] |
| YYYY-MM-DD HH:MM | [Triggering event — deployment, config change, traffic spike] | [deploy log/git] |
| YYYY-MM-DD HH:MM | [First symptom — error rate increase, latency spike] | [monitoring] |
| YYYY-MM-DD HH:MM | [Detection — alert fired, user report, team noticed] | [PagerDuty/Slack] |
| YYYY-MM-DD HH:MM | [Response started — on-call engaged, investigation began] | [Slack/team] |
| YYYY-MM-DD HH:MM | [Diagnosis — root cause identified or hypothesized] | [team] |
| YYYY-MM-DD HH:MM | [Mitigation action taken — rollback, fix, restart, config change] | [deploy log/git] |
| YYYY-MM-DD HH:MM | [Partial recovery — some users restored] | [monitoring] |
| YYYY-MM-DD HH:MM | [Full recovery — service fully restored] | [monitoring] |
| YYYY-MM-DD HH:MM | [Verification — confirmed stable, monitoring clean] | [monitoring] |
| YYYY-MM-DD HH:MM | [Incident closed] | [team] |
```

### Key Metrics
```
| Metric | Value |
|--------|-------|
| **Time to detect (TTD)** | X minutes (from trigger to detection) |
| **Time to respond (TTR)** | X minutes (from detection to first responder) |
| **Time to mitigate (TTM)** | X minutes (from response to mitigation) |
| **Time to resolve (TTR)** | X minutes (from trigger to full recovery) |
| **Total duration** | X hours X minutes |
| **User-facing impact duration** | X hours X minutes |
```

## 4. Root Cause Analysis

### The 5 Whys

Drill down from the symptom to the root cause:
```
1. WHY did [the incident happen]?
   → Because [immediate cause]

2. WHY did [immediate cause] happen?
   → Because [deeper cause]

3. WHY did [deeper cause] happen?
   → Because [even deeper cause]

4. WHY did [even deeper cause] happen?
   → Because [systemic cause]

5. WHY did [systemic cause] exist?
   → Because [root cause — process, system, or cultural gap]
```

### Contributing Factors

Incidents rarely have a single cause. Identify all contributing factors:
```
| Factor | Type | Contribution |
|--------|------|-------------|
| [Factor 1 — e.g., untested migration] | Technical | Primary cause |
| [Factor 2 — e.g., no staging test] | Process | Allowed primary cause to reach production |
| [Factor 3 — e.g., no rollback plan] | Process | Extended the incident duration |
| [Factor 4 — e.g., missing monitoring] | Observability | Delayed detection |
| [Factor 5 — e.g., no runbook] | Documentation | Slowed response |
```

### Cause Categories

Classify the root cause:

| Category | Examples |
|----------|---------|
| **Code defect** | Bug in logic, missing error handling, race condition |
| **Configuration error** | Wrong env var, bad config value, missing secret |
| **Deployment issue** | Bad deploy process, missing migration, dependency conflict |
| **Infrastructure** | Server failure, network issue, resource exhaustion |
| **Dependency failure** | Third-party API down, upstream service degraded |
| **Data issue** | Bad data, missing data, data migration failure |
| **Capacity** | Traffic spike, resource limits hit, connection pool exhausted |
| **Process gap** | Missing review, no testing, no monitoring, no runbook |
| **Communication** | Team not notified, unclear ownership, missing context |
| **Design flaw** | Architectural limitation, missing circuit breaker, no retry logic |

### Fault Tree (for Complex Incidents)
```
                    [INCIDENT: Service Outage]
                              │
                    ┌─────────┼─────────┐
                    │         │         │
            [Database      [App      [Monitoring
             timeout]      crash]    didn't alert]
                │            │           │
          ┌─────┴─────┐     │      [Alert threshold
          │           │     │       too high]
    [Long-running  [Connection   │
     query]        pool full]  [Not configured
          │           │        for this metric]
    [Missing    [Connection
     index]     leak in
                new code]
```

## 5. Impact Assessment

### User Impact
```
| Metric | Value |
|--------|-------|
| **Users affected** | X (Y% of total) |
| **Requests failed** | X (Y% error rate) |
| **Revenue impact** | $X (estimated lost transactions) |
| **Data affected** | X records (lost / corrupted / delayed) |
| **SLA impact** | X minutes of downtime against Y% SLA target |
| **Support tickets** | X tickets generated |
| **Public visibility** | [status page updated / social media mentions / press] |
```

### System Impact
```
| System | Impact | Duration |
|--------|--------|----------|
| [API] | 503 errors on all endpoints | X minutes |
| [Frontend] | Error page displayed | X minutes |
| [Mobile app] | Requests timing out | X minutes |
| [Background jobs] | Queue backlog of X jobs | X minutes |
| [Partner API] | Webhook delivery failed for X events | X minutes |
| [Database] | Connection pool exhausted | X minutes |
| [Cache] | Cache invalidated, cold start | X minutes |
```

### Business Impact
```
| Area | Impact | Quantified |
|------|--------|-----------|
| **Revenue** | Lost transactions during outage | $X |
| **SLA** | SLA credit owed to customers | $X |
| **Reputation** | Trust impact, social media | [Low/Medium/High] |
| **Compliance** | Regulatory reporting required? | [Yes/No] |
| **Internal** | Engineering time spent on incident | X person-hours |
| **Opportunity** | Delayed feature work due to incident | X days |
```

## 6. Response Evaluation

### What Went Well
```
✅ Things that worked during the incident:
- [e.g., Alert fired within 2 minutes of first error]
- [e.g., On-call responded within 5 minutes]
- [e.g., Rollback was clean and fast]
- [e.g., Status page was updated promptly]
- [e.g., Communication in Slack was clear and organized]
- [e.g., Customer support had talking points ready quickly]
```

### What Didn't Go Well
```
❌ Things that didn't work or slowed us down:
- [e.g., Took 30 minutes to identify which deploy caused the issue]
- [e.g., No runbook existed for this failure mode]
- [e.g., Logs were insufficient to diagnose the problem]
- [e.g., Rollback required manual database intervention]
- [e.g., On-call didn't have access to production logs]
- [e.g., Status page wasn't updated for 45 minutes]
```

### Where We Got Lucky
```
🍀 Things that could have made it worse:
- [e.g., Happened during low-traffic hours — peak would have been 10x worse]
- [e.g., Only affected one region — could have been global]
- [e.g., No data was permanently lost — could have been unrecoverable]
- [e.g., A team member happened to be online who knew this system]
```

## 7. Action Items

### Remediation Actions

Categorize action items by urgency and type:

#### Immediate (0-48 hours) — Prevent Recurrence of THIS Incident
```
| # | Action | Type | Owner | Deadline | Status |
|---|--------|------|-------|----------|--------|
| 1 | [Fix the specific bug/config that caused the incident] | Fix | [Name] | [Date] | ⬜ TODO |
| 2 | [Add monitoring for the specific failure mode] | Detection | [Name] | [Date] | ⬜ TODO |
| 3 | [Write runbook for this incident type] | Documentation | [Name] | [Date] | ⬜ TODO |
```

#### Short-Term (1-2 weeks) — Harden Against Similar Incidents
```
| # | Action | Type | Owner | Deadline | Status |
|---|--------|------|-------|----------|--------|
| 4 | [Add integration test for the failure scenario] | Prevention | [Name] | [Date] | ⬜ TODO |
| 5 | [Improve logging in affected area] | Detection | [Name] | [Date] | ⬜ TODO |
| 6 | [Add circuit breaker for external dependency] | Resilience | [Name] | [Date] | ⬜ TODO |
| 7 | [Update deployment checklist] | Process | [Name] | [Date] | ⬜ TODO |
```

#### Medium-Term (1-3 months) — Systemic Improvements
```
| # | Action | Type | Owner | Deadline | Status |
|---|--------|------|-------|----------|--------|
| 8 | [Implement automated canary deployments] | Prevention | [Name] | [Date] | ⬜ TODO |
| 9 | [Add load testing to CI/CD pipeline] | Prevention | [Name] | [Date] | ⬜ TODO |
| 10 | [Redesign the system to eliminate single point of failure] | Architecture | [Name] | [Date] | ⬜ TODO |
| 11 | [Implement chaos engineering practices] | Resilience | [Name] | [Date] | ⬜ TODO |
```

### Action Item Categories

| Type | Purpose | Examples |
|------|---------|---------|
| **Fix** | Directly fix the cause | Bug fix, config correction, data repair |
| **Detection** | Catch it faster next time | Monitoring, alerting, logging |
| **Prevention** | Stop it from happening | Tests, validation, guardrails, automation |
| **Resilience** | Reduce impact when it does happen | Circuit breakers, fallbacks, graceful degradation |
| **Process** | Improve how we work | Checklists, review gates, runbooks |
| **Documentation** | Capture knowledge | Runbooks, architecture docs, decision records |
| **Architecture** | Structural improvements | Eliminate SPOF, add redundancy, decouple |

## 8. Lessons Learned
```
## Key Takeaways

1. **[Lesson 1]**
   What happened: [brief description]
   What we learned: [insight]
   How we'll apply it: [specific action]

2. **[Lesson 2]**
   What happened: [brief description]
   What we learned: [insight]
   How we'll apply it: [specific action]

3. **[Lesson 3]**
   What happened: [brief description]
   What we learned: [insight]
   How we'll apply it: [specific action]
```

## 9. Prevention Framework

### Detection Improvements
```
What should have caught this earlier:

| Gap | Current State | Target State | Action Item |
|-----|---------------|-------------|-------------|
| [Monitoring gap] | No alert for X | Alert when X > threshold | Add dashboard + alert |
| [Logging gap] | No logs for Y operation | Structured logs with context | Add logging to service |
| [Testing gap] | No test for Z scenario | Integration test covering Z | Write test |
```

### Process Improvements
```
What process changes would prevent this:

| Gap | Current Process | Improved Process | Action Item |
|-----|----------------|-----------------|-------------|
| [Review gap] | No security review on config changes | Config changes require review | Update PR checklist |
| [Deploy gap] | Direct deploy to production | Staged rollout with canary | Implement canary deploys |
| [Runbook gap] | No runbook for this failure | Documented response procedure | Write runbook |
```

### Architectural Improvements
```
What system design changes would help:

| Weakness | Current Design | Improved Design | Effort | Priority |
|----------|---------------|-----------------|--------|----------|
| [SPOF] | Single database | Read replicas + failover | Large | High |
| [No fallback] | Hard dependency on API | Circuit breaker + cache fallback | Medium | High |
| [No isolation] | Shared connection pool | Per-service connection pools | Medium | Medium |
```

## 10. Incident Patterns

### Check for Recurring Patterns
```bash
# Check for previous incidents in the same area
ls project-decisions/ 2>/dev/null | grep "incident"

# Check for previous related issues in git
git log --oneline --all --grep="fix\|hotfix\|revert\|rollback" --since="6 months ago" | head -20

# Check for reverted changes (indicator of past incidents)
git log --oneline --all --grep="revert\|Revert" --since="6 months ago" | head -10

# Check for similar patterns
git log --oneline --all --grep="[incident-keyword]" --since="6 months ago" | head -10
```

### Recurring Incident Check
```
Is this a recurring incident?

| Question | Answer |
|----------|--------|
| Has this exact issue happened before? | [Yes — when / No] |
| Have similar issues happened in this area? | [Yes — describe / No] |
| Were previous action items completed? | [Yes / Partially / No] |
| Is this a variant of a known weakness? | [Yes — which / No] |
| Is there a systemic pattern? | [Yes — describe / No] |
```

If recurring, escalate the priority of systemic fixes.

## Output Document Template

Save to `project-decisions/YYYY-MM-DD-incident-[topic].md`:
```markdown
# Incident Report: [Title]

**Incident ID:** INC-[number]
**Date:** YYYY-MM-DD
**Severity:** [SEV-1 / SEV-2 / SEV-3 / SEV-4]
**Category:** [Availability / Performance / Data / Security / etc.]
**Status:** [Investigating / Mitigated / Resolved / Postmortem Complete]
**Duration:** X hours Y minutes
**Author:** [Name]
**Reviewers:** [Names]

---

## Executive Summary

[2-3 sentences: what happened, what was the impact, what was the root cause, what are we doing about it]

---

## Impact

| Metric | Value |
|--------|-------|
| **Duration** | X hours Y minutes |
| **Users affected** | X (Y% of total) |
| **Requests failed** | X |
| **Revenue impact** | $X |
| **SLA impact** | X minutes against Y% target |

---

## Timeline

All times in UTC.

| Time | Event |
|------|-------|
| HH:MM | [Event 1] |
| HH:MM | [Event 2] |
| HH:MM | **⚠️ Incident begins** |
| HH:MM | [Detection] |
| HH:MM | [Response] |
| HH:MM | [Diagnosis] |
| HH:MM | [Mitigation] |
| HH:MM | **✅ Incident resolved** |

**Time to detect:** X minutes
**Time to respond:** X minutes
**Time to mitigate:** X minutes
**Total duration:** X hours Y minutes

---

## Root Cause

### Summary
[1-2 sentences describing the root cause]

### 5 Whys
1. Why did [symptom]? → [cause 1]
2. Why did [cause 1]? → [cause 2]
3. Why did [cause 2]? → [cause 3]
4. Why did [cause 3]? → [cause 4]
5. Why did [cause 4]? → **[root cause]**

### Contributing Factors
| Factor | Type | Contribution |
|--------|------|-------------|
| [Factor 1] | [Technical/Process/etc.] | [Primary/Contributing] |
| [Factor 2] | [Technical/Process/etc.] | [Contributing] |

### Technical Details
[Detailed technical explanation of what went wrong, with code references if applicable]
```
[Relevant code snippet or configuration that caused/contributed to the issue]
```

---

## Response Evaluation

### What Went Well ✅
- [Thing 1]
- [Thing 2]
- [Thing 3]

### What Didn't Go Well ❌
- [Thing 1]
- [Thing 2]
- [Thing 3]

### Where We Got Lucky 🍀
- [Thing 1]
- [Thing 2]

---

## Action Items

### Immediate (0-48 hours)
| # | Action | Type | Owner | Deadline | Status |
|---|--------|------|-------|----------|--------|
| 1 | [Action] | [Fix/Detection/Prevention] | [Name] | [Date] | ⬜ |
| 2 | [Action] | [Fix/Detection/Prevention] | [Name] | [Date] | ⬜ |

### Short-Term (1-2 weeks)
| # | Action | Type | Owner | Deadline | Status |
|---|--------|------|-------|----------|--------|
| 3 | [Action] | [Prevention/Resilience] | [Name] | [Date] | ⬜ |
| 4 | [Action] | [Process/Documentation] | [Name] | [Date] | ⬜ |

### Medium-Term (1-3 months)
| # | Action | Type | Owner | Deadline | Status |
|---|--------|------|-------|----------|--------|
| 5 | [Action] | [Architecture/Prevention] | [Name] | [Date] | ⬜ |
| 6 | [Action] | [Architecture/Prevention] | [Name] | [Date] | ⬜ |

---

## Lessons Learned

1. **[Lesson 1]**: [What we learned and how we'll apply it]
2. **[Lesson 2]**: [What we learned and how we'll apply it]
3. **[Lesson 3]**: [What we learned and how we'll apply it]

---

## Recurring Incident Check

| Question | Answer |
|----------|--------|
| Has this happened before? | [Yes/No] |
| Were previous actions completed? | [Yes/No/N/A] |
| Is there a systemic pattern? | [Yes/No] |

---

## Appendix

### A. Related Commits
[List of relevant commits with hashes and messages]

### B. Monitoring Screenshots
[Links to dashboards, error rate graphs, latency charts]

### C. Communication Log
[Key Slack messages, status page updates, customer communications]

### D. Related Incidents
[Links to previous related incident reports]

---

## Decision Log

| Date | Event | By |
|------|-------|----|
| YYYY-MM-DD | Incident detected | [Name] |
| YYYY-MM-DD | Incident mitigated | [Name] |
| YYYY-MM-DD | Incident resolved | [Name] |
| YYYY-MM-DD | Postmortem drafted | [Name] |
| YYYY-MM-DD | Postmortem reviewed | [Name] |
| YYYY-MM-DD | Action items assigned | [Name] |
| YYYY-MM-DD | All action items completed | [Name] |
```

After saving, update the project-decisions index:
```bash
# Update README.md index
echo "# Project Decisions\n" > project-decisions/README.md
echo "| Date | Decision | Type | Status |" >> project-decisions/README.md
echo "|------|----------|------|--------|" >> project-decisions/README.md

for f in project-decisions/2*.md; do
  date=$(basename "$f" | cut -d'-' -f1-3)
  title=$(head -1 "$f" | sed 's/^# //' | sed 's/^Incident Report: //' | sed 's/^Scope Check: //' | sed 's/^Impact Analysis: //' | sed 's/^Tech Decision: //')
  type="Other"
  echo "$f" | grep -q "incident" && type="Incident Report"
  echo "$f" | grep -q "scope" && type="Scope Check"
  echo "$f" | grep -q "impact" && type="Impact Analysis"
  echo "$f" | grep -q -v "incident\|scope\|impact" && type="Tech Decision"
  status=$(grep "^**Status:**" "$f" | head -1 | sed 's/.*: //' | sed 's/\*//g')
  echo "| $date | [$title](./$(basename $f)) | $type | $status |" >> project-decisions/README.md
done
```

## Adaptation Rules

- **Always save to file** — every incident report gets persisted in `project-decisions/`
- **Blameless** — never name individuals as cause; focus on systems and processes
- **Be precise on timeline** — minute-level accuracy matters for incidents
- **Scan git history** — check recent deploys, reverts, and code changes
- **Quantify impact** — "500 users affected for 45 minutes" not "some users had issues"
- **5 Whys minimum** — keep asking why until you reach a systemic cause
- **Include what went well** — incidents reveal strengths too
- **Action items must be specific** — "improve monitoring" is not actionable; "add alert for error rate > 5% on /api/payments" is
- **Every action needs an owner** — unowned actions don't get done
- **Check for recurrence** — if this happened before, escalate systemic fixes
- **Include lucky breaks** — what could have made it worse helps prioritize prevention
- **Scale to severity** — SEV-4 gets 1 page, SEV-1 gets the full treatment

## Summary

End every incident report with:
1. **One-line summary** — what happened in one sentence
2. **Severity** — SEV level
3. **Duration** — total incident time
4. **Root cause** — one sentence
5. **Action items count** — X immediate, Y short-term, Z medium-term
6. **Recurring** — is this a pattern?
7. **File saved** — confirm the document location
