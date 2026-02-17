---
name: create-pr
description: Prepares and creates a pull request by analyzing changes, generating a descriptive title and body, running pre-PR checks, and submitting via GitHub/GitLab CLI. Use when the user says "create a PR", "open a pull request", "submit PR", "prepare PR", "ready for review", or "push and create PR".
allowed-tools: Read, Grep, Glob, Bash
---

# Create Pull Request Skill

When creating a pull request, follow this structured process. A good PR tells reviewers what changed, why, and how to verify it.

## 1. Pre-PR Discovery

Before creating the PR, gather context:

### Current Branch & Changes
```bash
# Current branch name
git branch --show-current

# Base branch (what we're merging into)
git log --oneline --decorate --first-parent HEAD | head -1
git remote show origin | grep "HEAD branch" | sed 's/.*: //'

# Commits on this branch (not on main/develop)
BASE_BRANCH=$(git remote show origin | grep "HEAD branch" | sed 's/.*: //')
git log --oneline ${BASE_BRANCH}..HEAD

# Files changed
git diff --name-status ${BASE_BRANCH}..HEAD

# Diff stats
git diff --stat ${BASE_BRANCH}..HEAD

# Full diff (for analysis)
git diff ${BASE_BRANCH}..HEAD
```

### Related Context
```bash
# Check for linked issues in commit messages
git log ${BASE_BRANCH}..HEAD --format="%B" | grep -iE "#[0-9]+|JIRA-[0-9]+|[A-Z]+-[0-9]+"

# Check for ticket references in branch name
git branch --show-current | grep -oE "[A-Z]+-[0-9]+|[0-9]+"

# Recent PR template
cat .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null
cat .github/pull_request_template.md 2>/dev/null
cat .gitlab/merge_request_templates/Default.md 2>/dev/null
cat docs/pull_request_template.md 2>/dev/null
```

## 2. Pre-PR Checks

Run automated checks before creating the PR:

### Code Quality
```bash
# Lint
npm run lint 2>/dev/null || yarn lint 2>/dev/null || pnpm lint 2>/dev/null
python -m flake8 2>/dev/null || python -m ruff check . 2>/dev/null
go vet ./... 2>/dev/null
bundle exec rubocop 2>/dev/null

# Format check
npx prettier --check . 2>/dev/null
python -m black --check . 2>/dev/null
gofmt -l . 2>/dev/null
```

### Tests
```bash
# Run tests
npm test 2>/dev/null || yarn test 2>/dev/null
python -m pytest 2>/dev/null
go test ./... 2>/dev/null
bundle exec rspec 2>/dev/null
php artisan test 2>/dev/null
cargo test 2>/dev/null
```

### Build
```bash
# Verify build succeeds
npm run build 2>/dev/null || yarn build 2>/dev/null
go build ./... 2>/dev/null
cargo build 2>/dev/null
```

### Type Check
```bash
# TypeScript
npx tsc --noEmit 2>/dev/null

# Python
python -m mypy . 2>/dev/null || python -m pyright 2>/dev/null
```

### Report Pre-Check Results
```
Pre-PR Checks:
✅ Lint — passed
✅ Tests — 142 passed, 0 failed
✅ Build — succeeded
✅ Type check — no errors
⚠️ Format — 2 files need formatting (auto-fix available)
❌ Test — 3 tests failing (must fix before PR)
```

If any critical checks fail, report them and suggest fixes BEFORE creating the PR.

## 3. Generate PR Title

### Title Format

Follow conventional commit style adapted for PR titles:
```
<type>(<scope>): <short description>
```

**Types:**
| Type | Use When |
|------|----------|
| `feat` | New feature or functionality |
| `fix` | Bug fix |
| `refactor` | Code restructuring without behavior change |
| `perf` | Performance improvement |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `chore` | Build, CI, dependency updates |
| `style` | Formatting, whitespace, semicolons |
| `security` | Security fix or improvement |
| `breaking` | Breaking API or behavior change |

**Scope:** The module, component, or area affected.

**Examples:**
```
feat(auth): add OAuth2 login with Google and GitHub
fix(orders): prevent duplicate charges on retry
refactor(api): extract validation middleware from route handlers
perf(search): add Redis caching for product queries
docs(api): add OpenAPI specs for user endpoints
test(payments): add integration tests for refund flow
chore(deps): upgrade Next.js to 15.1 and React to 19
security(auth): fix JWT token validation bypass
```

### Title Rules
- Max 72 characters
- Imperative mood ("add" not "added" or "adding")
- No period at the end
- Lowercase after the colon
- Specific — not "fix bug" or "update code"

## 4. Generate PR Body

### Standard PR Body Template
```markdown
## What

[1-3 sentences explaining what this PR does. Be specific about the change, not just the area.]

## Why

[Explain the motivation. Link to the issue/ticket. What problem does this solve? What user need does it address?]

## How

[Brief technical explanation of the approach. What design decisions were made and why?]

## Changes

[Group changes by area/purpose]

### [Area 1, e.g., "API Changes"]
- Description of change 1
- Description of change 2

### [Area 2, e.g., "Database"]
- Description of change 3

### [Area 3, e.g., "Frontend"]
- Description of change 4

## Testing

[How was this tested? What should reviewers verify?]

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed
- [ ] Edge cases covered

### How to Test Manually
1. Step-by-step instructions
2. For reviewers to verify
3. The changes work correctly

## Screenshots / Recordings

[If UI changes, include before/after screenshots or recordings]

| Before | After |
|--------|-------|
| [screenshot] | [screenshot] |

## Checklist

- [ ] Code follows project style guidelines
- [ ] Self-reviewed my own code
- [ ] Added/updated documentation
- [ ] Added/updated tests
- [ ] No new warnings or errors
- [ ] Database migrations are reversible
- [ ] Breaking changes are documented

## Related

- Closes #[issue-number]
- Related to #[issue-number]
- Depends on #[pr-number]
- [Link to design doc / ADR / Figma]
```

### Adapt Body Based on Change Type

#### Bug Fix PR
```markdown
## What

Fixes [specific bug description] that caused [specific symptom].

## Root Cause

[Explain what was causing the bug]

## Fix

[Explain the fix and why this approach was chosen]

## How to Reproduce (Before Fix)

1. Step to reproduce
2. Step to reproduce
3. Observe: [wrong behavior]

## After Fix

1. Same steps
2. Same steps
3. Observe: [correct behavior]

## Regression Risk

[What could this fix break? What areas should be tested?]
```

#### Feature PR
```markdown
## What

Adds [feature name] that allows users to [capability].

## User Story

As a [role], I want to [action] so that [benefit].

## Implementation

[Technical approach, architecture decisions, trade-offs]

## API Changes (if applicable)

### New Endpoints
- `POST /api/[resource]` — [description]
- `GET /api/[resource]/:id` — [description]

### Request/Response Examples
[Include curl examples or request/response bodies]

## Database Changes (if applicable)

### New Tables/Columns
- `table_name.column_name` — [type] — [purpose]

### Migration
- Up: [what the migration does]
- Down: [how to rollback]

## Feature Flag

[Is this behind a feature flag? How to enable/disable?]

## Follow-up Work

- [ ] [Future task 1]
- [ ] [Future task 2]
```

#### Refactor PR
```markdown
## What

Refactors [area] to [improvement].

## Motivation

[Why now? What problem was the old code causing?]

## What Changed

[Structural changes, moved files, renamed things, new patterns]

## What Did NOT Change

[Explicitly state that behavior is unchanged. This reassures reviewers.]

## Verification

[How can reviewers confirm behavior is preserved?]
- All existing tests pass without modification
- [Additional verification steps]
```

#### Dependency Update PR
```markdown
## What

Upgrades [package] from [old version] to [new version].

## Why

[Security fix / new features needed / maintenance / EOL]

## Breaking Changes

[List any breaking changes from the upgrade and how they were handled]

## Changelog Highlights

- [Key change 1 from package changelog]
- [Key change 2]

## Migration Steps Taken

- [Step 1]
- [Step 2]
```

#### Database Migration PR
```markdown
## What

Adds/modifies [table/column/index] to support [feature/fix].

## Schema Changes

### Up Migration
[Describe what the migration does]

### Down Migration
[Describe rollback steps — must be reversible]

## Data Impact

- **Existing rows affected**: [count or estimate]
- **Backfill required**: [yes/no, if yes describe the backfill]
- **Downtime required**: [yes/no]
- **Estimated migration time**: [duration for production data volume]

## Deployment Notes

[Any special steps needed during deployment]
```

## 5. Labels & Metadata

### Auto-Detect Labels
Based on the changes, suggest appropriate labels:

| Condition | Label |
|-----------|-------|
| New feature/functionality | `feature`, `enhancement` |
| Bug fix | `bug`, `fix` |
| Only test files changed | `tests` |
| Only docs changed | `documentation` |
| CI/CD files changed | `ci`, `infrastructure` |
| Dependencies updated | `dependencies` |
| Security-related changes | `security` |
| Database migrations added | `database`, `migration` |
| Breaking changes detected | `breaking-change` |
| < 50 lines changed | `size/small` |
| 50-200 lines changed | `size/medium` |
| 200-500 lines changed | `size/large` |
| > 500 lines changed | `size/xl` |
| UI/frontend changes | `frontend`, `ui` |
| API changes | `backend`, `api` |
| Performance improvements | `performance` |

### Reviewers
```bash
# Suggest reviewers based on file ownership
git log --format='%aN' -- [changed-files] | sort | uniq -c | sort -rn | head -5

# Check CODEOWNERS
cat .github/CODEOWNERS CODEOWNERS 2>/dev/null
```

## 6. Platform-Specific PR Creation

### GitHub (using gh CLI)
```bash
# Check if gh is installed and authenticated
gh auth status

# Create PR
gh pr create \
  --title "feat(auth): add OAuth2 login with Google and GitHub" \
  --body "$(cat pr-body.md)" \
  --base main \
  --label "feature,size/medium" \
  --reviewer "teammate1,teammate2" \
  --assignee "@me"

# Create as draft
gh pr create \
  --title "feat(auth): add OAuth2 login with Google and GitHub" \
  --body "$(cat pr-body.md)" \
  --base main \
  --draft

# Create and open in browser
gh pr create \
  --title "feat(auth): add OAuth2 login with Google and GitHub" \
  --body "$(cat pr-body.md)" \
  --base main \
  --web
```

### GitLab (using glab CLI)
```bash
# Check if glab is installed and authenticated
glab auth status

# Create merge request
glab mr create \
  --title "feat(auth): add OAuth2 login with Google and GitHub" \
  --description "$(cat pr-body.md)" \
  --target-branch main \
  --label "feature,size/medium" \
  --reviewer "teammate1,teammate2" \
  --assignee "@me"

# Create as draft
glab mr create \
  --title "feat(auth): add OAuth2 login with Google and GitHub" \
  --description "$(cat pr-body.md)" \
  --target-branch main \
  --draft
```

### Bitbucket (manual or API)
```bash
# No official CLI — provide the URL to create manually
REPO_URL=$(git remote get-url origin | sed 's/\.git$//')
echo "Create PR at: ${REPO_URL}/pull-requests/new"
```

### Azure DevOps (using az CLI)
```bash
# Create PR
az repos pr create \
  --title "feat(auth): add OAuth2 login with Google and GitHub" \
  --description "$(cat pr-body.md)" \
  --target-branch main \
  --reviewers "teammate1" \
  --draft
```

## 7. Pre-Submission Checklist

Before actually creating the PR, verify:
```markdown
## Pre-Submission Verification

### Code
- [ ] All changes are committed (no uncommitted work)
- [ ] Branch is up to date with base branch (rebased or merged)
- [ ] No merge conflicts
- [ ] No unintended files included (check .gitignore)
- [ ] No debug code left (console.log, debugger, print statements)
- [ ] No commented-out code left
- [ ] No hardcoded secrets or credentials
- [ ] No TODO/FIXME without ticket numbers

### Quality
- [ ] Lint passes
- [ ] Tests pass
- [ ] Build succeeds
- [ ] Type check passes (if applicable)
- [ ] No new warnings introduced

### PR Content
- [ ] Title follows convention and is descriptive
- [ ] Body explains what, why, and how
- [ ] Related issues are linked
- [ ] Testing instructions are clear
- [ ] Screenshots included (if UI changes)
- [ ] Breaking changes are documented
- [ ] Reviewers are assigned
- [ ] Labels are applied
```
```bash
# Check for debug statements
grep -rn "console\.log\|debugger\|print(\|pdb\|binding\.pry\|byebug" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.py" --include="*.rb" src/ app/ lib/ 2>/dev/null | grep -v "node_modules" | grep -v "test" | grep -v "spec"

# Check for uncommitted changes
git status --porcelain

# Check for merge conflicts
git diff --check

# Check branch is up to date
git fetch origin
BASE_BRANCH=$(git remote show origin | grep "HEAD branch" | sed 's/.*: //')
BEHIND=$(git rev-list --count HEAD..origin/${BASE_BRANCH})
if [ "$BEHIND" -gt "0" ]; then
  echo "⚠️ Branch is ${BEHIND} commits behind ${BASE_BRANCH}. Rebase recommended."
fi
```

## 8. Post-Creation Actions

After the PR is created:
```bash
# Get PR URL
gh pr view --web

# Add additional context as comments if needed
gh pr comment --body "Additional context: ..."

# Request specific review on certain files
gh pr comment --body "@teammate1 Could you focus on the database migration in `src/db/migrations/`?"

# Link to CI status
gh pr checks
```

## Output Format

Present the PR preparation as:

### 1. Pre-Check Results
```
✅ Lint — passed
✅ Tests — 156 passed, 0 failed
✅ Build — succeeded
✅ Type check — no errors
✅ No debug statements found
✅ No merge conflicts
⚠️ Branch is 2 commits behind main — rebase recommended
```

### 2. PR Preview
Show the complete title and body before creating:
```
Title: feat(auth): add OAuth2 login with Google and GitHub

Labels: feature, size/medium
Reviewers: @teammate1, @teammate2
Base: main ← feature/oauth-login
```

[Full PR body]

### 3. Confirmation
Ask the user to confirm before actually creating the PR:
- "Ready to create this PR? (yes/no/edit)"
- If edit: ask what to change
- If yes: create and provide the PR URL

## Adaptation Rules

- **Detect the platform** — check git remote URL to determine GitHub/GitLab/Bitbucket/Azure
- **Use project template** — if a PR template exists, fill it in rather than using the default
- **Match existing style** — look at recent merged PRs to match the team's conventions
- **Handle monorepos** — if changes span multiple packages, list affected packages
- **Handle stacked PRs** — if the branch is based on another feature branch, note the dependency
- **Draft by default** — if checks fail, create as draft and note what needs fixing
- **Respect branch protection** — check if the base branch has required reviewers or checks

## Summary

End every PR creation with:
1. **PR URL** — direct link to the created PR
2. **Status** — created / created as draft / blocked (with reason)
3. **Pre-check results** — summary of lint/test/build status
4. **Reviewers assigned** — who needs to review
5. **Next steps** — anything the author needs to do (fix tests, rebase, add screenshots)
