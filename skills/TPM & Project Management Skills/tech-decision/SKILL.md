---
name: tech-decision
description: Evaluates technical proposals, "should we do X instead of Y?" questions, tool comparisons, and architecture suggestions. Analyzes feasibility, compares options with structured pros/cons, estimates effort and risk, and provides a clear recommendation. Saves output to project-decisions/ folder. Use when the user says "should we", "what if we", "is it worth", "should we switch to", "compare X vs Y", "evaluate this proposal", "tech decision", or brings up a technical suggestion from a team discussion.
allowed-tools: Read, Grep, Glob, Bash
---

# Tech Decision Skill

When evaluating a technical proposal or decision, follow this structured process. The goal is to turn casual "should we do X?" discussions into clear, data-driven analysis the team can act on.

**IMPORTANT**: Always save the output as a markdown file in the `project-decisions/` directory at the project root. Create the directory if it doesn't exist.

## 0. Output Setup

Before starting analysis, set up the output:
```bash
# Create project-decisions directory if it doesn't exist
mkdir -p project-decisions

# Generate filename from the decision topic
# Format: YYYY-MM-DD-short-description.md
# Example: 2026-02-19-bigquery-vs-looker-studio.md
```

The final document will be saved as:
```
project-decisions/YYYY-MM-DD-[kebab-case-topic].md
```

## 1. Understand the Proposal

### Parse the Request

Extract from the question or discussion:

- **What's being proposed?** — the specific change or decision
- **Who proposed it?** — context on their perspective
- **What problem does it solve?** — the underlying need
- **What's the current state?** — how things work today
- **What triggered this?** — why now? new tool, pain point, opportunity?
- **Who's affected?** — which teams, users, or systems

### Scan the Codebase for Context
```bash
# Find references to current tools/systems mentioned
grep -rn "[current-tool]" --include="*.ts" --include="*.js" --include="*.py" --include="*.yaml" --include="*.yml" --include="*.json" --include="*.md" --include="*.env*" . 2>/dev/null | grep -v "node_modules\|\.git" | head -20

# Find references to proposed tools/systems
grep -rn "[proposed-tool]" --include="*.ts" --include="*.js" --include="*.py" --include="*.yaml" --include="*.yml" --include="*.json" --include="*.md" --include="*.env*" . 2>/dev/null | grep -v "node_modules\|\.git" | head -20

# Check existing integrations and dependencies
cat package.json pyproject.toml docker-compose.yml 2>/dev/null | grep -iE "[tool-a]\|[tool-b]"

# Check for existing decision records
ls project-decisions/ docs/adr/ docs/decisions/ 2>/dev/null

# Check for related configuration
find . -name "*.config.*" -o -name "*.yaml" -o -name "*.yml" -o -name "*.toml" | grep -v "node_modules\|\.git" | head -20
```

## 2. Frame the Decision

### Decision Statement

Write a clear, neutral decision statement:
```
Decision: Should we [proposed change] instead of [current approach]?

Context: [1-2 sentences on why this came up]

Constraints:
- Timeline: [any deadline pressure]
- Budget: [cost considerations]
- Team capacity: [available engineering time]
- Technical constraints: [compatibility, legacy systems, compliance]
```

### Identify Options

Always evaluate at least 3 options:
```
Option A: Keep current approach (status quo / do nothing)
Option B: [The proposed change]
Option C: [A hybrid or alternative approach]
Option D: [Another alternative if applicable]
```

Never evaluate just the proposed option — always compare against the status quo and at least one alternative.

## 3. Analyze Each Option

For each option, evaluate:

### 3a. Feasibility Analysis

| Question | Assessment |
|----------|-----------|
| **Can we actually do this?** | Yes / Yes with caveats / Uncertain / No |
| **Do we have the skills?** | Team has experience / Learning needed / External help needed |
| **Do we have the tools?** | Available / Need to purchase / Need to build |
| **Does it integrate with our stack?** | Native support / Adapter available / Custom integration needed |
| **Are there blockers?** | None / Soft blockers / Hard blockers |
| **Is it proven?** | Mature & widely used / Emerging / Experimental |

### 3b. Effort & Timeline
```
Estimated effort breakdown:

| Phase | Duration | People | Notes |
|-------|----------|--------|-------|
| Research / Spike | Xd | 1 | Validate assumptions |
| Proof of Concept | Xd | 1-2 | Build minimal working version |
| Implementation | Xd | X | Full build |
| Migration | Xd | X | Move from current to new |
| Testing | Xd | X | Verify behavior preserved |
| Documentation | Xd | 1 | Update docs, runbooks |
| Rollout | Xd | X | Deploy, monitor, iterate |

Total: X person-days (~X weeks with Y people)
```

### 3c. Cost Analysis
```
| Cost Type | Current (Option A) | Proposed (Option B) | Alternative (Option C) |
|-----------|--------------------|--------------------|-----------------------|
| Monthly service cost | $X | $X | $X |
| Engineering effort (one-time) | $0 (already done) | $X (Y days × rate) | $X |
| Ongoing maintenance | X hrs/month | X hrs/month | X hrs/month |
| Training / ramp-up | $0 | $X | $X |
| Migration cost | $0 | $X | $X |
| Risk cost (if things go wrong) | $X | $X | $X |
| **Total Year 1** | **$X** | **$X** | **$X** |
| **Total Year 2+** | **$X/yr** | **$X/yr** | **$X/yr** |
```

### 3d. Risk Assessment

For each option:
```
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| [Risk 1] | High/Med/Low | High/Med/Low | [How to reduce] |
| [Risk 2] | High/Med/Low | High/Med/Low | [How to reduce] |
| [Risk 3] | High/Med/Low | High/Med/Low | [How to reduce] |
```

Common risks to evaluate:
- **Vendor lock-in** — how hard is it to switch away later?
- **Team knowledge** — does only one person understand this?
- **Scalability** — will this still work at 10x current scale?
- **Reliability** — what's the uptime/SLA guarantee?
- **Security** — does this introduce new attack vectors?
- **Data integrity** — could data be lost or corrupted during migration?
- **Rollback** — if it fails, how hard is it to revert?
- **Compatibility** — does it break existing integrations?
- **Maintenance burden** — will this add ongoing toil?
- **Compliance** — does this affect regulatory compliance (GDPR, SOC2, HIPAA)?

### 3e. Impact Analysis
```
| Area | Impact | Details |
|------|--------|---------|
| **Codebase** | X files changed | [which modules/services] |
| **API contracts** | Breaking / Non-breaking | [what changes for consumers] |
| **Database** | Schema change / No change | [migrations needed] |
| **Infrastructure** | New services / Config change | [what needs to be provisioned] |
| **CI/CD** | Pipeline changes needed | [new steps, env vars, secrets] |
| **Monitoring** | New dashboards/alerts | [what to monitor] |
| **Documentation** | Docs to update | [README, API docs, runbooks] |
| **Team workflows** | Process changes | [how daily work changes] |
| **Other teams** | Affected / Not affected | [who needs to know] |
| **End users** | Visible / Invisible | [UX changes, downtime] |
```

## 4. Comparison Matrix

### Side-by-Side Comparison
```
| Criteria | Weight | Option A (Status Quo) | Option B (Proposed) | Option C (Alternative) |
|----------|--------|-----------------------|--------------------|-----------------------|
| **Solves the problem** | 25% | ⭐⭐ (2/5) | ⭐⭐⭐⭐ (4/5) | ⭐⭐⭐ (3/5) |
| **Implementation effort** | 20% | ⭐⭐⭐⭐⭐ (5/5) | ⭐⭐ (2/5) | ⭐⭐⭐ (3/5) |
| **Ongoing cost** | 15% | ⭐⭐⭐ (3/5) | ⭐⭐⭐⭐ (4/5) | ⭐⭐⭐ (3/5) |
| **Risk** | 15% | ⭐⭐⭐⭐ (4/5) | ⭐⭐ (2/5) | ⭐⭐⭐ (3/5) |
| **Scalability** | 10% | ⭐⭐ (2/5) | ⭐⭐⭐⭐ (4/5) | ⭐⭐⭐⭐ (4/5) |
| **Team experience** | 10% | ⭐⭐⭐⭐⭐ (5/5) | ⭐⭐ (2/5) | ⭐⭐⭐ (3/5) |
| **Reversibility** | 5% | ⭐⭐⭐⭐⭐ (5/5) | ⭐⭐ (2/5) | ⭐⭐⭐ (3/5) |
| **Weighted Score** | 100% | **3.25** | **3.10** | **3.10** |
```

### Pros & Cons Summary

For each option:
```
Option B: [Proposed Change]

✅ Pros:
- [Specific, quantified benefit 1]
- [Specific, quantified benefit 2]
- [Specific, quantified benefit 3]

❌ Cons:
- [Specific, quantified drawback 1]
- [Specific, quantified drawback 2]
- [Specific, quantified drawback 3]

⚠️ Unknowns:
- [Thing we don't know yet that could change the analysis]
- [Assumption that needs validation]
```

## 5. Recommendation

### Decision Framework

Use this framework to make the recommendation:
```
IF the proposal clearly solves a real problem
AND the effort is justified by the benefit
AND the risks are manageable
AND the team has capacity
→ RECOMMEND: Proceed

IF the proposal solves a real problem
BUT has significant unknowns
→ RECOMMEND: Spike first (time-boxed investigation)

IF the proposal is a "nice to have"
AND requires significant effort
→ RECOMMEND: Defer (revisit in X months)

IF the proposal introduces more risk than it reduces
OR the current solution is adequate
→ RECOMMEND: Keep current approach
```

### Recommendation Format
```
## Recommendation

**Decision: [Go / Spike First / Defer / Don't Do]**

**Recommended option: [Option X — name]**

**Reasoning:**
[2-3 sentences explaining why this option is best given the constraints]

**Key factors:**
1. [Most important reason]
2. [Second most important reason]
3. [Third most important reason]

**What we'd gain:**
- [Concrete benefit 1]
- [Concrete benefit 2]

**What we'd give up or risk:**
- [Concrete trade-off 1]
- [Concrete trade-off 2]

**Conditions for success:**
- [What needs to be true for this to work]
- [What we should monitor]
```

## 6. Next Steps

### If Recommendation is "Go"
```
## Next Steps

1. [ ] [Immediate action — e.g., create implementation tickets]
2. [ ] [Technical step — e.g., set up staging environment]
3. [ ] [People step — e.g., assign team, notify stakeholders]
4. [ ] [Validation step — e.g., run POC, benchmark]
5. [ ] [Documentation — e.g., update architecture docs]

**Timeline:**
- Week 1: [milestone]
- Week 2: [milestone]
- Week 3: [milestone]

**Success metrics:**
- [How we'll know this was the right decision]
- [Measurable outcome 1]
- [Measurable outcome 2]

**Rollback trigger:**
- [Condition under which we'd revert]
```

### If Recommendation is "Spike First"
```
## Next Steps: Time-Boxed Spike

**Duration:** [X hours / X days]
**Owner:** [Who will do the investigation]
**Deadline:** [Date]

**Questions to answer:**
1. [Specific question the spike should answer]
2. [Specific question]
3. [Specific question]

**Deliverable:**
A brief write-up (saved to project-decisions/) with:
- Answers to the above questions
- Working proof of concept (if applicable)
- Updated effort estimate
- Go / No-Go recommendation

**Decision checkpoint:** [Date] — review spike results and make final decision
```

### If Recommendation is "Defer"
```
## Next Steps: Defer

**Revisit date:** [Date — e.g., next quarter planning]
**Trigger to reconsider earlier:** [Condition — e.g., if current tool costs exceed $X/month]

**What to do now:**
1. [ ] Document this analysis for future reference (this document)
2. [ ] [Minor improvement to current approach if applicable]
3. [ ] Add to quarterly review agenda
```

### If Recommendation is "Don't Do"
```
## Next Steps: Keep Current Approach

**Reasoning documented above.**

**What to do instead:**
1. [ ] [Alternative small improvement]
2. [ ] [Address the underlying need differently]
3. [ ] Share this analysis with the proposer with context
```

## 7. Save the Output

Always save the complete analysis to a file:
```bash
# Create the output file
# Format: project-decisions/YYYY-MM-DD-kebab-case-topic.md

mkdir -p project-decisions

cat > project-decisions/$(date +%Y-%m-%d)-[topic].md << 'EOF'
# Tech Decision: [Title]

**Date:** [YYYY-MM-DD]
**Status:** [Proposed / Accepted / Rejected / Superseded]
**Proposed by:** [Name/Team]
**Decision makers:** [Names/Roles]
**Stakeholders:** [Who needs to know]

---

[Full analysis from sections 2-6 above]

---

## Decision Log

| Date | Event | By |
|------|-------|----|
| [Date] | Decision proposed | [Name] |
| [Date] | Analysis completed | [Name] |
| [Date] | Decision made: [outcome] | [Name] |
| [Date] | Implementation started | [Name] |
| [Date] | Implementation completed | [Name] |
| [Date] | Reviewed: [outcome validated/revised] | [Name] |

EOF
```

### File Naming Convention
```
project-decisions/
├── 2026-01-15-migrate-to-postgres.md
├── 2026-01-28-bigquery-vs-looker-studio.md
├── 2026-02-03-adopt-graphql.md
├── 2026-02-10-switch-email-provider.md
├── 2026-02-19-agent-for-analytics.md
└── README.md
```

### Auto-Generate Index

After saving the decision, update or create an index file:
```bash
# Generate index of all decisions
echo "# Project Decisions\n" > project-decisions/README.md
echo "| Date | Decision | Status |" >> project-decisions/README.md
echo "|------|----------|--------|" >> project-decisions/README.md

for f in project-decisions/2*.md; do
  date=$(basename "$f" | cut -d'-' -f1-3)
  title=$(head -1 "$f" | sed 's/^# //' | sed 's/^Tech Decision: //')
  status=$(grep "^**Status:**" "$f" | sed 's/.*: //' | sed 's/\*//g')
  echo "| $date | [$title](./$( basename $f )) | $status |" >> project-decisions/README.md
done
```

## Output Document Template

The saved file should follow this exact structure:
```markdown
# Tech Decision: [Clear Title]

**Date:** YYYY-MM-DD
**Status:** Proposed
**Proposed by:** [Name]
**Decision makers:** [Names]

---

## Context

[What's happening, why this decision came up, what triggered it]

## Decision

[What are we deciding between?]

## Options Considered

### Option A: [Status Quo — Current Approach]

**Description:** [How things work today]

✅ Pros:
- [Pro 1]
- [Pro 2]

❌ Cons:
- [Con 1]
- [Con 2]

**Effort:** None (already in place)
**Cost:** [Current ongoing cost]
**Risk:** Low

---

### Option B: [Proposed Change]

**Description:** [What would change]

✅ Pros:
- [Pro 1]
- [Pro 2]

❌ Cons:
- [Con 1]
- [Con 2]

**Effort:** [X person-days]
**Cost:** [One-time + ongoing]
**Risk:** [Low/Medium/High]

---

### Option C: [Alternative / Hybrid]

**Description:** [Alternative approach]

✅ Pros:
- [Pro 1]
- [Pro 2]

❌ Cons:
- [Con 1]
- [Con 2]

**Effort:** [X person-days]
**Cost:** [One-time + ongoing]
**Risk:** [Low/Medium/High]

---

## Comparison Matrix

| Criteria | Weight | Option A | Option B | Option C |
|----------|--------|----------|----------|----------|
| Solves the problem | 25% | X/5 | X/5 | X/5 |
| Implementation effort | 20% | X/5 | X/5 | X/5 |
| Ongoing cost | 15% | X/5 | X/5 | X/5 |
| Risk | 15% | X/5 | X/5 | X/5 |
| Scalability | 10% | X/5 | X/5 | X/5 |
| Team experience | 10% | X/5 | X/5 | X/5 |
| Reversibility | 5% | X/5 | X/5 | X/5 |
| **Weighted Score** | | **X.XX** | **X.XX** | **X.XX** |

## Recommendation

**Decision: [Go / Spike First / Defer / Don't Do]**

**Recommended option: [Option X]**

[Reasoning — 2-3 paragraphs explaining why]

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| [Risk 1] | Med | High | [Approach] |
| [Risk 2] | Low | High | [Approach] |

## Next Steps

1. [ ] [Action item 1]
2. [ ] [Action item 2]
3. [ ] [Action item 3]

**Timeline:**
- [Week 1 milestone]
- [Week 2 milestone]

**Success metrics:**
- [Metric 1]
- [Metric 2]

---

## Decision Log

| Date | Event | By |
|------|-------|----|
| YYYY-MM-DD | Decision proposed | [Name] |
```

## Adaptation Rules

- **Always save to file** — every analysis gets persisted in `project-decisions/`
- **Always compare 3+ options** — never evaluate a proposal in isolation
- **Be specific** — "saves 2 hours/week" not "saves time"
- **Quantify when possible** — costs in dollars, effort in days, risk with likelihood
- **Stay neutral** — present facts, let the recommendation follow from the analysis
- **Scan the codebase** — check what's actually in use before assuming
- **Consider the team** — a technically superior option that nobody knows how to use is a bad choice
- **Account for migration** — switching costs are real and often underestimated
- **Include rollback plan** — every "Go" recommendation needs an exit strategy
- **Update the index** — keep the README.md in project-decisions/ current

## Summary

End every tech decision analysis with:
1. **One-line recommendation** — Go / Spike / Defer / Don't Do
2. **Recommended option** — which option and why (one sentence)
3. **Effort estimate** — how long the recommended path takes
4. **Biggest risk** — the single most important risk to watch
5. **Next action** — the one thing to do right now
6. **File saved** — confirm the decision document location
