---
name: tech-debt-report
description: Quantifies technical debt across the codebase by scanning for code smells, outdated dependencies, complexity hotspots, missing tests, TODOs, dead code, and architectural issues. Produces a prioritized report with effort estimates and business impact. Saves output to project-decisions/ folder. Use when the user says "tech debt report", "how much tech debt", "technical debt", "code health", "codebase health check", "what needs cleanup", "debt audit", "code quality report", "health check", or "what's the state of our codebase?".
allowed-tools: Read, Grep, Glob, Bash
---

# Tech Debt Report Skill

When generating a tech debt report, follow this structured process. The goal is to turn the vague feeling of "our code is messy" into a quantified, prioritized list that leadership and engineering can use to make informed decisions about debt paydown.

**IMPORTANT**: Always save the output as a markdown file in the `project-decisions/` directory at the project root. Create the directory if it doesn't exist.

## 0. Output Setup
```bash
# Create project-decisions directory if it doesn't exist
mkdir -p project-decisions

# File will be saved as:
# project-decisions/YYYY-MM-DD-tech-debt-report.md
```

## 1. Codebase Overview

### Project Stats
```bash
# Total lines of code (excluding deps and build)
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.go" -o -name "*.java" -o -name "*.rb" -o -name "*.php" -o -name "*.rs" \) ! -path '*/node_modules/*' ! -path '*/vendor/*' ! -path '*/dist/*' ! -path '*/build/*' ! -path '*/.git/*' -exec cat {} + 2>/dev/null | wc -l

# File count by language
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.go" -o -name "*.java" -o -name "*.rb" -o -name "*.php" \) ! -path '*/node_modules/*' ! -path '*/vendor/*' ! -path '*/dist/*' ! -path '*/build/*' | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Total source files
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.go" \) ! -path '*/node_modules/*' ! -path '*/dist/*' ! -path '*/build/*' | wc -l

# Total test files
find . -type f \( -name "*.test.*" -o -name "*.spec.*" -o -name "test_*" -o -name "*_test.*" \) ! -path '*/node_modules/*' | wc -l

# Test to source ratio
echo "Source files: $(find . -type f \( -name '*.ts' -o -name '*.js' -o -name '*.py' \) ! -path '*/node_modules/*' ! -path '*/dist/*' ! -name '*.test.*' ! -name '*.spec.*' ! -name 'test_*' | wc -l)"
echo "Test files: $(find . -type f \( -name '*.test.*' -o -name '*.spec.*' -o -name 'test_*' \) ! -path '*/node_modules/*' | wc -l)"

# Age of the codebase
echo "First commit: $(git log --reverse --format='%ai' | head -1)"
echo "Latest commit: $(git log --format='%ai' -1)"
echo "Total commits: $(git rev-list --count HEAD)"
echo "Contributors: $(git log --format='%aN' | sort -u | wc -l)"

# Directory structure
find . -maxdepth 2 -type d ! -path '*/node_modules/*' ! -path '*/.git/*' ! -path '*/dist/*' ! -path '*/build/*' ! -path '*/vendor/*' | sort
```

## 2. Debt Category Scans

### 2a. Code Complexity Debt

#### Large Files
```bash
# Files over 500 lines (likely need splitting)
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.go" -o -name "*.java" -o -name "*.rb" -o -name "*.php" \) ! -path '*/node_modules/*' ! -path '*/dist/*' ! -path '*/build/*' -exec wc -l {} + 2>/dev/null | sort -rn | head -20

# Files over 300 lines (getting unwieldy)
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" \) ! -path '*/node_modules/*' ! -path '*/dist/*' -exec sh -c 'lines=$(wc -l < "$1"); if [ "$lines" -gt 300 ]; then echo "$lines $1"; fi' _ {} \; 2>/dev/null | sort -rn

# Count of oversized files
echo "Files > 500 lines: $(find . -type f \( -name '*.ts' -o -name '*.js' -o -name '*.py' \) ! -path '*/node_modules/*' ! -path '*/dist/*' -exec sh -c 'lines=$(wc -l < "$1"); if [ "$lines" -gt 500 ]; then echo "$1"; fi' _ {} \; 2>/dev/null | wc -l)"
echo "Files > 300 lines: $(find . -type f \( -name '*.ts' -o -name '*.js' -o -name '*.py' \) ! -path '*/node_modules/*' ! -path '*/dist/*' -exec sh -c 'lines=$(wc -l < "$1"); if [ "$lines" -gt 300 ]; then echo "$1"; fi' _ {} \; 2>/dev/null | wc -l)"
```

#### Long Functions
```bash
# Find long functions (rough heuristic for JS/TS)
grep -rn "^\s*\(export \)\?\(async \)\?\(function\|const.*=>.*{\)" --include="*.ts" --include="*.js" src/ 2>/dev/null | head -30

# For Python
grep -rn "^\s*def \|^\s*async def " --include="*.py" src/ app/ 2>/dev/null | head -30

# Deep nesting (indentation > 4 levels)
find . -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -path '*/node_modules/*' ! -path '*/dist/*' -exec sh -c '
  deep=$(awk "{ match(\$0, /^[[:space:]]*/); if (RLENGTH > 16) print FILENAME\":\"NR\": \"RLENGTH/2\" levels - \"\$0 }" "$1" | head -5)
  if [ -n "$deep" ]; then echo "$deep"; fi
' _ {} \; 2>/dev/null | head -20
```

#### Code Duplication
```bash
# Find duplicate lines (rough detection)
find . -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -path '*/node_modules/*' ! -path '*/dist/*' ! -path '*/test*' -exec cat {} + 2>/dev/null | sort | uniq -c | sort -rn | grep -v "^\s*[0-9]* \s*$\|^\s*[0-9]* import\|^\s*[0-9]* //\|^\s*[0-9]* #\|^\s*[0-9]* }\|^\s*[0-9]* {" | head -20

# Find similarly named functions across files (possible duplication)
grep -rn "function \|const.*= \(.*\) =>\|def \|func " --include="*.ts" --include="*.js" --include="*.py" --include="*.go" src/ 2>/dev/null | awk -F'[( ]' '{print $NF}' | sort | uniq -c | sort -rn | awk '$1 > 2' | head -20
```

#### Any Type Abuse (TypeScript)
```bash
# Count `any` usage
echo "Total 'any' types: $(grep -rn ": any\|<any>\|as any\| any;" --include="*.ts" --include="*.tsx" src/ 2>/dev/null | grep -v "node_modules\|test\|spec\|\.d\.ts" | wc -l)"

# List files with most `any` usage
grep -rln ": any\|<any>\|as any" --include="*.ts" --include="*.tsx" src/ 2>/dev/null | grep -v "node_modules\|test\|spec\|\.d\.ts" | xargs -I{} sh -c 'echo "$(grep -c ": any\|<any>\|as any" "$1") $1"' _ {} 2>/dev/null | sort -rn | head -15

# @ts-ignore / @ts-expect-error
echo "Total @ts-ignore: $(grep -rn "@ts-ignore\|@ts-expect-error\|@ts-nocheck" --include="*.ts" --include="*.tsx" src/ 2>/dev/null | grep -v "node_modules" | wc -l)"
```

### 2b. TODO / FIXME / HACK Debt
```bash
# Count all debt markers
echo "=== Debt Markers ==="
echo "TODO: $(grep -rn "TODO" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" --include="*.java" --include="*.rb" --include="*.php" src/ app/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "FIXME: $(grep -rn "FIXME" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" src/ app/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "HACK: $(grep -rn "HACK" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" src/ app/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "XXX: $(grep -rn "XXX" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" src/ app/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "WORKAROUND: $(grep -rn "WORKAROUND\|workaround" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "TEMPORARY: $(grep -rn "TEMPORARY\|temporary\|TEMP " --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "DEPRECATED: $(grep -rn "@deprecated\|DEPRECATED" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules" | wc -l)"

# List all with context
grep -rn "TODO\|FIXME\|HACK\|XXX\|WORKAROUND" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" src/ app/ 2>/dev/null | grep -v "node_modules"

# TODOs without ticket numbers (untracked debt)
grep -rn "TODO" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules" | grep -v "#[0-9]\|JIRA-\|[A-Z]\+-[0-9]\+" | head -20

# Age of TODOs (when were they added?)
for todo in $(grep -rln "TODO\|FIXME\|HACK" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null | grep -v "node_modules" | head -10); do
  echo "=== $todo ==="
  git log -1 --format="%ai %s" -- "$todo" 2>/dev/null
done
```

### 2c. Dependency Debt
```bash
# Outdated packages
npm outdated 2>/dev/null | head -30
pip list --outdated 2>/dev/null | head -30

# Security vulnerabilities
npm audit --json 2>/dev/null | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    if 'vulnerabilities' in data:
        vulns = data['vulnerabilities']
        critical = sum(1 for v in vulns.values() if v.get('severity') == 'critical')
        high = sum(1 for v in vulns.values() if v.get('severity') == 'high')
        moderate = sum(1 for v in vulns.values() if v.get('severity') == 'moderate')
        low = sum(1 for v in vulns.values() if v.get('severity') == 'low')
        print(f'Critical: {critical}, High: {high}, Moderate: {moderate}, Low: {low}')
        print(f'Total: {len(vulns)}')
except: pass
" 2>/dev/null

pip audit 2>/dev/null | tail -5

# Check for lockfile presence
ls package-lock.json yarn.lock pnpm-lock.yaml Pipfile.lock poetry.lock Gemfile.lock composer.lock 2>/dev/null

# Check for unpinned versions
cat package.json 2>/dev/null | grep -E '"\^|"~|"\*|"latest"' | head -20

# Check Node.js version
node --version 2>/dev/null
cat .nvmrc .node-version 2>/dev/null

# Check for deprecated packages
npm ls --json 2>/dev/null | grep -i "deprecated" | head -10

# Count total dependencies
echo "Direct dependencies: $(cat package.json 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get('dependencies',{})))" 2>/dev/null)"
echo "Dev dependencies: $(cat package.json 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get('devDependencies',{})))" 2>/dev/null)"
```

### 2d. Test Debt
```bash
# Test coverage (if configured)
npx jest --coverage --coverageReporters=text-summary 2>/dev/null | tail -10
python -m pytest --cov=. --cov-report=term-missing 2>/dev/null | tail -15
go test -cover ./... 2>/dev/null | tail -10

# Source files without corresponding test files
echo "=== Files Without Tests ==="
find src/ app/ -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -name "*.test.*" ! -name "*.spec.*" ! -name "test_*" ! -name "*.d.ts" ! -name "index.*" ! -path '*/node_modules/*' ! -path '*/dist/*' 2>/dev/null | while read f; do
  base=$(basename "$f" | sed 's/\.\(ts\|tsx\|js\|jsx\|py\)$//')
  if ! find . \( -name "${base}.test.*" -o -name "${base}.spec.*" -o -name "test_${base}.*" \) ! -path '*/node_modules/*' 2>/dev/null | grep -q .; then
    echo "NO TEST: $f"
  fi
done | head -30

# Count untested files
TOTAL_SRC=$(find src/ app/ -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -name "*.test.*" ! -name "*.spec.*" ! -name "test_*" ! -name "*.d.ts" ! -name "index.*" ! -path '*/node_modules/*' ! -path '*/dist/*' 2>/dev/null | wc -l)
TESTED=0
for f in $(find src/ app/ -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -name "*.test.*" ! -name "*.spec.*" ! -name "*.d.ts" ! -name "index.*" ! -path '*/node_modules/*' ! -path '*/dist/*' 2>/dev/null); do
  base=$(basename "$f" | sed 's/\.\(ts\|tsx\|js\|jsx\|py\)$//')
  if find . \( -name "${base}.test.*" -o -name "${base}.spec.*" -o -name "test_${base}.*" \) ! -path '*/node_modules/*' 2>/dev/null | grep -q .; then
    TESTED=$((TESTED + 1))
  fi
done
echo "Source files: $TOTAL_SRC"
echo "With tests: $TESTED"
echo "Without tests: $((TOTAL_SRC - TESTED))"
echo "Test coverage (file level): $(( TESTED * 100 / (TOTAL_SRC > 0 ? TOTAL_SRC : 1) ))%"

# Empty or minimal test files
find . \( -name "*.test.*" -o -name "*.spec.*" \) ! -path '*/node_modules/*' -exec sh -c 'lines=$(wc -l < "$1"); if [ "$lines" -lt 10 ]; then echo "MINIMAL ($lines lines): $1"; fi' _ {} \; 2>/dev/null | head -10

# Skipped tests
grep -rn "\.skip\|xit(\|xdescribe(\|@pytest\.mark\.skip\|@unittest\.skip\|pending " --include="*.test.*" --include="*.spec.*" --include="test_*" . 2>/dev/null | grep -v "node_modules" | head -10
echo "Skipped tests: $(grep -rn '\.skip\|xit(\|xdescribe(\|@pytest\.mark\.skip' --include='*.test.*' --include='*.spec.*' --include='test_*' . 2>/dev/null | grep -v 'node_modules' | wc -l)"
```

### 2e. Documentation Debt
```bash
# Check for README
ls README.md README.rst README 2>/dev/null

# Check README last updated
git log -1 --format="%ai" -- README.md 2>/dev/null

# Check for API documentation
find . -name "openapi*" -o -name "swagger*" -o -name "*.api.yaml" -o -name "*.api.json" 2>/dev/null | head -5

# Check for architecture docs
find . -path "*/docs/*" -o -path "*/documentation/*" 2>/dev/null | head -20

# Check for inline documentation (JSDoc, docstrings)
echo "=== Documentation Coverage ==="
TOTAL_FUNCS=$(grep -rn "^\s*\(export \)\?\(async \)\?\(function\|const.*=>.*{\)\|^\s*def \|^\s*func " --include="*.ts" --include="*.js" --include="*.py" --include="*.go" src/ app/ 2>/dev/null | grep -v "node_modules\|test\|spec" | wc -l)
DOCUMENTED=$(grep -rn "/\*\*\|\"\"\"" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules\|test\|spec" | wc -l)
echo "Functions/methods: $TOTAL_FUNCS"
echo "With docstrings: $DOCUMENTED"
echo "Documentation coverage: $(( DOCUMENTED * 100 / (TOTAL_FUNCS > 0 ? TOTAL_FUNCS : 1) ))%"

# Check for stale docs
find . -path "*/docs/*" -name "*.md" -exec sh -c 'echo "$(git log -1 --format="%ai" -- "$1" 2>/dev/null) $1"' _ {} \; 2>/dev/null | sort | head -10

# Check for .env.example
ls .env.example .env.sample .env.template 2>/dev/null

# Check for CONTRIBUTING.md
ls CONTRIBUTING.md CONTRIBUTING 2>/dev/null

# Check for ADRs/decision records
ls project-decisions/ docs/adr/ docs/decisions/ 2>/dev/null | head -10
```

### 2f. Dead Code Debt
```bash
# Unused exports (TypeScript)
npx ts-prune 2>/dev/null | head -30

# Unused Python code
pip install vulture 2>/dev/null && python -m vulture src/ --min-confidence 80 2>/dev/null | head -30

# Commented-out code blocks
grep -rn "^\s*//.*function\|^\s*//.*class\|^\s*//.*const\|^\s*//.*def \|^\s*#.*def \|^\s*//.*return\|^\s*//.*import" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules" | head -20
echo "Commented-out code blocks: $(grep -rn '^\s*//.*function\|^\s*//.*class\|^\s*//.*const\|^\s*//.*def \|^\s*#.*def ' --include='*.ts' --include='*.js' --include='*.py' src/ app/ 2>/dev/null | grep -v 'node_modules' | wc -l)"

# Unused files (not imported anywhere)
for f in $(find src/ app/ -type f \( -name "*.ts" -o -name "*.js" -o -name "*.py" \) ! -name "index.*" ! -name "*.test.*" ! -name "*.spec.*" ! -path '*/node_modules/*' ! -path '*/dist/*' 2>/dev/null | head -50); do
  base=$(basename "$f" | sed 's/\.\(ts\|tsx\|js\|jsx\|py\)$//')
  refs=$(grep -rln "$base" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "$f" | grep -v "node_modules\|test\|spec" | wc -l)
  if [ "$refs" -eq "0" ]; then
    echo "POSSIBLY UNUSED: $f"
  fi
done | head -15

# Legacy/deprecated directories
find . -maxdepth 3 -type d -name "legacy" -o -name "deprecated" -o -name "old" -o -name "backup" -o -name "archive" 2>/dev/null | grep -v "node_modules\|\.git"
```

### 2g. Configuration & Infrastructure Debt
```bash
# Check for hardcoded values
grep -rn "localhost\|127\.0\.0\.1\|hardcode\|HARDCODE" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules\|test\|spec\|\.md" | head -15

# Check for magic numbers
grep -rn "[^a-zA-Z_][0-9]\{3,\}[^a-zA-Z_0-9]\|setTimeout.*[0-9]\{4,\}\|setInterval.*[0-9]\{4,\}" --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules\|test\|spec\|\.md\|port\|PORT" | head -15

# Check for environment-specific config in code
grep -rn "production\|staging\|development" --include="*.ts" --include="*.js" --include="*.py" src/ app/ 2>/dev/null | grep -v "node_modules\|test\|\.md\|config\|env" | head -10

# Check Docker configuration
cat Dockerfile 2>/dev/null | grep -E "^FROM" | head -5
# Check for unpinned base images
cat Dockerfile 2>/dev/null | grep "FROM.*:latest\|FROM.*:$"

# Check CI/CD configuration age
git log -1 --format="%ai" -- .github/workflows/ .gitlab-ci.yml Jenkinsfile 2>/dev/null

# Check for secrets in code (critical debt)
grep -rn "password.*=.*['\"].\+['\"]\|api_key.*=.*['\"].\+['\"]\|secret.*=.*['\"].\+['\"]" --include="*.ts" --include="*.js" --include="*.py" --include="*.json" --include="*.yaml" --include="*.yml" . 2>/dev/null | grep -v "node_modules\|test\|spec\|\.md\|\.env\.\|example\|sample\|template\|placeholder\|changeme\|xxx" | head -10
```

### 2h. Architectural Debt
```bash
# Circular dependency check (rough)
grep -rn "import.*from.*\.\." --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules" | awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -10

# Check for god modules (files imported by many others)
for f in $(find src/ app/ -type f \( -name "*.ts" -o -name "*.js" \) ! -path '*/node_modules/*' ! -path '*/dist/*' ! -name "index.*" 2>/dev/null | head -50); do
  base=$(basename "$f" | sed 's/\.\(ts\|tsx\|js\|jsx\)$//')
  refs=$(grep -rn "import.*$base\|from.*$base\|require.*$base" --include="*.ts" --include="*.js" src/ app/ 2>/dev/null | grep -v "node_modules\|$f" | wc -l)
  if [ "$refs" -gt 10 ]; then
    echo "GOD MODULE ($refs imports): $f"
  fi
done | sort -t'(' -k2 -rn | head -10

# Check for mixed concerns (files in wrong directories)
# API files with database queries
grep -rln "SELECT\|findOne\|findMany\|\.query(" --include="*.ts" --include="*.js" --include="*.py" src/api/ src/routes/ app/api/ 2>/dev/null | head -10

# Check for monolithic files (too many exports)
for f in $(find src/ app/ -type f \( -name "*.ts" -o -name "*.js" \) ! -path '*/node_modules/*' ! -path '*/dist/*' 2>/dev/null | head -50); do
  exports=$(grep -c "^export " "$f" 2>/dev/null)
  if [ "$exports" -gt 10 ]; then
    echo "MANY EXPORTS ($exports): $f"
  fi
done | sort -t'(' -k2 -rn | head -10

# Check for inconsistent patterns
echo "=== Pattern Consistency ==="
echo "Files using callbacks: $(grep -rln "callback\|cb(" --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "Files using promises: $(grep -rln "\.then(\|Promise" --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules" | wc -l)"
echo "Files using async/await: $(grep -rln "async \|await " --include="*.ts" --include="*.js" src/ 2>/dev/null | grep -v "node_modules" | wc -l)"
```

### 2i. Change Hotspot Analysis
```bash
# Files most frequently changed in last 6 months (high churn = fragile code)
git log --name-only --since="6 months ago" --format="" -- src/ app/ 2>/dev/null | sort | uniq -c | sort -rn | head -20

# Files with most recent bug fixes
git log --oneline --since="6 months ago" --grep="fix\|bug\|hotfix\|patch" -- src/ app/ 2>/dev/null | head -20

# Files most changed AND most bugs (highest risk)
echo "=== Highest Risk Files (most changes + most fixes) ==="
git log --name-only --since="6 months ago" --grep="fix\|bug" --format="" -- src/ app/ 2>/dev/null | sort | uniq -c | sort -rn | head -10

# Files with most reverts
git log --oneline --since="6 months ago" --grep="revert\|Revert" -- src/ app/ 2>/dev/null | head -10

# Bus factor — files only one person has changed
echo "=== Bus Factor Risk ==="
for f in $(git log --name-only --since="12 months ago" --format="" -- src/ 2>/dev/null | sort -u | head -50); do
  authors=$(git log --format='%aN' --since="12 months ago" -- "$f" 2>/dev/null | sort -u | wc -l)
  if [ "$authors" -eq 1 ]; then
    author=$(git log --format='%aN' -1 -- "$f" 2>/dev/null)
    echo "SINGLE AUTHOR ($author): $f"
  fi
done | head -15
```

## 3. Debt Scoring & Prioritization

### Debt Score Per Category
```
| Category | Items Found | Severity | Business Impact | Score (1-10) |
|----------|-------------|----------|----------------|-------------|
| Code Complexity | X large files, Y deep nesting | Medium | Slows development | X |
| TODO/FIXME/HACK | X untracked items | Low-Medium | Hidden bugs | X |
| Dependency Debt | X vulnerable, Y outdated | High | Security risk | X |
| Test Debt | X% untested, Y skipped | High | Regression risk | X |
| Documentation | X% undocumented | Medium | Onboarding slower | X |
| Dead Code | X unused files/functions | Low | Confusion, larger bundle | X |
| Configuration | X hardcoded values, Y secrets | High | Security/ops risk | X |
| Architecture | X god modules, Y circular deps | High | Slows everything | X |
| Change Hotspots | X high-churn fragile files | High | Frequent bugs | X |

Overall Debt Score: X.X / 10
```

### Debt Health Rating
```
1-3:  🟢 HEALTHY — Normal amount of debt, well-managed
4-5:  🟡 MODERATE — Debt is accumulating, needs attention
6-7:  🟠 CONCERNING — Debt is slowing the team significantly
8-10: 🔴 CRITICAL — Debt is a major risk to the project's future
```

### Prioritization Matrix
```
                     EFFORT TO FIX
                     Low      Medium    High
              ┌────────┬─────────┬─────────┐
    High      │ 🔴 DO  │ 🟠 PLAN │ 🟡 PLAN │
              │ FIRST  │ NEXT    │ LATER   │
B   ──────────┼────────┼─────────┼─────────┤
U   Medium    │ 🟠 DO  │ 🟡 PLAN │ 🟢 PARK │
S             │ SOON   │ NEXT    │         │
I   ──────────┼────────┼─────────┼─────────┤
N   Low       │ 🟡 DO  │ 🟢 PARK │ 🟢 PARK │
E             │ IF TIME│         │         │
S             └────────┴─────────┴─────────┘
S
```

## 4. Trend Analysis
```bash
# Debt trend — are things getting better or worse?

# TODOs over time (sample last 6 months)
echo "=== TODO Trend ==="
for month in 6 5 4 3 2 1 0; do
  date=$(date -d "$month months ago" +%Y-%m-01 2>/dev/null || date -v-${month}m +%Y-%m-01 2>/dev/null)
  commit=$(git rev-list -1 --before="$date" HEAD 2>/dev/null)
  if [ -n "$commit" ]; then
    count=$(git show $commit:src/ 2>/dev/null | grep -c "TODO\|FIXME\|HACK" 2>/dev/null || echo "?")
    echo "$date: $count markers"
  fi
done

# Dependencies trend
echo "=== Dependency Age ==="
echo "Last package.json update: $(git log -1 --format='%ai' -- package.json 2>/dev/null)"
echo "Last lockfile update: $(git log -1 --format='%ai' -- package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null)"

# Code growth trend
echo "=== Codebase Growth ==="
for month in 6 5 4 3 2 1 0; do
  date=$(date -d "$month months ago" +%Y-%m-01 2>/dev/null || date -v-${month}m +%Y-%m-01 2>/dev/null)
  commit=$(git rev-list -1 --before="$date" HEAD 2>/dev/null)
  if [ -n "$commit" ]; then
    files=$(git ls-tree -r --name-only $commit -- src/ 2>/dev/null | wc -l)
    echo "$date: $files source files"
  fi
done
```

## 5. Remediation Roadmap

### Quick Wins (< 1 day each)
```
| # | Item | Category | Effort | Impact |
|---|------|----------|--------|--------|
| 1 | [Remove X commented-out code blocks] | Dead Code | 1h | Clean codebase |
| 2 | [Add .env.example with all required vars] | Documentation | 30m | Faster onboarding |
| 3 | [Pin Docker base image versions] | Configuration | 30m | Reproducible builds |
| 4 | [Update X critical dependency CVEs] | Dependencies | 2h | Security |
| 5 | [Remove X unused imports/files] | Dead Code | 1h | Smaller bundle |
```

### Sprint-Sized (1-5 days each)
```
| # | Item | Category | Effort | Impact |
|---|------|----------|--------|--------|
| 6 | [Split UserService.ts (800 lines) into 4 focused services] | Complexity | 3d | Maintainability |
| 7 | [Add tests for payment service (0% → 80%)] | Testing | 3d | Regression safety |
| 8 | [Replace 45 `any` types with proper interfaces] | Complexity | 2d | Type safety |
| 9 | [Upgrade Node.js from 16 to 20] | Dependencies | 2d | Security, performance |
| 10 | [Document API endpoints in OpenAPI spec] | Documentation | 3d | Team productivity |
```

### Large Efforts (1-4 weeks each)
```
| # | Item | Category | Effort | Impact |
|---|------|----------|--------|--------|
| 11 | [Extract shared library from duplicated code] | Architecture | 2w | DRY, consistency |
| 12 | [Migrate from callbacks to async/await] | Complexity | 2w | Readability |
| 13 | [Implement proper repository pattern for DB access] | Architecture | 3w | Testability |
| 14 | [Add comprehensive error handling across all API endpoints] | Complexity | 2w | Reliability |
```

### Suggested Sprint Allocation
```
Recommended: 20% of each sprint allocated to debt reduction

Sprint capacity: 40 person-days
Feature work: 32 person-days (80%)
Debt paydown: 8 person-days (20%)

Sprint N:     Quick wins #1-5 (5 days) + Sprint item #6 start (3 days)
Sprint N+1:   Sprint items #7-8 (5 days) + Quick wins (3 days)
Sprint N+2:   Sprint items #9-10 (5 days) + Quick wins (3 days)
Sprint N+3+:  Large effort #11 start (8 days over multiple sprints)

Projected debt score reduction:
Current: X.X / 10
After 1 sprint: X.X / 10
After 3 sprints: X.X / 10
After 6 sprints: X.X / 10
```

## Output Document Template

Save to `project-decisions/YYYY-MM-DD-tech-debt-report.md`:
```markdown
# Tech Debt Report

**Date:** YYYY-MM-DD
**Analyzed by:** [Name]
**Codebase:** [Project name]
**Overall Debt Score:** X.X / 10 — [🟢 Healthy / 🟡 Moderate / 🟠 Concerning / 🔴 Critical]

---

## Executive Summary

[2-3 sentences: overall health, biggest concerns, recommended action]

---

## Codebase Overview

| Metric | Value |
|--------|-------|
| Lines of code | X |
| Source files | X |
| Test files | X |
| Test:Source ratio | X:1 |
| Languages | [list] |
| Age | X years |
| Contributors | X |

---

## Debt by Category

| Category | Score (1-10) | Items | Top Issue |
|----------|-------------|-------|-----------|
| Code Complexity | X | X items | [Top issue] |
| TODOs/FIXMEs | X | X markers | [Top issue] |
| Dependencies | X | X vulnerable | [Top issue] |
| Test Coverage | X | X% untested | [Top issue] |
| Documentation | X | X% undocumented | [Top issue] |
| Dead Code | X | X items | [Top issue] |
| Configuration | X | X items | [Top issue] |
| Architecture | X | X issues | [Top issue] |
| Change Hotspots | X | X fragile files | [Top issue] |

---

## Detailed Findings

### Code Complexity
[Findings with file references]

### Dependencies
[Vulnerability counts, outdated packages]

### Test Coverage
[Coverage stats, untested critical files]

### Documentation
[Coverage stats, stale docs]

### Architecture
[God modules, circular deps, inconsistent patterns]

### Change Hotspots
[Most fragile files, bus factor risks]

---

## Trend

[Is debt increasing or decreasing? Data from trend analysis]

---

## Remediation Roadmap

### Quick Wins (< 1 day)
[Table of quick wins]

### Sprint-Sized (1-5 days)
[Table of sprint items]

### Large Efforts (1-4 weeks)
[Table of large items]

---

## Recommended Sprint Allocation

[20% debt budget, prioritized items per sprint]

---

## Appendix

### A. Full TODO/FIXME List
[Complete list with file locations]

### B. Dependency Audit
[Full npm audit / pip audit output]

### C. Untested Files
[Complete list of files without tests]

### D. Bus Factor Report
[Files with single author]

---

## Decision Log

| Date | Event | By |
|------|-------|----|
| YYYY-MM-DD | Tech debt report generated | [Name] |
| YYYY-MM-DD | Reviewed by team | [Name] |
| YYYY-MM-DD | Remediation roadmap approved | [Name] |
| YYYY-MM-DD | Sprint N debt items completed | [Name] |
```

After saving, update the project-decisions index.

## Adaptation Rules

- **Always save to file** — every report gets persisted in `project-decisions/`
- **Scan, don't guess** — run actual commands to quantify debt
- **Prioritize by business impact** — "slows every PR by 30 minutes" is more compelling than "messy code"
- **Include effort estimates** — leadership needs to know the cost of fixing vs cost of ignoring
- **Show trends** — is it getting better or worse? This drives urgency
- **Be specific** — "userService.ts is 847 lines with 23 functions" not "some files are too long"
- **Include quick wins** — show that progress can start immediately
- **Recommend sprint allocation** — 20% is a proven sustainable rate for debt paydown
- **Connect to incidents** — if debt caused an outage, link to the incident report
- **Compare to benchmarks** — "industry average test coverage for our stage is 70%, we're at 35%"
- **Run quarterly** — schedule regular debt assessments to track progress

## Summary

End every tech debt report with:
1. **Overall score** — X.X / 10 with health rating
2. **Top 3 concerns** — biggest risks to the project
3. **Quick win count** — how many items can be fixed in < 1 day
4. **Recommended sprint budget** — % of capacity for debt paydown
5. **Projected improvement** — score after 3 sprints of focused work
6. **File saved** — confirm the document location
