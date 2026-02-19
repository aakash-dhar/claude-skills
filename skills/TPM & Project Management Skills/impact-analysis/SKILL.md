---
name: impact-analysis
description: Analyzes how a proposed change affects existing systems, services, teams, APIs, databases, infrastructure, and workflows. Maps blast radius, identifies dependencies, flags breaking changes, and produces a detailed impact report. Saves output to project-decisions/ folder. Use when the user says "what would break if", "impact analysis", "blast radius", "what does this affect", "who is impacted", "dependency check", "what breaks if we change", "ripple effects", or "change impact".
allowed-tools: Read, Grep, Glob, Bash
---

# Impact Analysis Skill

When analyzing the impact of a proposed change, follow this structured process. The goal is to map the full blast radius before anyone writes a line of code — catching surprises early is far cheaper than catching them in production.

**IMPORTANT**: Always save the output as a markdown file in the `project-decisions/` directory at the project root. Create the directory if it doesn't exist.

## 0. Output Setup
```bash
# Create project-decisions directory if it doesn't exist
mkdir -p project-decisions

# File will be saved as:
# project-decisions/YYYY-MM-DD-impact-[kebab-case-topic].md
```

## 1. Understand the Proposed Change

### Parse the Request

Extract from the question or discussion:

- **What's changing?** — specific code, system, API, database, service, or process
- **Why is it changing?** — new feature, bug fix, refactor, migration, deprecation
- **What's the desired outcome?** — what should be different after the change
- **What's NOT changing?** — explicitly call out what stays the same

### Classify the Change Type

| Change Type | Typical Blast Radius | Risk Level |
|-------------|---------------------|------------|
| **Config change** | Narrow — one service | 🟢 Low |
| **Bug fix** | Narrow — one function/module | 🟢 Low |
| **New feature (additive)** | Narrow to medium — new code, no existing changes | 🟡 Medium |
| **API change (backward compatible)** | Medium — consumers may need updates | 🟡 Medium |
| **API change (breaking)** | Wide — all consumers must update | 🔴 High |
| **Database schema change** | Wide — queries, models, migrations, backfill | 🔴 High |
| **Service migration/replacement** | Wide — integrations, config, monitoring, runbooks | 🔴 High |
| **Shared library update** | Wide — all consumers of the library | 🟠 High |
| **Infrastructure change** | Wide — deployment, networking, DNS, secrets | 🔴 High |
| **Authentication/authorization change** | Very wide — every authenticated endpoint | 🔴 Critical |
| **Data model/domain change** | Very wide — every layer of the application | 🔴 Critical |

## 2. Map the Blast Radius

### 2a. Code-Level Impact
```bash
# Find all files that reference the changing code
grep -rn "[changing-module-or-function]" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" --include="*.java" --include="*.rb" --include="*.php" src/ app/ lib/ 2>/dev/null | grep -v "node_modules\|\.git\|test\|spec\|mock"

# Find all imports of the changing module
grep -rn "import.*from.*[module]\|require.*[module]\|from [module] import" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules\|\.git"

# Count affected files
grep -rln "[changing-module-or-function]" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules\|\.git" | wc -l

# Find downstream dependencies (what uses the thing that uses the changing code)
# Build a two-level dependency chain
for file in $(grep -rln "[module]" --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules"); do
  basename "$file"
  grep -rln "$(basename $file .ts)\|$(basename $file .js)" --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules" | sed 's/^/  → /'
done

# Check for re-exports (the change may propagate through barrel files)
grep -rn "export.*from.*[module]\|module\.exports.*require.*[module]" --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules"

# Find interface/type usages (TypeScript)
grep -rn "interface\|type " [changing-file] 2>/dev/null | while read line; do
  typename=$(echo "$line" | grep -oP '(?:interface|type)\s+\K\w+')
  echo "Type: $typename"
  grep -rn "$typename" --include="*.ts" --include="*.tsx" src/ 2>/dev/null | grep -v "node_modules" | wc -l
  echo " references"
done
```

### 2b. API Impact
```bash
# Find API routes that use the changing code
grep -rn "[module-or-function]" --include="*.ts" --include="*.js" --include="*.py" src/api/ src/routes/ app/api/ 2>/dev/null

# List all endpoints in affected route files
for file in $(grep -rln "[module]" src/api/ src/routes/ 2>/dev/null); do
  echo "=== $file ==="
  grep -n "router\.\(get\|post\|put\|delete\|patch\)\|app\.\(get\|post\|put\|delete\|patch\)\|@app\.route\|@GetMapping\|@PostMapping" "$file" 2>/dev/null
done

# Check for API versioning
find . -path "*/api/v*" -o -path "*/v1/*" -o -path "*/v2/*" 2>/dev/null | head -20

# Check for OpenAPI/Swagger specs
find . -name "openapi*" -o -name "swagger*" -o -name "*.api.yaml" -o -name "*.api.json" 2>/dev/null | head -10

# Check for API consumers (external clients, mobile apps, other services)
grep -rn "fetch\|axios\|http\.\|HttpClient\|requests\." --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -i "[affected-endpoint]"
```

### 2c. Database Impact
```bash
# Find models/schemas related to the change
grep -rn "model\|schema\|entity\|@Entity\|@Table\|class.*Model\|class.*Schema" --include="*.ts" --include="*.js" --include="*.py" --include="*.java" src/db/ src/models/ src/entities/ app/models/ 2>/dev/null | grep -i "[keyword]"

# Find all queries that reference affected tables/columns
grep -rn "SELECT\|INSERT\|UPDATE\|DELETE\|FROM\|JOIN\|WHERE" --include="*.ts" --include="*.js" --include="*.py" --include="*.sql" src/ 2>/dev/null | grep -i "[table-or-column]"

# Find ORM queries referencing affected models
grep -rn "findOne\|findMany\|findAll\|create\|update\|delete\|where\|include\|select" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -i "[model]"

# Check for existing migrations
ls -la src/db/migrations/ db/migrations/ migrations/ prisma/migrations/ 2>/dev/null | tail -10

# Check migration history count
find . -path "*/migrations/*" -name "*.ts" -o -name "*.js" -o -name "*.py" -o -name "*.sql" 2>/dev/null | wc -l

# Check for views, functions, triggers that reference the table
grep -rn "CREATE VIEW\|CREATE FUNCTION\|CREATE TRIGGER" --include="*.sql" . 2>/dev/null | grep -i "[table]"

# Check for indexes on affected columns
grep -rn "INDEX\|@@index\|add_index\|createIndex" --include="*.ts" --include="*.js" --include="*.py" --include="*.sql" --include="*.prisma" src/ prisma/ 2>/dev/null | grep -i "[column]"
```

### 2d. Service & Integration Impact
```bash
# Find external service calls related to the change
grep -rn "fetch\|axios\|http\.\|requests\.\|HttpClient\|RestTemplate" --include="*.ts" --include="*.js" --include="*.py" --include="*.java" src/ 2>/dev/null | grep -i "[service-keyword]"

# Find message queue producers/consumers
grep -rn "publish\|subscribe\|emit\|on(\|producer\|consumer\|queue\|topic\|channel" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -i "[keyword]"

# Find webhook endpoints
grep -rn "webhook\|callback\|notify" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null

# Find gRPC/protobuf definitions
find . -name "*.proto" 2>/dev/null | xargs grep -l "[keyword]" 2>/dev/null

# Find GraphQL schema references
grep -rn "type \|input \|query \|mutation \|subscription " --include="*.graphql" --include="*.gql" . 2>/dev/null | grep -i "[keyword]"

# Check for shared libraries/packages (monorepo)
cat package.json 2>/dev/null | grep -E "workspace|lerna|nx|turbo"
find . -name "package.json" -maxdepth 3 | xargs grep -l "[module]" 2>/dev/null
```

### 2e. Infrastructure Impact
```bash
# Check Docker configuration
grep -rn "[keyword]" Dockerfile docker-compose.yml docker-compose.yaml 2>/dev/null

# Check CI/CD pipelines
grep -rn "[keyword]" .github/workflows/*.yml .gitlab-ci.yml Jenkinsfile bitbucket-pipelines.yml 2>/dev/null

# Check environment configuration
grep -rn "[keyword]" .env.example .env.sample .env.template 2>/dev/null
grep -rn "[keyword]" --include="*.yaml" --include="*.yml" --include="*.toml" --include="*.ini" config/ infra/ k8s/ terraform/ 2>/dev/null

# Check Kubernetes manifests
grep -rn "[keyword]" --include="*.yaml" --include="*.yml" k8s/ kubernetes/ helm/ 2>/dev/null

# Check Terraform/Infrastructure as Code
grep -rn "[keyword]" --include="*.tf" --include="*.tfvars" terraform/ infra/ 2>/dev/null

# Check nginx/reverse proxy config
grep -rn "[keyword]" --include="*.conf" --include="*.nginx" . 2>/dev/null

# Check monitoring/alerting config
grep -rn "[keyword]" --include="*.yaml" --include="*.yml" --include="*.json" monitoring/ alerts/ datadog/ grafana/ 2>/dev/null
```

### 2f. Test Impact
```bash
# Find tests that cover the changing code
grep -rn "[module-or-function]" --include="*.test.*" --include="*.spec.*" --include="test_*" --include="*_test.*" . 2>/dev/null | grep -v "node_modules"

# Count affected test files
grep -rln "[module-or-function]" --include="*.test.*" --include="*.spec.*" --include="test_*" . 2>/dev/null | grep -v "node_modules" | wc -l

# Find e2e/integration tests that might break
grep -rn "[keyword]" --include="*.test.*" --include="*.spec.*" --include="*.cy.*" tests/e2e/ tests/integration/ cypress/ playwright/ 2>/dev/null

# Find test fixtures/mocks that reference affected code
grep -rn "[keyword]" --include="*.ts" --include="*.js" --include="*.py" tests/fixtures/ tests/mocks/ tests/helpers/ __mocks__/ 2>/dev/null
```

### 2g. Documentation Impact
```bash
# Find docs referencing the changing code
grep -rn "[keyword]" --include="*.md" --include="*.mdx" --include="*.rst" --include="*.txt" docs/ README.md CONTRIBUTING.md 2>/dev/null

# Find API documentation
grep -rn "[keyword]" --include="*.yaml" --include="*.json" --include="*.md" docs/api/ openapi/ swagger/ 2>/dev/null

# Find inline documentation
grep -rn "@see\|@link\|@deprecated\|@since" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -i "[keyword]"

# Find ADRs/decision records referencing this area
grep -rn "[keyword]" --include="*.md" docs/adr/ docs/decisions/ project-decisions/ 2>/dev/null
```

## 3. Dependency Graph

### Direct Dependencies (Level 1)
```
Things the changing code DEPENDS ON (upstream):
- [dependency 1] — [how it's used]
- [dependency 2] — [how it's used]

Things that DEPEND ON the changing code (downstream):
- [dependent 1] — [how it uses this code]
- [dependent 2] — [how it uses this code]
```

### Transitive Dependencies (Level 2)
```
Things that depend on Level 1 dependents:
- [dependent 1] → [level 2 dependent]
- [dependent 2] → [level 2 dependent]
```

### Visual Dependency Map
```
                    ┌─────────────┐
                    │  Changing    │
                    │  Component   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼─────┐ ┌───▼─────┐
        │ Service A  │ │ API     │ │ Worker  │
        │ (direct)   │ │ (direct)│ │ (direct)│
        └─────┬──────┘ └───┬─────┘ └───┬─────┘
              │            │            │
         ┌────▼────┐  ┌───▼────┐  ┌───▼────┐
         │Frontend │  │Mobile  │  │Reports │
         │(indirect)│ │App     │  │Service │
         └─────────┘  │(indirect)│ │(indirect)│
                      └────────┘  └────────┘
```

## 4. Breaking Change Detection

### API Breaking Changes

| Change | Breaking? | Impact |
|--------|-----------|--------|
| Remove an endpoint | 🔴 Yes | All consumers must update |
| Remove a field from response | 🔴 Yes | Consumers relying on field will break |
| Rename a field | 🔴 Yes | All consumers must update field references |
| Change field type | 🔴 Yes | Type parsing will fail for consumers |
| Add required field to request | 🔴 Yes | Existing calls missing field will fail |
| Add optional field to request | 🟢 No | Backward compatible |
| Add field to response | 🟢 No | Consumers should ignore unknown fields |
| Add new endpoint | 🟢 No | Additive, no existing code affected |
| Change error format | 🟡 Maybe | If consumers parse error messages |
| Change status codes | 🟡 Maybe | If consumers check specific codes |
| Change pagination format | 🔴 Yes | All paginated consumers break |
| Add rate limiting | 🟡 Maybe | High-volume consumers may be affected |

### Database Breaking Changes

| Change | Breaking? | Impact |
|--------|-----------|--------|
| Add nullable column | 🟢 No | Existing queries unaffected |
| Add non-nullable column without default | 🔴 Yes | INSERT statements fail |
| Remove column | 🔴 Yes | SELECT, INSERT referencing column fail |
| Rename column | 🔴 Yes | All queries referencing column fail |
| Change column type | 🟡 Maybe | Depends on type compatibility |
| Add table | 🟢 No | Additive |
| Remove table | 🔴 Yes | All queries referencing table fail |
| Add index | 🟢 No | Performance improvement, no behavior change |
| Remove index | 🟡 Maybe | Performance degradation, possible timeouts |
| Add constraint | 🟡 Maybe | Existing data may violate constraint |
| Change primary key | 🔴 Yes | Foreign keys, joins, all references break |

### Shared Library Breaking Changes

| Change | Breaking? | Impact |
|--------|-----------|--------|
| Remove exported function | 🔴 Yes | All importers fail |
| Change function signature | 🔴 Yes | All callers must update |
| Change return type | 🔴 Yes | All consumers of return value break |
| Add optional parameter | 🟢 No | Backward compatible |
| Change default value | 🟡 Maybe | Behavior change for callers relying on default |
| Remove exported type | 🔴 Yes | TypeScript consumers fail to compile |
| Change enum values | 🔴 Yes | Switch statements and comparisons break |

## 5. Team & People Impact

### Who Needs to Know

| Team/Person | Why | Action Needed |
|-------------|-----|---------------|
| [Frontend team] | API contract changes | Update API calls |
| [Mobile team] | API contract changes | Update app, new release needed |
| [DevOps] | Infrastructure changes | Update config, pipelines |
| [QA] | Test plan changes | Update test cases, regression testing |
| [Product] | Feature behavior changes | Update docs, notify customers |
| [Security] | Auth/data changes | Security review |
| [Support] | User-facing changes | Update runbooks, train team |
| [Data team] | Schema changes | Update queries, dashboards, reports |
| [Partner teams] | API changes | Coordinate migration timeline |

### Communication Plan
```
| When | Who | Channel | Message |
|------|-----|---------|---------|
| Before starting | [team] | Slack #engineering | Heads up: we're changing X |
| Before deploy | [team] | Slack #deploys | Migration plan, expected downtime |
| During deploy | [team] | Slack #incidents | Status updates |
| After deploy | [team] | Slack #engineering | Completed, what to watch for |
| After validation | [stakeholders] | Email | Summary of changes |
```

## 6. Migration & Rollout Strategy

### If Breaking Changes Exist
```
Recommended rollout strategy:

Phase 1: Prepare (before any changes go live)
- [ ] Notify affected teams/consumers with timeline
- [ ] Create feature flag for gradual rollout
- [ ] Set up monitoring for affected endpoints/services
- [ ] Prepare rollback plan

Phase 2: Backward Compatible (deploy first)
- [ ] Add new code alongside old code
- [ ] Support both old and new format simultaneously
- [ ] Deploy and verify new code works

Phase 3: Migrate (gradual cutover)
- [ ] Migrate consumers one by one to new format
- [ ] Monitor error rates per consumer
- [ ] Keep old code path active as fallback

Phase 4: Cleanup (after all consumers migrated)
- [ ] Remove old code path
- [ ] Remove feature flag
- [ ] Update documentation
- [ ] Close migration tickets
```

### Rollback Plan
```
If something goes wrong:

Trigger: [What condition triggers a rollback]
- Error rate exceeds X%
- Latency exceeds Xms p99
- Data inconsistency detected
- Customer reports exceed X

Rollback steps:
1. [Revert deployment to previous version]
2. [Run down migration if database changed]
3. [Restore config/env vars if changed]
4. [Notify affected teams]
5. [Investigate root cause]

Estimated rollback time: [X minutes]
Data loss risk during rollback: [None / Possible — describe]
```

## 7. Risk Matrix

### Impact vs Likelihood
```
                    LIKELIHOOD
                    Low     Medium    High
           ┌────────┬─────────┬─────────┐
    High   │ 🟡     │ 🟠      │ 🔴      │
           │ Monitor│ Plan    │ Mitigate │
I   ───────┼────────┼─────────┼─────────┤
M   Medium │ 🟢     │ 🟡      │ 🟠      │
P          │ Accept │ Monitor │ Plan     │
A   ───────┼────────┼─────────┼─────────┤
C   Low    │ 🟢     │ 🟢      │ 🟡      │
T          │ Accept │ Accept  │ Monitor  │
           └────────┴─────────┴─────────┘
```

### Risk Register

| # | Risk | Likelihood | Impact | Score | Mitigation | Owner |
|---|------|-----------|--------|-------|------------|-------|
| 1 | [Risk description] | High | High | 🔴 | [Mitigation approach] | [Name] |
| 2 | [Risk description] | Medium | High | 🟠 | [Mitigation approach] | [Name] |
| 3 | [Risk description] | Low | Medium | 🟢 | [Mitigation approach] | [Name] |

## 8. Checklist by Change Type

### API Change Checklist
- [ ] All consumers identified and notified
- [ ] Backward compatibility period defined
- [ ] API versioning strategy determined
- [ ] OpenAPI/Swagger spec updated
- [ ] Client SDKs updated (if any)
- [ ] Rate limits reviewed
- [ ] Error responses documented
- [ ] Integration tests updated
- [ ] API documentation updated
- [ ] Changelog entry added
- [ ] Mobile app release scheduled (if applicable)

### Database Change Checklist
- [ ] Migration is reversible (up and down)
- [ ] Migration tested on production-size data
- [ ] Migration time estimated (minutes? hours?)
- [ ] Downtime window scheduled (if needed)
- [ ] Backfill strategy defined (if adding non-nullable columns)
- [ ] Indexes planned for new columns used in WHERE/JOIN
- [ ] ORM models updated
- [ ] Affected queries identified and updated
- [ ] Affected reports/dashboards identified
- [ ] Backup verified before migration

### Service Migration Checklist
- [ ] All integration points mapped
- [ ] Feature parity verified (new service does everything old one does)
- [ ] Configuration migrated (env vars, secrets)
- [ ] Monitoring and alerting configured
- [ ] Runbooks updated
- [ ] DNS/routing changes planned
- [ ] Rollback procedure documented
- [ ] Data migration plan (if applicable)
- [ ] Load testing performed on new service
- [ ] Team trained on new service

### Infrastructure Change Checklist
- [ ] All affected services identified
- [ ] Configuration changes documented
- [ ] Secrets/credentials rotated if needed
- [ ] CI/CD pipelines updated
- [ ] Monitoring dashboards updated
- [ ] Alert thresholds reviewed
- [ ] Disaster recovery plan updated
- [ ] Capacity planning reviewed
- [ ] Security group/firewall rules reviewed
- [ ] DNS propagation time accounted for

### Shared Library Change Checklist
- [ ] All consumers identified (grep for imports)
- [ ] Semantic version bump determined (major/minor/patch)
- [ ] Breaking changes documented in CHANGELOG
- [ ] Migration guide written for breaking changes
- [ ] Consumers tested with new version
- [ ] Published to package registry
- [ ] Consumers updated and deployed

## Output Document Template

Save to `project-decisions/YYYY-MM-DD-impact-[topic].md`:
```markdown
# Impact Analysis: [Title of the Proposed Change]

**Date:** YYYY-MM-DD
**Requested by:** [Name]
**Analyzed by:** [Name]
**Change type:** [Feature / Migration / Refactor / Infrastructure / API Change]
**Risk level:** [🟢 Low / 🟡 Medium / 🟠 High / 🔴 Critical]

---

## Proposed Change

[Clear description of what's changing and why]

## Blast Radius Summary

| Area | Impact Level | Details |
|------|-------------|---------|
| **Source code** | X files | [List key files] |
| **API endpoints** | X endpoints | [List affected endpoints] |
| **Database** | X tables/columns | [List affected tables] |
| **Services** | X services | [List affected services] |
| **Infrastructure** | [Yes/No] | [What changes] |
| **Tests** | X test files | [Tests that need updating] |
| **Documentation** | X docs | [Docs that need updating] |
| **Teams affected** | X teams | [List teams] |
| **External consumers** | X consumers | [List consumers] |

## Dependency Map

[ASCII diagram showing affected components and their relationships]

## Breaking Changes

| Change | Type | Who's Affected | Migration Path |
|--------|------|---------------|----------------|
| [Change 1] | API / DB / Library | [Consumer] | [How to migrate] |

## Impact by Area

### Code Impact
[Details of affected files, functions, modules]

### API Impact
[Details of affected endpoints, request/response changes]

### Database Impact
[Details of schema changes, migrations, data backfill]

### Infrastructure Impact
[Details of config, deployment, monitoring changes]

### Team Impact
[Who needs to know, what they need to do]

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| [Risk 1] | H/M/L | H/M/L | [Approach] |

## Rollout Strategy

[Phased rollout plan if breaking changes exist]

## Rollback Plan

[How to revert if things go wrong]

## Checklist

[Relevant checklist based on change type]

## Recommendation

**Proceed: [Yes / Yes with conditions / Needs more analysis / No]**

[Reasoning and conditions]

## Next Steps

1. [ ] [Action item 1]
2. [ ] [Action item 2]
3. [ ] [Action item 3]

---

## Decision Log

| Date | Event | By |
|------|-------|----|
| YYYY-MM-DD | Impact analysis requested | [Name] |
| YYYY-MM-DD | Analysis completed | [Name] |
| YYYY-MM-DD | Decision: [proceed/defer/reject] | [Name] |
```

After saving, update the project-decisions index:
```bash
# Update README.md index
echo "# Project Decisions\n" > project-decisions/README.md
echo "| Date | Decision | Type | Status |" >> project-decisions/README.md
echo "|------|----------|------|--------|" >> project-decisions/README.md

for f in project-decisions/2*.md; do
  date=$(basename "$f" | cut -d'-' -f1-3)
  title=$(head -1 "$f" | sed 's/^# //' | sed 's/^Impact Analysis: //' | sed 's/^Tech Decision: //')
  type="Impact Analysis"
  echo "$f" | grep -q "impact" || type="Tech Decision"
  status=$(grep "^**Status:**\|^**Proceed:**" "$f" | head -1 | sed 's/.*: //' | sed 's/\*//g')
  echo "| $date | [$title](./$(basename $f)) | $type | $status |" >> project-decisions/README.md
done
```

## Adaptation Rules

- **Always save to file** — every analysis gets persisted in `project-decisions/`
- **Scale to the change** — small config change gets 1-page analysis, major migration gets full treatment
- **Be exhaustive on blast radius** — it's better to flag a false positive than miss a real dependency
- **Scan the actual codebase** — don't guess, grep
- **Include transitive dependencies** — the thing that depends on the thing you're changing
- **Quantify** — "15 files affected" not "some files affected"
- **Identify the scariest risk** — lead with what could go most wrong
- **Always include rollback** — every change needs a way back
- **Consider timing** — same change might be fine in a quiet week but risky during peak traffic
- **Think about data** — code can be reverted instantly, data changes cannot

## Summary

End every impact analysis with:
1. **Blast radius** — one-line summary (e.g., "Affects 3 services, 15 files, 2 API endpoints, 1 DB table")
2. **Risk level** — 🟢 Low / 🟡 Medium / 🟠 High / 🔴 Critical
3. **Breaking changes** — count and summary
4. **Teams to notify** — who needs a heads up
5. **Biggest risk** — single most important risk
6. **Recommendation** — proceed / proceed with conditions / needs spike / don't proceed
7. **File saved** — confirm the document location
