---
name: refactor
description: Detects code smells, suggests design pattern improvements, and restructures code while preserving existing behavior. Use when the user says "refactor this", "clean up this code", "this code is messy", "improve code quality", "reduce complexity", "simplify this", "make this more maintainable", "code smells", or "this function is too long".
allowed-tools: Read, Grep, Glob, Bash
---

# Refactor Skill

When refactoring code, follow this structured process. The golden rule: change structure without changing behavior. Every refactoring should be verifiable by existing tests — if there are no tests, write them first.

## 1. Pre-Refactor Analysis

Before changing anything, understand the current state:

### Read and Map the Code
```bash
# Understand the file and its dependencies
cat [target-file]

# Find what imports this file (who depends on it)
grep -rn "import.*from.*[filename]" --include="*.ts" --include="*.js" --include="*.py" src/ 2>/dev/null
grep -rn "require.*[filename]" --include="*.js" src/ 2>/dev/null

# Find what this file imports (what it depends on)
grep -E "^import|^from|require\(" [target-file]

# Check for existing tests
find . -name "*[filename]*test*" -o -name "*[filename]*spec*" -o -name "test_*[filename]*" 2>/dev/null
```

### Measure Current Complexity
```bash
# Line count per function (rough estimate)
grep -n "function\|const.*=.*=>\|def \|func \|fn \|pub fn\|class " [target-file]

# File line count
wc -l [target-file]

# Nesting depth (count indentation levels)
awk '{ match($0, /^[[:space:]]*/); print RLENGTH/2, $0 }' [target-file] | sort -rn | head -10
```

### Verify Test Coverage
```bash
# Run existing tests to establish a baseline
npm test -- --coverage --testPathPattern=[filename] 2>/dev/null
python -m pytest --cov=[module] tests/test_[filename].py 2>/dev/null
go test -cover ./[package]/ 2>/dev/null

# If no tests exist, flag this immediately
```

**CRITICAL**: If no tests cover the code being refactored, the FIRST step is to add characterization tests that capture the current behavior. Never refactor untested code.

## 2. Code Smell Detection

Scan for these common smells, grouped by severity:

### 🔴 Critical Smells — Refactor First

#### Long Functions / Methods
```
Symptom: Function > 30 lines or does more than one thing
Detection: Count lines, look for multiple levels of abstraction
```
```bash
# Find long functions (rough heuristic)
awk '/^[[:space:]]*(export )?(async )?(function|const.*=>|def |func |fn |pub fn)/{name=$0; count=0} {count++} /^[[:space:]]*\}/{if(count>30) print count, name}' [target-file]
```

#### God Class / God Module
```
Symptom: File > 500 lines, class with 10+ methods, module that does everything
Detection: Line count, method count, number of imports
```
```bash
# Count methods in classes
grep -c "^\s*\(public\|private\|protected\|async\)\?\s*\w\+\s*(" [target-file]

# Count imports (high import count = too many responsibilities)
grep -c "^import\|^from.*import\|require(" [target-file]
```

#### Deeply Nested Code
```
Symptom: 4+ levels of indentation, nested if/else/for/try
Detection: Indentation analysis

// 🔴 SMELL — deeply nested
function processOrder(order) {
  if (order) {
    if (order.items.length > 0) {
      for (const item of order.items) {
        if (item.quantity > 0) {
          if (item.inStock) {
            // actual logic buried 5 levels deep
          }
        }
      }
    }
  }
}

// ✅ REFACTORED — early returns + extracted functions
function processOrder(order) {
  if (!order) return;
  if (order.items.length === 0) return;

  const validItems = order.items.filter(item => item.quantity > 0 && item.inStock);
  validItems.forEach(processItem);
}
```

#### Duplicated Code
```
Symptom: Same logic copy-pasted in multiple places
Detection: Similar code blocks, only differing in small details
```
```bash
# Find duplicate lines (rough detection)
sort [target-file] | uniq -d | grep -v "^$\|^import\|^//\|^#\|^\s*$" | head -20

# Find similar functions across files
grep -rn "function.*validate\|def.*validate\|func.*Validate" --include="*.ts" --include="*.py" --include="*.go" src/ 2>/dev/null
```

#### Primitive Obsession
```
Symptom: Using raw strings/numbers instead of domain types

// 🔴 SMELL — raw strings everywhere
function createUser(name: string, email: string, role: string, status: string) { ... }
function sendEmail(to: string, subject: string, body: string, priority: string) { ... }

// ✅ REFACTORED — domain types
function createUser(input: CreateUserInput): User { ... }
function sendEmail(message: EmailMessage): void { ... }

interface CreateUserInput {
  name: string;
  email: Email;        // validated email type
  role: UserRole;      // 'admin' | 'editor' | 'viewer'
  status: UserStatus;  // 'active' | 'inactive' | 'suspended'
}
```

### 🟡 Warning Smells — Refactor Soon

#### Feature Envy
```
Symptom: A function that uses more data from another class/module than its own

// 🔴 SMELL — OrderPrinter knows too much about Order internals
class OrderPrinter {
  print(order: Order) {
    const subtotal = order.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
    const tax = subtotal * order.taxRate;
    const shipping = order.weight > 10 ? 15.99 : 5.99;
    const total = subtotal + tax + shipping;
    // ...
  }
}

// ✅ REFACTORED — move calculation to Order
class Order {
  get subtotal() { return this.items.reduce((sum, i) => sum + i.price * i.quantity, 0); }
  get tax() { return this.subtotal * this.taxRate; }
  get shippingCost() { return this.weight > 10 ? 15.99 : 5.99; }
  get total() { return this.subtotal + this.tax + this.shippingCost; }
}

class OrderPrinter {
  print(order: Order) {
    // Just use order.total, order.subtotal, etc.
  }
}
```

#### Long Parameter Lists
```
Symptom: Function with 4+ parameters

// 🔴 SMELL
function createUser(name, email, role, department, manager, startDate, salary, location) { ... }

// ✅ REFACTORED — parameter object
function createUser(input: CreateUserInput) { ... }
```

#### Switch/If-Else Chains
```
Symptom: Long switch or if/else chains that map types to behavior

// 🔴 SMELL
function calculateDiscount(customerType: string, amount: number): number {
  if (customerType === 'gold') return amount * 0.2;
  if (customerType === 'silver') return amount * 0.1;
  if (customerType === 'bronze') return amount * 0.05;
  if (customerType === 'employee') return amount * 0.3;
  return 0;
}

// ✅ REFACTORED — strategy map
const DISCOUNT_RATES: Record<string, number> = {
  gold: 0.2,
  silver: 0.1,
  bronze: 0.05,
  employee: 0.3,
};

function calculateDiscount(customerType: string, amount: number): number {
  return amount * (DISCOUNT_RATES[customerType] ?? 0);
}
```

#### Boolean Blindness
```
Symptom: Functions that take boolean flags to change behavior

// 🔴 SMELL — what does `true` mean here?
processOrder(order, true, false, true);

// ✅ REFACTORED — named options
processOrder(order, {
  expedited: true,
  giftWrap: false,
  sendNotification: true,
});
```

#### Magic Numbers and Strings
```
Symptom: Unexplained literal values in code

// 🔴 SMELL
if (retryCount > 3) { ... }
if (user.role === 'admin') { ... }
const timeout = 30000;

// ✅ REFACTORED
const MAX_RETRIES = 3;
const ROLES = { ADMIN: 'admin', EDITOR: 'editor' } as const;
const REQUEST_TIMEOUT_MS = 30_000;
```

#### Dead Code
```
Symptom: Unreachable code, unused variables, commented-out blocks
```
```bash
# Find unused exports (TypeScript)
npx ts-prune 2>/dev/null

# Find unused variables
npx eslint --rule '{"no-unused-vars": "error"}' [target-file] 2>/dev/null

# Find commented-out code blocks
grep -n "^\s*//.*function\|^\s*//.*const\|^\s*//.*class\|^\s*#.*def " [target-file]
```

### 🟢 Minor Smells — Refactor When Convenient

#### Inconsistent Naming
```
Symptom: Mixed conventions in the same codebase

// 🔴 SMELL — mixed naming
const user_name = getUserName();
const userEmail = get_user_email();
const UserRole = fetchRole();

// ✅ REFACTORED — consistent camelCase
const userName = getUserName();
const userEmail = getUserEmail();
const userRole = fetchRole();
```

#### Comments That Should Be Code
```
Symptom: Comments explaining WHAT, not WHY

// 🔴 SMELL — comment restates the code
// Check if user is admin
if (user.role === 'admin') { ... }

// ✅ REFACTORED — self-documenting code
if (user.isAdmin()) { ... }

// 🔴 SMELL — comment as section header
// --- Validation ---
if (!name) throw new Error('Name required');
if (!email) throw new Error('Email required');
if (!isValidEmail(email)) throw new Error('Invalid email');

// ✅ REFACTORED — extract to function
validateUserInput({ name, email });
```

#### Overly Complex Conditionals
```
// 🔴 SMELL
if ((user.role === 'admin' || user.role === 'superadmin') && user.isActive && !user.isSuspended && user.emailVerified) { ... }

// ✅ REFACTORED
const canAccessAdminPanel = (user: User): boolean =>
  ['admin', 'superadmin'].includes(user.role) &&
  user.isActive &&
  !user.isSuspended &&
  user.emailVerified;

if (canAccessAdminPanel(user)) { ... }
```

## 3. Refactoring Techniques

### Extract Function
```
When: Code block does a distinct sub-task within a larger function
Goal: Each function does one thing

// Before
function processOrder(order: Order) {
  // validate
  if (!order.items.length) throw new Error('Empty order');
  if (order.items.some(i => i.quantity <= 0)) throw new Error('Invalid quantity');
  if (!order.shippingAddress) throw new Error('No address');

  // calculate totals
  const subtotal = order.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  const tax = subtotal * 0.08;
  const shipping = subtotal > 10000 ? 0 : 599;
  const total = subtotal + tax + shipping;

  // save
  return db.orders.create({ ...order, subtotal, tax, shipping, total });
}

// After
function processOrder(order: Order) {
  validateOrder(order);
  const pricing = calculateOrderPricing(order);
  return db.orders.create({ ...order, ...pricing });
}

function validateOrder(order: Order): void {
  if (!order.items.length) throw new Error('Empty order');
  if (order.items.some(i => i.quantity <= 0)) throw new Error('Invalid quantity');
  if (!order.shippingAddress) throw new Error('No address');
}

function calculateOrderPricing(order: Order): OrderPricing {
  const subtotal = order.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  const tax = subtotal * TAX_RATE;
  const shipping = subtotal > FREE_SHIPPING_THRESHOLD ? 0 : STANDARD_SHIPPING;
  return { subtotal, tax, shipping, total: subtotal + tax + shipping };
}
```

### Extract Class / Module
```
When: A class/file has too many responsibilities
Goal: Single Responsibility Principle

// Before — UserService does everything
class UserService {
  createUser() { ... }
  updateUser() { ... }
  deleteUser() { ... }
  login() { ... }
  logout() { ... }
  resetPassword() { ... }
  verifyEmail() { ... }
  sendWelcomeEmail() { ... }
  sendPasswordResetEmail() { ... }
  generateReport() { ... }
  exportToCsv() { ... }
}

// After — split by responsibility
class UserService { createUser(); updateUser(); deleteUser(); }
class AuthService { login(); logout(); resetPassword(); verifyEmail(); }
class UserNotificationService { sendWelcomeEmail(); sendPasswordResetEmail(); }
class UserReportService { generateReport(); exportToCsv(); }
```

### Replace Conditional with Polymorphism
```
// Before — switch on type everywhere
function calculateArea(shape: Shape): number {
  switch (shape.type) {
    case 'circle': return Math.PI * shape.radius ** 2;
    case 'rectangle': return shape.width * shape.height;
    case 'triangle': return (shape.base * shape.height) / 2;
  }
}

function getPerimeter(shape: Shape): number {
  switch (shape.type) {
    case 'circle': return 2 * Math.PI * shape.radius;
    case 'rectangle': return 2 * (shape.width + shape.height);
    case 'triangle': return shape.sideA + shape.sideB + shape.sideC;
  }
}

// After — each shape knows its own calculations
interface Shape {
  area(): number;
  perimeter(): number;
}

class Circle implements Shape {
  constructor(private radius: number) {}
  area() { return Math.PI * this.radius ** 2; }
  perimeter() { return 2 * Math.PI * this.radius; }
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  area() { return this.width * this.height; }
  perimeter() { return 2 * (this.width + this.height); }
}
```

### Introduce Parameter Object
```
// Before
function searchProducts(
  query: string,
  category: string,
  minPrice: number,
  maxPrice: number,
  sortBy: string,
  sortOrder: string,
  page: number,
  pageSize: number,
): Promise<Product[]> { ... }

// After
interface ProductSearchParams {
  query: string;
  category?: string;
  priceRange?: { min: number; max: number };
  sort?: { field: string; order: 'asc' | 'desc' };
  pagination?: { page: number; pageSize: number };
}

function searchProducts(params: ProductSearchParams): Promise<Product[]> { ... }
```

### Replace Nested Conditionals with Guard Clauses
```
// Before — nested pyramid
function processPayment(payment: Payment): Result {
  if (payment) {
    if (payment.amount > 0) {
      if (payment.currency) {
        if (isValidCard(payment.card)) {
          // actual logic here
          return chargeCard(payment);
        } else {
          throw new Error('Invalid card');
        }
      } else {
        throw new Error('Currency required');
      }
    } else {
      throw new Error('Amount must be positive');
    }
  } else {
    throw new Error('Payment required');
  }
}

// After — flat with guard clauses
function processPayment(payment: Payment): Result {
  if (!payment) throw new Error('Payment required');
  if (payment.amount <= 0) throw new Error('Amount must be positive');
  if (!payment.currency) throw new Error('Currency required');
  if (!isValidCard(payment.card)) throw new Error('Invalid card');

  return chargeCard(payment);
}
```

### Decompose Conditional
```
// Before — complex condition
if (date.getMonth() >= 5 && date.getMonth() <= 8 && date.getHours() >= 9 && date.getHours() <= 17) {
  rate = summerPeakRate;
} else if (date.getMonth() >= 11 || date.getMonth() <= 1) {
  rate = winterRate;
} else {
  rate = standardRate;
}

// After — named conditions
const isSummer = date.getMonth() >= 5 && date.getMonth() <= 8;
const isPeakHours = date.getHours() >= 9 && date.getHours() <= 17;
const isWinter = date.getMonth() >= 11 || date.getMonth() <= 1;

if (isSummer && isPeakHours) {
  rate = summerPeakRate;
} else if (isWinter) {
  rate = winterRate;
} else {
  rate = standardRate;
}
```

### Replace Temp with Query
```
// Before — temporary variables
function getPrice(order: Order): number {
  const basePrice = order.quantity * order.itemPrice;
  const discount = Math.max(0, order.quantity - 100) * order.itemPrice * 0.05;
  const shipping = Math.min(basePrice * 0.1, 50);
  return basePrice - discount + shipping;
}

// After — computed properties / methods
class Order {
  get basePrice(): number {
    return this.quantity * this.itemPrice;
  }

  get discount(): number {
    return Math.max(0, this.quantity - 100) * this.itemPrice * 0.05;
  }

  get shipping(): number {
    return Math.min(this.basePrice * 0.1, 50);
  }

  get totalPrice(): number {
    return this.basePrice - this.discount + this.shipping;
  }
}
```

### Move Function / Move Field
```
When: A function uses more data from another module than its own

// Before — shipping logic lives in OrderService but only uses ShippingConfig
class OrderService {
  calculateShipping(order: Order): number {
    const config = this.shippingConfig;
    if (order.weight > config.heavyThreshold) return config.heavyRate;
    if (order.destination.country !== 'US') return config.internationalRate;
    return config.standardRate;
  }
}

// After — move to where the data lives
class ShippingService {
  calculate(weight: number, destination: Address): number {
    if (weight > this.config.heavyThreshold) return this.config.heavyRate;
    if (destination.country !== 'US') return this.config.internationalRate;
    return this.config.standardRate;
  }
}
```

## 4. Stack-Specific Refactoring Patterns

### TypeScript / JavaScript
- Replace `any` types with proper interfaces
- Convert callbacks to async/await
- Replace class components with functional components + hooks (React)
- Extract custom hooks for shared stateful logic (React)
- Replace `var` with `const`/`let`
- Convert CommonJS `require` to ES module `import`
- Replace string enums with `as const` objects or TypeScript enums
- Extract validation schemas (Zod/Joi) from inline checks
- Replace nested `.then()` chains with async/await
- Use optional chaining (`?.`) and nullish coalescing (`??`) instead of manual checks

### Python
- Replace manual dict access with dataclasses or Pydantic models
- Convert synchronous code to async where I/O bound
- Replace nested `try/except` with context managers
- Extract repeated query logic into repository methods
- Replace `*args/**kwargs` abuse with explicit parameters
- Use `functools.lru_cache` for expensive pure computations
- Replace string formatting with f-strings
- Convert class-based views to function-based (or vice versa) based on complexity
- Extract configuration into settings/config module
- Replace manual iteration with list comprehensions where cleaner

### Go
- Replace `interface{}` / `any` with proper typed interfaces
- Extract small interfaces at the point of use (dependency inversion)
- Replace error string matching with sentinel errors or typed errors
- Group related functions into methods on a struct
- Replace global state with dependency injection
- Extract middleware into composable functions
- Replace manual JSON marshaling with struct tags
- Use table-driven patterns for repetitive logic
- Replace panics with error returns
- Extract reusable code from handlers into service layer

### React / Next.js
- Extract reusable UI into components
- Lift state up or push it down based on usage
- Replace prop drilling with context or state management
- Extract data fetching into custom hooks or server components
- Replace `useEffect` for data fetching with React Query / SWR / server components
- Memoize expensive computations with `useMemo`
- Split large components into smaller, focused ones
- Replace inline styles with CSS modules or Tailwind classes
- Move business logic out of components into utility functions
- Replace client components with server components where possible (Next.js)

### Vue / Nuxt
- Extract reusable logic into composables
- Replace Options API with Composition API
- Extract shared state into Pinia stores
- Replace event bus with proper state management
- Move business logic out of components into composables/services
- Replace mixins with composables

### Java / Spring Boot
- Extract interfaces for dependency injection
- Replace service locator pattern with constructor injection
- Apply repository pattern for data access
- Extract DTOs to separate request/response from domain models
- Replace checked exceptions with runtime exceptions where appropriate
- Extract validation into dedicated validator classes
- Replace inheritance with composition
- Apply builder pattern for complex object construction

### Ruby / Rails
- Extract service objects from fat controllers
- Replace callbacks with explicit service calls
- Extract query objects for complex database queries
- Replace concerns/mixins with composition
- Extract decorators/presenters for view logic
- Move business logic from models to service objects
- Replace `method_missing` with explicit methods
- Extract form objects for complex form handling

### PHP / Laravel
- Extract actions/services from controllers
- Replace query scopes with repository pattern for complex queries
- Extract form requests for validation
- Replace facades with dependency injection (testability)
- Extract event/listener pairs for side effects
- Replace trait abuse with composition
- Extract DTOs for data transfer between layers
- Move business logic from Eloquent models to service classes

### Database / SQL
- Extract repeated queries into views or stored procedures
- Replace N+1 queries with joins or eager loading
- Extract complex WHERE clauses into named scopes
- Replace raw SQL with query builder or ORM methods
- Normalize denormalized tables (or denormalize for performance)
- Extract migration logic into reversible migrations
- Replace string concatenation in queries with parameterized queries

## 5. Refactoring Process

### Step-by-Step Workflow
```
1. IDENTIFY — Find the smell or problem area
2. TEST — Ensure tests exist (write characterization tests if not)
3. PLAN — Choose the refactoring technique
4. SMALL STEPS — Make one small change at a time
5. VERIFY — Run tests after each change
6. COMMIT — Commit each successful step separately
7. REVIEW — Verify the overall improvement
```

### Safe Refactoring Rules
- **Never refactor and add features at the same time** — separate commits
- **Never refactor without tests** — if no tests exist, add them first
- **Small steps** — each change should be reversible and individually testable
- **Run tests frequently** — after every change, not just at the end
- **Preserve the API** — if other code depends on this, keep the external interface stable
- **Document why** — commit messages should explain the refactoring rationale

## Output Format

### For Each Smell Found

**[SEVERITY] Smell Name — File:Line**
- **What**: Description of the smell
- **Why it matters**: Impact on maintainability/readability/bugs
- **Technique**: Which refactoring pattern to apply
- **Before**: Current code (relevant snippet)
- **After**: Refactored code
- **Steps**: Ordered refactoring steps
- **Risk**: What could break, how to verify

### Refactoring Plan

Present changes in priority order:
```
Priority 1 — Quick Wins (< 30 min each, high impact)
1. Extract validateOrder() from processOrder() — reduces function from 80 to 30 lines
2. Replace magic numbers with named constants — 12 occurrences across 4 files
3. Remove dead code in utils.ts — 3 unused functions, 45 lines

Priority 2 — Medium Effort (1-2 hours each)
4. Split UserService into UserService + AuthService — 600 line file → 2x ~200 lines
5. Replace nested callbacks with async/await in paymentHandler.ts
6. Extract duplicate validation logic into shared validator

Priority 3 — Larger Refactors (half-day to full-day)
7. Introduce repository pattern for database access — currently mixed into services
8. Replace inheritance hierarchy with composition in notification system
```

### Complexity Metrics
```
Before Refactoring:
- File: src/services/orderService.ts
- Lines: 487
- Functions: 12
- Max nesting depth: 6
- Cyclomatic complexity: 34
- Dependencies: 15 imports

After Refactoring:
- Files: 4 (orderService, orderValidator, orderPricing, orderNotifier)
- Lines: 125 avg (500 total)
- Functions: 5 avg per file
- Max nesting depth: 2
- Cyclomatic complexity: 8 avg
- Dependencies: 5 avg imports per file
```

## Adaptation Rules

- **Match the project's style** — don't introduce patterns the team doesn't use
- **Respect the architecture** — refactor within the existing architectural decisions
- **Consider the team** — prefer simpler patterns that everyone can understand
- **Don't over-engineer** — refactor to the level of complexity actually needed
- **Prioritize by impact** — fix the worst smells first, not the easiest ones
- **Keep PRs focused** — one refactoring theme per PR for easier review
- **Check for tests first** — if untested, add characterization tests before refactoring

## Summary

End every refactoring with:
1. **Smells found** — count and severity breakdown
2. **Changes made** — files modified, functions extracted, lines saved
3. **Complexity reduction** — before/after metrics
4. **Tests status** — all passing, new tests added
5. **Breaking changes** — any API changes that affect other code
6. **Remaining tech debt** — what's left to address in future
7. **Recommended follow-ups** — next refactoring priorities
