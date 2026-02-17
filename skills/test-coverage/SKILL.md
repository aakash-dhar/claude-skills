---
name: test-coverage
description: Analyzes test coverage gaps, identifies untested code paths, prioritizes which tests to write based on risk, and helps improve overall test quality. Use when the user says "what's not tested?", "check test coverage", "what tests are missing?", "improve coverage", or "are my tests good enough?".
allowed-tools: Read, Grep, Glob, Bash
---

# Test Coverage Analysis Skill

When analyzing test coverage, follow this structured process. The goal is not 100% coverage — it's ensuring the riskiest code is tested well.

## 1. Discover the Testing Setup

Before analyzing, understand the project's testing landscape:
```bash
# Detect testing framework
# Node.js
cat package.json | grep -E "jest|vitest|mocha|ava|tap|playwright|cypress|testing-library"

# Python
cat requirements.txt pyproject.toml setup.cfg 2>/dev/null | grep -E "pytest|unittest|nose|coverage|tox"

# Ruby
cat Gemfile 2>/dev/null | grep -E "rspec|minitest|capybara|factory_bot"

# Go
grep -r "_test.go" --include="*.go" -l .

# Java
cat pom.xml build.gradle 2>/dev/null | grep -E "junit|mockito|testng|jacoco"

# PHP
cat composer.json 2>/dev/null | grep -E "phpunit|pest|mockery"
```

Identify:
- **Testing framework** in use (Jest, Vitest, Pytest, RSpec, JUnit, etc.)
- **Coverage tool** configured (Istanbul/nyc, coverage.py, SimpleCov, JaCoCo, etc.)
- **Test directory structure** (co-located vs separate test folder)
- **Naming conventions** (*.test.ts, *.spec.ts, test_*.py, *_test.go)
- **Test types present** (unit, integration, e2e, snapshot)
- **CI integration** (are tests running in CI? is coverage enforced?)

## 2. Run Existing Coverage
```bash
# Node.js (Jest)
npx jest --coverage --coverageReporters=text

# Node.js (Vitest)
npx vitest run --coverage

# Python (Pytest)
python -m pytest --cov=. --cov-report=term-missing

# Go
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out

# Ruby (RSpec)
COVERAGE=true bundle exec rspec

# Java (Maven + JaCoCo)
mvn test jacoco:report

# PHP (PHPUnit)
php artisan test --coverage
```

Record:
- **Overall line coverage percentage**
- **Overall branch coverage percentage**
- **Files with 0% coverage** (completely untested)
- **Files with < 50% coverage** (poorly tested)
- **Uncovered lines** (specific line numbers)

## 3. Identify What's NOT Tested

### 3a. Find Files Without Tests
```bash
# Node.js — find source files without matching test files
find src -name "*.ts" -o -name "*.js" | while read f; do
  base=$(basename "$f" | sed 's/\.\(ts\|js\)$//')
  if ! find . -name "${base}.test.*" -o -name "${base}.spec.*" | grep -q .; then
    echo "NO TEST: $f"
  fi
done

# Python — find modules without test files
find src -name "*.py" ! -name "__init__.py" | while read f; do
  base=$(basename "$f" .py)
  if ! find . -name "test_${base}.py" -o -name "${base}_test.py" | grep -q .; then
    echo "NO TEST: $f"
  fi
done

# Go — find packages without test files
find . -name "*.go" ! -name "*_test.go" -exec dirname {} \; | sort -u | while read d; do
  if ! ls "$d"/*_test.go 2>/dev/null | grep -q .; then
    echo "NO TEST: $d"
  fi
done
```

### 3b. Find Untested Code Paths

Look for these commonly missed patterns:

- **Error handlers and catch blocks** — the most commonly untested code
- **Edge cases** — null, undefined, empty arrays, zero, negative numbers, boundary values
- **Else branches** — the unhappy path in if/else
- **Switch default cases** — fallback handling
- **Early returns and guard clauses** — validation at the top of functions
- **Timeout and retry logic** — what happens when things fail
- **Race conditions** — concurrent operations
- **Cleanup code** — finally blocks, destructors, shutdown handlers
- **Configuration branches** — code that runs differently per environment
- **Deprecated or feature-flagged code** — code behind flags that's still reachable

### 3c. Find Dead or Unreachable Code
```bash
# Node.js — find unused exports
npx ts-prune

# Python — find unused code
pip install vulture && vulture src/

# General — find functions not referenced anywhere
grep -rn "function\|def\|func " src/ | while read line; do
  fname=$(echo "$line" | grep -oP '(?:function|def|func)\s+\K\w+')
  count=$(grep -rn "$fname" src/ | wc -l)
  if [ "$count" -le 1 ]; then
    echo "POSSIBLY UNUSED: $line"
  fi
done
```

## 4. Risk-Based Prioritization

Not all untested code is equally important. Prioritize by risk:

### 🔴 Critical — Test These First
- **Authentication and authorization** — login, signup, password reset, permission checks
- **Payment and billing** — charge, refund, subscription logic
- **Data mutation** — create, update, delete operations
- **API endpoints** — especially public-facing ones
- **Input validation** — sanitization and parsing of user input
- **Security-sensitive code** — encryption, token generation, access control
- **Core business logic** — the main value of your application

### 🟠 High — Test These Next
- **Error handling** — catch blocks, error boundaries, fallback behavior
- **Database queries** — complex queries, transactions, migrations
- **Third-party integrations** — API calls, webhooks, callbacks
- **State management** — reducers, stores, state transitions
- **File operations** — uploads, downloads, processing
- **Background jobs** — queues, cron jobs, workers

### 🟡 Medium — Test When Possible
- **UI components** — interactive components, forms, modals
- **Utility functions** — helpers, formatters, transformers
- **Configuration** — environment-specific logic
- **Middleware** — request/response processing pipeline
- **Caching logic** — cache invalidation, TTL, fallbacks

### 🟢 Low — Test If Time Permits
- **Static components** — presentational components with no logic
- **Type definitions** — interfaces, types, enums
- **Constants and config objects** — static values
- **Logging** — log formatting and output
- **Dev-only code** — seeders, fixtures, debug utilities

## 5. Test Quality Analysis

Coverage percentage alone doesn't mean tests are good. Analyze quality:

### Assertion Quality
```
// 🔴 BAD — test runs but asserts nothing meaningful
test('creates user', async () => {
  const result = await createUser({ name: 'Alice' });
  expect(result).toBeTruthy(); // too vague
});

// ✅ GOOD — specific, meaningful assertions
test('creates user with correct fields', async () => {
  const result = await createUser({ name: 'Alice' });
  expect(result.id).toBeDefined();
  expect(result.name).toBe('Alice');
  expect(result.createdAt).toBeInstanceOf(Date);
});
```

### Test Independence
```
// 🔴 BAD — tests depend on each other's state
let userId;
test('creates user', async () => {
  const user = await createUser({ name: 'Alice' });
  userId = user.id;
});
test('fetches user', async () => {
  const user = await getUser(userId); // depends on previous test
});

// ✅ GOOD — each test sets up its own state
test('fetches user', async () => {
  const created = await createUser({ name: 'Alice' });
  const fetched = await getUser(created.id);
  expect(fetched.name).toBe('Alice');
});
```

### Common Test Smells
- **No assertions** — test runs code but checks nothing
- **Testing implementation, not behavior** — brittle tests that break on refactors
- **Over-mocking** — mocking so much that the test proves nothing
- **Flaky tests** — tests that pass/fail randomly (timing, order-dependent, network)
- **Duplicate tests** — same scenario tested multiple times in different places
- **Giant test files** — 1000+ line test files that are hard to maintain
- **Missing cleanup** — tests that leave behind state (DB records, files, env vars)
- **Snapshot overuse** — snapshots accepted without review, hiding regressions
- **Copy-paste tests** — duplicated setup that should be extracted into helpers
- **Happy path only** — only testing success, never failure

## 6. Stack-Specific Checks

### Node.js / Jest / Vitest
- Check for missing `afterEach` cleanup (open handles, DB connections)
- Verify async tests use `await` or return promises (silent failures otherwise)
- Check for missing `jest.mock()` cleanup between tests
- Look for `setTimeout` in tests without `jest.useFakeTimers()`
- Verify snapshot tests are intentional and reviewed
- Check for missing error boundary tests in React components
- Verify `act()` wrapping on React state updates in tests
- Look for missing `waitFor` / `findBy` on async UI updates

### Python / Pytest
- Check for missing `conftest.py` fixtures for common setup
- Verify database tests use transactions and rollback (`@pytest.mark.django_db`)
- Look for missing `parametrize` on tests that should cover multiple inputs
- Check for missing `mock.patch` cleanup (use context managers or decorators)
- Verify async tests use `@pytest.mark.asyncio`
- Check for missing exception tests (`with pytest.raises(...)`)
- Look for hardcoded file paths in tests (use `tmp_path` fixture)
- Verify test isolation — no tests reading/writing shared state

### React / Next.js
- Check for missing `render` tests on all user-facing components
- Verify form components test validation, submission, and error states
- Look for missing accessibility tests (`@testing-library/jest-dom` matchers)
- Check for untested loading and error states
- Verify hooks are tested with `renderHook` from testing-library
- Check for missing tests on context providers and consumers
- Look for untested route guards and redirects
- Verify Server Components have integration tests
- Check for missing tests on API routes / Server Actions

### Vue / Nuxt
- Check for missing `mount` / `shallowMount` tests on components
- Verify Pinia stores are tested with `createTestingPinia`
- Look for missing `emitted()` checks on event emissions
- Check for untested computed properties and watchers
- Verify composables are tested independently
- Check for missing tests on Nuxt middleware and plugins

### Go
- Check for missing table-driven tests on functions with multiple inputs
- Verify error return values are tested (not just happy path)
- Look for missing `t.Parallel()` on independent tests
- Check for missing `t.Helper()` on test utility functions
- Verify interfaces are tested with mock implementations
- Check for missing benchmark tests on performance-critical code (`func BenchmarkX`)
- Look for missing `t.Cleanup()` for resource teardown
- Verify HTTP handlers are tested with `httptest.NewServer`

### Java / Spring Boot
- Check for missing `@SpringBootTest` integration tests
- Verify `@MockBean` is used appropriately (not over-mocked)
- Look for missing `@Transactional` on database tests (auto rollback)
- Check for missing controller tests with `MockMvc`
- Verify exception handlers are tested
- Check for missing `@ParameterizedTest` on multi-input tests
- Look for untested `@Scheduled` tasks and async methods
- Verify repository custom queries have integration tests

### Ruby / RSpec
- Check for missing `describe` blocks for each public method
- Verify `let` and `before` blocks handle proper setup/teardown
- Look for missing `context` blocks for different scenarios
- Check for missing `shared_examples` for common behavior
- Verify factory definitions cover all required fields (`FactoryBot`)
- Check for missing request specs on API endpoints
- Look for untested ActiveRecord callbacks and validations
- Verify background jobs (Sidekiq/Resque) have specs

### PHP / Laravel
- Check for missing Feature tests on routes and controllers
- Verify database tests use `RefreshDatabase` or `DatabaseTransactions`
- Look for missing tests on form requests (validation rules)
- Check for missing `assertDatabaseHas` / `assertDatabaseMissing` assertions
- Verify mail, notification, and event fakes are used
- Check for missing tests on Eloquent scopes and accessors
- Look for untested middleware
- Verify queue jobs and listeners have tests

### Mobile (React Native / Flutter)
- Check for missing widget/component tests
- Verify navigation flows are tested
- Look for missing tests on platform-specific code (iOS vs Android)
- Check for untested offline/error states
- Verify async storage operations are tested
- Check for missing tests on deep link handling
- Look for untested permission request flows
- Verify API response parsing is tested with realistic mock data

### API / Integration Tests
- Check for missing tests on all HTTP methods (GET, POST, PUT, DELETE, PATCH)
- Verify authentication is tested (valid token, expired token, no token, wrong role)
- Look for missing tests on pagination, filtering, and sorting
- Check for missing tests on rate limiting behavior
- Verify webhook handlers are tested with realistic payloads
- Check for missing tests on file upload/download endpoints
- Look for untested CORS behavior
- Verify error responses match API documentation/contract

### Database
- Check for missing migration tests (up and rollback)
- Verify complex queries are tested with realistic data volumes
- Look for missing tests on database constraints (unique, foreign key, check)
- Check for untested transaction behavior (commit, rollback, deadlock)
- Verify connection pooling and timeout handling is tested
- Check for missing seed data validation tests

## 7. Coverage Improvement Plan

After analysis, provide a prioritized plan:

### Immediate (This Sprint)
- List specific files and functions to test first based on risk
- Provide test scaffolding for the highest-priority untested code
- Suggest specific test cases with descriptions

### Short-Term (Next 2 Sprints)
- Increase coverage on medium-risk areas
- Add integration tests for critical user flows
- Fix test quality issues (weak assertions, flaky tests)

### Long-Term (This Quarter)
- Set up coverage thresholds in CI (fail build if coverage drops)
- Add e2e tests for critical user journeys
- Implement mutation testing to verify test effectiveness
- Set up coverage trend tracking

## Output Format

### Coverage Report

**Overall Coverage:**
| Metric | Current | Target | Gap |
|--------|---------|--------|-----|
| Line Coverage | X% | 80% | X% |
| Branch Coverage | X% | 70% | X% |
| Function Coverage | X% | 85% | X% |

**Untested Files (by risk):**

🔴 **Critical — No Tests:**
- `src/auth/login.ts` — handles user authentication
- `src/payments/charge.ts` — processes payments

🟠 **High — Partial Coverage:**
- `src/api/users.ts` — 40% covered, missing error paths
- `src/db/queries.ts` — 55% covered, missing edge cases

**Test Quality Issues:**
- `tests/user.test.ts:45` — assertion too vague (`toBeTruthy`)
- `tests/order.test.ts` — tests are order-dependent
- `tests/api.test.ts:120` — over-mocked, not testing real behavior

### For Each Untested Area

**File: `src/auth/login.ts`**
- **Risk**: 🔴 Critical
- **Why it matters**: Handles user authentication — bugs here = security breach
- **Missing tests**:
  1. Valid login with correct credentials
  2. Login with wrong password (should return 401)
  3. Login with non-existent email (should return 401, same message as wrong password)
  4. Login with expired account
  5. Rate limiting after 5 failed attempts
  6. SQL injection attempt in email field
  7. Session creation and token generation
- **Test scaffold**:
```
describe('login', () => {
  it('returns a token for valid credentials', async () => { ... });
  it('returns 401 for incorrect password', async () => { ... });
  it('returns 401 for non-existent email', async () => { ... });
  it('locks account after 5 failed attempts', async () => { ... });
  it('rejects SQL injection in email', async () => { ... });
});
```

## Summary

End every analysis with:
1. **Current state** — Overall coverage and quality assessment
2. **Biggest gaps** — The riskiest untested code
3. **Top 5 tests to write first** — Highest impact, with brief descriptions
4. **Test quality issues** — Problems with existing tests that need fixing
5. **CI recommendations** — Coverage thresholds, pre-commit hooks, reporting
6. **What's done well** — Good testing practices already in place to maintain
