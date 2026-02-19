---
name: build-vs-buy
description: Compares building a solution internally versus buying/adopting a third-party tool, service, or library. Analyzes cost (total cost of ownership), effort, risk, vendor lock-in, maintenance burden, feature fit, and strategic alignment. Produces a structured recommendation. Saves output to project-decisions/ folder. Use when the user says "build vs buy", "build or buy", "should we build this ourselves", "should we use a service for this", "roll our own or use", "make vs buy", "build in-house or", "vendor comparison", "evaluate tools for", or "should we use X or build our own".
allowed-tools: Read, Grep, Glob, Bash
---

# Build vs Buy Skill

When evaluating whether to build in-house or adopt an external solution, follow this structured process. The goal is to make a clear, data-driven recommendation that accounts for the full lifecycle — not just the initial build.

**IMPORTANT**: Always save the output as a markdown file in the `project-decisions/` directory at the project root. Create the directory if it doesn't exist.

## 0. Output Setup
```bash
# Create project-decisions directory if it doesn't exist
mkdir -p project-decisions

# File will be saved as:
# project-decisions/YYYY-MM-DD-build-vs-buy-[kebab-case-topic].md
# Example: project-decisions/2026-02-19-build-vs-buy-auth-system.md
```

## 1. Frame the Problem

### Define the Need

Before comparing options, get crystal clear on what you actually need:
```
What problem are we solving?
[Clear problem statement — not the solution, the problem]

Who needs it?
[Target users/systems — internal team, end users, API consumers]

What are the hard requirements? (must have)
1. [Requirement 1]
2. [Requirement 2]
3. [Requirement 3]

What are the soft requirements? (nice to have)
1. [Nice to have 1]
2. [Nice to have 2]

What are the constraints?
- Budget: [$ amount or range]
- Timeline: [when do we need this by?]
- Team capacity: [available engineering time]
- Compliance: [GDPR, SOC2, HIPAA, PCI, etc.]
- Scale: [users, requests, data volume]

What does success look like?
[Measurable outcome — e.g., "auth flow takes < 200ms, supports 10K concurrent users"]
```

### Check What Already Exists
```bash
# Do we already have something in this area?
grep -rn "[feature-keyword]" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules\|\.git\|test\|spec" | head -20

# Check existing dependencies
cat package.json 2>/dev/null | grep -i "[keyword]"
cat requirements.txt pyproject.toml 2>/dev/null | grep -i "[keyword]"
cat docker-compose.yml 2>/dev/null | grep -i "[keyword]"

# Check for existing integrations
grep -rn "api_key\|API_KEY\|client_id\|CLIENT_ID" --include="*.ts" --include="*.js" --include="*.py" --include="*.env*" . 2>/dev/null | grep -v "node_modules\|\.git" | grep -i "[keyword]"

# Check for previous decisions on this topic
ls project-decisions/ 2>/dev/null | grep -i "[keyword]"
grep -rn "[keyword]" --include="*.md" docs/ project-decisions/ 2>/dev/null
```

## 2. Define the Options

Always evaluate at least 3 options:
```
Option A: BUILD — Build in-house from scratch
Option B: BUY — Adopt [specific vendor/tool 1]
Option C: BUY — Adopt [specific vendor/tool 2]
Option D: HYBRID — Build core + use vendor for [specific part]
Option E: OPEN SOURCE — Use [OSS solution] and self-host
```

### Common Option Patterns

| Pattern | When It Works Best |
|---------|-------------------|
| **Build** | Core differentiator, unique requirements, full control needed, team has expertise |
| **Buy (SaaS)** | Commodity feature, time-to-market critical, team lacks expertise, managed service preferred |
| **Buy (Licensed)** | On-premise required, compliance restrictions, one-time cost preferred |
| **Open Source (managed)** | Need flexibility + support, community ecosystem matters |
| **Open Source (self-hosted)** | Cost-sensitive, team has ops capability, customization needed |
| **Hybrid** | Core logic is custom, but commodity parts (email, payments, auth) are bought |

## 3. Evaluate Each Option

### 3a. Feature Fit Analysis
```
| Requirement | Weight | Build | Buy (Vendor A) | Buy (Vendor B) | OSS |
|-------------|--------|-------|----------------|----------------|-----|
| [Must have 1] | 🔴 Must | ✅ Custom | ✅ Yes | ✅ Yes | ⚠️ Partial |
| [Must have 2] | 🔴 Must | ✅ Custom | ✅ Yes | ❌ No | ✅ Yes |
| [Must have 3] | 🔴 Must | ✅ Custom | ⚠️ Workaround | ✅ Yes | ✅ Yes |
| [Nice to have 1] | 🟡 Want | ❌ Not in v1 | ✅ Yes | ✅ Yes | ⚠️ Plugin |
| [Nice to have 2] | 🟡 Want | ❌ Not in v1 | ✅ Yes | ❌ No | ✅ Yes |
| [Nice to have 3] | 🟢 Nice | ❌ Future | ✅ Yes | ✅ Yes | ❌ No |
| **Feature Score** | | **3/3 must** | **2.5/3 must** | **2/3 must** | **2.5/3 must** |
```

Legend:
- ✅ Fully supported
- ⚠️ Partially supported or workaround needed
- ❌ Not supported
- 🔧 Requires customization

### 3b. Total Cost of Ownership (TCO) — 3 Year View

#### BUILD
```
| Cost Category | Year 1 | Year 2 | Year 3 | Total |
|---------------|--------|--------|--------|-------|
| **Initial Development** | | | | |
| Engineering (X devs × Y weeks × $rate) | $X | — | — | $X |
| Design / UX | $X | — | — | $X |
| Testing / QA | $X | — | — | $X |
| **Infrastructure** | | | | |
| Servers / Cloud | $X | $X | $X | $X |
| Database / Storage | $X | $X | $X | $X |
| Monitoring / Logging | $X | $X | $X | $X |
| **Ongoing Maintenance** | | | | |
| Bug fixes (X hrs/month × $rate) | $X | $X | $X | $X |
| Feature enhancements | $X | $X | $X | $X |
| Security patches | $X | $X | $X | $X |
| Dependency updates | $X | $X | $X | $X |
| On-call / operations | $X | $X | $X | $X |
| **Hidden Costs** | | | | |
| Knowledge transfer / documentation | $X | $X | — | $X |
| Opportunity cost (team not building features) | $X | $X | $X | $X |
| Recruitment (if specialized skills needed) | $X | — | — | $X |
| **TOTAL BUILD** | **$X** | **$X** | **$X** | **$X** |
```

#### BUY
```
| Cost Category | Year 1 | Year 2 | Year 3 | Total |
|---------------|--------|--------|--------|-------|
| **Subscription / License** | | | | |
| Base subscription (tier × price) | $X | $X | $X | $X |
| Per-user / per-seat cost | $X | $X | $X | $X |
| Overage charges (estimated) | $X | $X | $X | $X |
| **Integration** | | | | |
| Integration development | $X | — | — | $X |
| Data migration | $X | — | — | $X |
| Custom configuration | $X | — | — | $X |
| **Ongoing** | | | | |
| Integration maintenance | $X | $X | $X | $X |
| Vendor management | $X | $X | $X | $X |
| Training | $X | $X | — | $X |
| **Hidden Costs** | | | | |
| Price increases (assume X%/year) | — | $X | $X | $X |
| Migration cost (if switching later) | — | — | $X | $X |
| Workarounds for missing features | $X | $X | $X | $X |
| **TOTAL BUY** | **$X** | **$X** | **$X** | **$X** |
```

#### OPEN SOURCE (Self-Hosted)
```
| Cost Category | Year 1 | Year 2 | Year 3 | Total |
|---------------|--------|--------|--------|-------|
| **Setup** | | | | |
| Deployment / Configuration | $X | — | — | $X |
| Integration development | $X | — | — | $X |
| Customization | $X | — | — | $X |
| **Infrastructure** | | | | |
| Servers / Cloud | $X | $X | $X | $X |
| Database / Storage | $X | $X | $X | $X |
| **Ongoing** | | | | |
| Upgrades / Patches | $X | $X | $X | $X |
| Operations / Monitoring | $X | $X | $X | $X |
| Bug fixes / Contributions | $X | $X | $X | $X |
| Commercial support (optional) | $X | $X | $X | $X |
| **TOTAL OSS** | **$X** | **$X** | **$X** | **$X** |
```

#### TCO Comparison Summary
```
| | Year 1 | Year 2 | Year 3 | 3-Year Total |
|---|--------|--------|--------|-------------|
| BUILD | $X | $X | $X | $X |
| BUY (Vendor A) | $X | $X | $X | $X |
| BUY (Vendor B) | $X | $X | $X | $X |
| OSS (self-hosted) | $X | $X | $X | $X |
| HYBRID | $X | $X | $X | $X |
```

### 3c. Time to Value
```
| Option | Time to MVP | Time to Full Feature | Notes |
|--------|------------|---------------------|-------|
| BUILD | X weeks | X months | Need design + dev + test |
| BUY (Vendor A) | X days | X weeks | Integration time |
| BUY (Vendor B) | X days | X weeks | Integration time |
| OSS | X weeks | X weeks | Setup + customization |
| HYBRID | X weeks | X weeks | Fastest for core, buy rest |
```

### 3d. Risk Assessment

#### Build Risks
```
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Takes longer than estimated | High | High | Phased delivery, MVP first |
| Team lacks domain expertise | Med | High | Hire / consult expert |
| Ongoing maintenance burden | High | Med | Dedicated on-call rotation |
| Key developer leaves | Med | High | Documentation, pair programming |
| Security vulnerabilities | Med | High | Security review, pen testing |
| Under-engineered at scale | Med | High | Load testing early |
| Opportunity cost (not building features) | High | Med | Track feature delays |
```

#### Buy Risks
```
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Vendor goes out of business | Low | Critical | Evaluate funding, data export |
| Vendor raises prices significantly | Med | High | Negotiate multi-year, plan exit |
| Vendor doesn't add features we need | Med | Med | Evaluate roadmap, API extensibility |
| Vendor has outage (downtime for us) | Med | High | Check SLA, build fallback |
| Data sovereignty / compliance issue | Low | Critical | Verify data residency, certifications |
| Vendor lock-in (hard to migrate) | High | High | Abstract integration layer |
| Integration is harder than expected | Med | Med | POC before committing |
| Vendor API changes / deprecation | Med | Med | Pin versions, monitor changelog |
| Data breach at vendor | Low | Critical | Review security practices, DPA |
```

#### OSS Risks
```
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Project becomes unmaintained | Med | High | Check commit frequency, community |
| Security vulnerability not patched | Med | High | Monitor CVEs, have fork plan |
| Major version upgrade breaks things | Med | Med | Pin versions, test upgrades |
| Missing feature requires forking | Med | Med | Contribute upstream, evaluate first |
| Operational burden higher than expected | High | Med | Consider managed hosting |
| Community support insufficient | Med | Low | Budget for commercial support |
```

### 3e. Vendor Lock-In Assessment
```
| Lock-in Factor | Low Risk | Medium Risk | High Risk |
|---------------|----------|-------------|-----------|
| **Data portability** | Standard format export, API access to all data | Export available but lossy | No bulk export, proprietary format |
| **API standards** | Open standards (REST, GraphQL, OAuth) | Proprietary but documented | Undocumented, SDK-only |
| **Integration depth** | Thin wrapper, easy to swap | Moderate coupling, ~1 week to swap | Deep integration, months to migrate |
| **Data format** | Standard (JSON, CSV, SQL) | Convertible with effort | Proprietary, vendor-specific |
| **Contract terms** | Month-to-month, no lock-in | Annual with early exit option | Multi-year, penalty for early exit |
| **Feature dependency** | Could rebuild core in 1-2 weeks | Would take 1-2 months to replace | 6+ months to replicate |
| **Knowledge dependency** | Team can maintain alternatives | Some retraining needed | Vendor-certified skills required |
```

Score:
```
Low Lock-in (1-2):    Easy to switch, minimal migration effort
Medium Lock-in (3-4): Switching is feasible but painful (weeks)
High Lock-in (5):     Switching is a major project (months)
```

### 3f. Strategic Alignment
```
| Question | Build | Buy |
|----------|-------|-----|
| Is this our core differentiator? | ✅ Build what makes you unique | ❌ Don't buy your advantage |
| Is this a commodity? | ❌ Don't reinvent the wheel | ✅ Buy commodities |
| Does this give us competitive advantage? | ✅ If yes, build | ❌ If no, buy |
| Is the team's time better spent elsewhere? | ❌ Building costs opportunity | ✅ Buying frees the team |
| Do we need full control? | ✅ Build for full control | ❌ Limited control with vendor |
| Is speed to market critical? | ❌ Building takes time | ✅ Buying is faster |
| Do we have the expertise? | ✅ If yes, leverage it | ✅ If no, buy expertise |
| Will requirements change frequently? | ✅ Build for flexibility | ❌ Vendor may not keep up |
| Is this regulated? | ⚠️ Build if specific compliance needed | ⚠️ Buy if vendor is certified |
| Is this a long-term need? | ✅ Build amortizes over time | ⚠️ Buy costs compound |
```

### 3g. Team & Organizational Fit
```
| Factor | Build | Buy |
|--------|-------|-----|
| **Current team skills** | [Does team have the expertise?] | [Can team integrate and maintain?] |
| **Hiring impact** | [Need to hire specialists?] | [Need vendor management skills?] |
| **On-call impact** | [New on-call rotation needed?] | [Vendor handles uptime?] |
| **Knowledge concentration** | [Bus factor risk?] | [Vendor provides docs/support?] |
| **Team morale** | [Will team enjoy building this?] | [Will team resent "not invented here"?] |
| **Growth opportunity** | [Learning opportunity for team?] | [Frees team for higher-value work?] |
```

## 4. Comparison Matrix
```
| Criteria | Weight | Build | Buy (A) | Buy (B) | OSS | Hybrid |
|----------|--------|-------|---------|---------|-----|--------|
| **Feature fit** | 20% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Total cost (3yr)** | 20% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Time to value** | 15% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Risk** | 15% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Lock-in** | 10% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Maintenance burden** | 10% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Strategic alignment** | 5% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Team fit** | 5% | X/5 | X/5 | X/5 | X/5 | X/5 |
| **Weighted Score** | 100% | **X.XX** | **X.XX** | **X.XX** | **X.XX** | **X.XX** |
```

## 5. Vendor Evaluation (When Buying)

### Vendor Health Check

For each vendor under consideration:
```
| Factor | Vendor A | Vendor B |
|--------|----------|----------|
| **Company age** | X years | X years |
| **Funding / Revenue** | [Series X / $Xm ARR / Public] | [Series X / $Xm ARR] |
| **Customer count** | X customers | X customers |
| **Notable customers** | [Names] | [Names] |
| **Employee count** | X | X |
| **Growth trend** | [Growing/Stable/Declining] | [Growing/Stable/Declining] |
| **Certifications** | [SOC2, ISO 27001, HIPAA, etc.] | [SOC2, etc.] |
| **Uptime SLA** | X% | X% |
| **Support quality** | [24/7 / Business hours / Community] | [24/7 / Business hours] |
| **Documentation quality** | [Excellent / Good / Poor] | [Good / Poor] |
| **API quality** | [REST / GraphQL / SDK] | [REST / SDK] |
| **Data residency** | [Regions available] | [Regions available] |
| **Data export** | [Format, completeness] | [Format, completeness] |
| **Contract flexibility** | [Monthly / Annual / Multi-year] | [Monthly / Annual] |
| **Pricing transparency** | [Public / Quote-based] | [Public / Quote-based] |
| **Community / Ecosystem** | [Size, activity] | [Size, activity] |
| **Recent incidents** | [Any notable outages?] | [Any notable outages?] |
```

### Vendor Red Flags

Watch out for:
```
🔴 No SOC2 / ISO 27001 certification
🔴 No data export capability
🔴 Requires multi-year contract with no exit clause
🔴 Pricing not transparent (must "talk to sales")
🔴 No public status page or incident history
🔴 Documentation is outdated or incomplete
🔴 Single-region only (no redundancy)
🔴 Startup with < 2 years and no significant funding
🟡 No self-serve trial (can't evaluate before buying)
🟡 Major version changes frequently (API instability)
🟡 Small community / ecosystem
🟡 Support only via email, slow response times
```

## 6. Recommendation Framework

### Decision Heuristics
```
BUILD when:
✅ This IS your core product / differentiator
✅ No vendor meets > 80% of hard requirements
✅ You need full control of the data and logic
✅ Team has deep expertise and available capacity
✅ Requirements are highly unique to your business
✅ Long-term cost of buying exceeds building
✅ Regulatory requirements prevent external vendors

BUY when:
✅ This is NOT your core differentiator (commodity)
✅ A vendor meets > 90% of requirements out of the box
✅ Time-to-market is critical
✅ Team lacks expertise or capacity
✅ The vendor is well-established and financially stable
✅ TCO over 3 years is lower than building
✅ Vendor provides better security/compliance than you could build

OPEN SOURCE when:
✅ Good OSS solution exists with active community
✅ Team has ops capability to self-host
✅ Need vendor-independence but don't need to build from scratch
✅ Customization is important
✅ Budget is constrained but team time is available

HYBRID when:
✅ Core logic must be custom but supporting services are commodity
✅ "Build the brain, buy the plumbing"
✅ Need speed for some parts, control for others
✅ Can clearly separate custom from commodity components
```

### Recommendation Format
```
## Recommendation

**Decision: [BUILD / BUY (Vendor X) / OPEN SOURCE (Tool X) / HYBRID]**

**Confidence: [High / Medium / Low]**

**One-line rationale:**
[Single sentence — e.g., "Auth is not our differentiator, Clerk meets 95% of requirements,
and building would cost 3x more over 3 years while delaying feature work by 6 weeks."]

**Key reasons:**
1. [Most important reason for this choice]
2. [Second most important reason]
3. [Third most important reason]

**What we gain:**
- [Concrete benefit 1]
- [Concrete benefit 2]
- [Concrete benefit 3]

**What we give up:**
- [Concrete trade-off 1]
- [Concrete trade-off 2]

**Conditions that would change this decision:**
- [If X changes, we should reconsider — e.g., "if we outgrow the vendor's pricing tier"]
- [If Y happens, we should switch — e.g., "if vendor is acquired or raises prices > 50%"]
```

## 7. Implementation Plan

### If BUILD
```
## Implementation Plan: Build

Phase 1: Design & Spike (Week 1-2)
- [ ] Finalize technical design
- [ ] Validate key technical assumptions
- [ ] Set up infrastructure

Phase 2: MVP (Week 3-6)
- [ ] Build core functionality
- [ ] Internal testing
- [ ] Documentation

Phase 3: Harden (Week 7-8)
- [ ] Error handling, edge cases
- [ ] Security review
- [ ] Performance testing
- [ ] Monitoring & alerting

Phase 4: Launch (Week 9)
- [ ] Staged rollout
- [ ] Monitor metrics
- [ ] On-call handoff

Ongoing:
- [ ] X hrs/week maintenance budget
- [ ] Quarterly security review
- [ ] Annual architecture review
```

### If BUY
```
## Implementation Plan: Buy

Phase 1: Procurement (Week 1)
- [ ] Finalize vendor contract
- [ ] Provisioning and access
- [ ] Security review / DPA signed

Phase 2: Integration (Week 2-3)
- [ ] Build integration layer (abstracted, swappable)
- [ ] Data migration (if applicable)
- [ ] Configuration and customization

Phase 3: Testing (Week 4)
- [ ] Integration testing
- [ ] Performance testing
- [ ] Failover/fallback testing

Phase 4: Launch (Week 5)
- [ ] Staged rollout
- [ ] Monitor metrics
- [ ] Team training

Ongoing:
- [ ] Monthly vendor relationship check
- [ ] Quarterly cost review
- [ ] Annual renewal evaluation
```

### If HYBRID
```
## Implementation Plan: Hybrid

Phase 1: Buy components (Week 1-2)
- [ ] Procure vendor for [commodity part]
- [ ] Integrate and test vendor component

Phase 2: Build core (Week 2-6)
- [ ] Build custom [core differentiator]
- [ ] Connect to vendor components
- [ ] Integration testing

Phase 3: Launch (Week 7-8)
- [ ] End-to-end testing
- [ ] Staged rollout
- [ ] Documentation

Ongoing:
- [ ] Maintain custom code (X hrs/week)
- [ ] Manage vendor relationship
- [ ] Monitor integration health
```

### Exit Strategy (Always Required)
```
## Exit Strategy

If we need to switch from [chosen option] in the future:

**Data portability:**
- [How to export data — format, completeness, timeline]

**Integration abstraction:**
- [How our code is structured to make switching easier]
- [Adapter/wrapper pattern used? Interface defined?]

**Migration plan:**
- [High-level steps to switch to alternative]
- [Estimated effort: X person-weeks]
- [Can run both in parallel during migration? Yes/No]

**Trigger points for re-evaluation:**
- [ ] Vendor raises prices by > X%
- [ ] Vendor is acquired
- [ ] Our scale exceeds vendor's capability
- [ ] Vendor has > X hours of downtime per quarter
- [ ] Compliance requirements change
```

## Output Document Template

Save to `project-decisions/YYYY-MM-DD-build-vs-buy-[topic].md`:
```markdown
# Build vs Buy: [Title]

**Date:** YYYY-MM-DD
**Status:** [Proposed / Accepted / Rejected]
**Decision makers:** [Names]
**Stakeholders:** [Names]

---

## Problem Statement

[What we need and why]

## Requirements

### Must Have
1. [Requirement]
2. [Requirement]

### Nice to Have
1. [Requirement]
2. [Requirement]

### Constraints
- Budget: [amount]
- Timeline: [deadline]
- Compliance: [requirements]

---

## Options Evaluated

### Option A: Build In-House
[Description, pros, cons]

### Option B: Buy — [Vendor Name]
[Description, pros, cons]

### Option C: [Alternative]
[Description, pros, cons]

---

## Feature Fit

[Feature comparison table]

---

## Total Cost of Ownership (3-Year)

[TCO tables for each option + summary comparison]

---

## Risk Assessment

[Risk tables for each option]

---

## Comparison Matrix

[Weighted scoring matrix]

---

## Vendor Evaluation

[Vendor health check — if buying]

---

## Recommendation

**Decision: [BUILD / BUY / OSS / HYBRID]**
**Confidence: [High / Medium / Low]**

[Reasoning — 2-3 paragraphs]

---

## Implementation Plan

[Phased plan for the recommended option]

---

## Exit Strategy

[How to switch if needed]

---

## Next Steps

1. [ ] [Action item]
2. [ ] [Action item]
3. [ ] [Action item]

---

## Decision Log

| Date | Event | By |
|------|-------|----|
| YYYY-MM-DD | Analysis started | [Name] |
| YYYY-MM-DD | Vendor demos completed | [Name] |
| YYYY-MM-DD | Decision made: [outcome] | [Name] |
| YYYY-MM-DD | Implementation started | [Name] |
```

After saving, update the project-decisions index:
```bash
echo "# Project Decisions\n" > project-decisions/README.md
echo "| Date | Decision | Type | Status |" >> project-decisions/README.md
echo "|------|----------|------|--------|" >> project-decisions/README.md

for f in project-decisions/2*.md; do
  date=$(basename "$f" | cut -d'-' -f1-3)
  title=$(head -1 "$f" | sed 's/^# //' | sed 's/^Build vs Buy: //' | sed 's/^Tech Decision: //' | sed 's/^Impact Analysis: //' | sed 's/^Scope Check: //' | sed 's/^Incident Report: //' | sed 's/^Tech Debt Report//')
  type="Other"
  echo "$f" | grep -q "build-vs-buy" && type="Build vs Buy"
  echo "$f" | grep -q "incident" && type="Incident Report"
  echo "$f" | grep -q "scope" && type="Scope Check"
  echo "$f" | grep -q "impact" && type="Impact Analysis"
  echo "$f" | grep -q "tech-debt" && type="Tech Debt Report"
  echo "$f" | grep -q -v "build-vs-buy\|incident\|scope\|impact\|tech-debt" && type="Tech Decision"
  status=$(grep "^**Status:**" "$f" | head -1 | sed 's/.*: //' | sed 's/\*//g')
  echo "| $date | [$title](./$(basename $f)) | $type | $status |" >> project-decisions/README.md
done
```

## Adaptation Rules

- **Always save to file** — every analysis gets persisted in `project-decisions/`
- **Always include 3-year TCO** — first-year cost is misleading, total lifecycle cost matters
- **Always include exit strategy** — even when buying, plan for the possibility of switching
- **Abstract integrations** — if buying, always recommend wrapping vendor in an adapter/interface
- **Check the codebase** — see what already exists before evaluating options
- **Quantify opportunity cost** — engineering time spent building is time NOT spent on features
- **Include hidden costs** — on-call, documentation, training, vendor management overhead
- **Evaluate vendor health** — a cheap vendor that goes bankrupt is expensive
- **Consider the team** — a perfect vendor that nobody knows how to integrate is imperfect
- **Think about scale** — a solution that works at current scale may not work at 10x
- **Be honest about "build" estimates** — teams consistently underestimate build cost by 2-3x
- **Default to buy for commodities** — auth, email, payments, monitoring are rarely worth building
- **Default to build for differentiators** — your core value proposition should be under your control

## Summary

End every build vs buy analysis with:
1. **Recommendation** — Build / Buy (Vendor) / OSS / Hybrid
2. **Confidence** — High / Medium / Low
3. **One-line rationale** — why this option wins
4. **3-year TCO** — cost comparison in one line
5. **Time to value** — when we'd have a working solution
6. **Biggest risk** — the single most important risk to watch
7. **Next action** — the one thing to do right now
8. **File saved** — confirm the document location
