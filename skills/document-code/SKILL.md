---
name: document-code
description: Generates documentation for code including function/class docstrings, module READMEs, API documentation, inline comments, and architecture overviews. Use when the user says "document this", "add docs", "add comments", "generate README", "what does this code do?", "write API docs", or "this needs documentation".
allowed-tools: Read, Grep, Glob, Bash
---

# Document Code Skill

When documenting code, follow this structured process. Good documentation explains WHY, not just WHAT. The code already shows what it does — documentation should explain the intent, context, trade-offs, and gotchas.

## 1. Understand Before Documenting

Before writing any documentation:

- **Read the code thoroughly** — understand the full context
- **Identify the audience** — is this for new team members, API consumers, future maintainers, or yourself?
- **Check existing docs** — don't duplicate or contradict what already exists
- **Understand the architecture** — how does this code fit into the bigger picture?

## 2. Documentation Types

Determine what kind of documentation is needed:

| Type | When to Use |
|------|-------------|
| **Inline comments** | Complex logic, workarounds, non-obvious decisions |
| **Function/method docstrings** | Every public function, method, and class |
| **Module/file headers** | Every file that isn't self-explanatory from its name |
| **README** | Every module, package, or service |
| **API documentation** | Every endpoint consumers interact with |
| **Architecture docs** | System design, data flow, component relationships |
| **Changelog** | Tracking changes between versions |
| **Migration guides** | Breaking changes that require action |

## 3. Inline Comments

### When to Comment
- **Complex algorithms** — explain the approach, not each line
- **Workarounds and hacks** — explain WHY and link to the issue/ticket
- **Non-obvious business rules** — explain the domain logic
- **Performance choices** — explain why a less readable approach was chosen
- **Regex patterns** — always explain what they match
- **Magic numbers** — explain where they come from

### When NOT to Comment
- Don't restate what the code already says
- Don't comment every line
- Don't leave commented-out code (use version control)
- Don't write TODOs without ticket numbers
```
// 🔴 BAD — restates the code
// Set age to 18
const age = 18;

// ✅ GOOD — explains WHY
// Minimum age required by COPPA compliance (Children's Online Privacy Protection Act)
const MINIMUM_AGE = 18;

// 🔴 BAD — vague TODO
// TODO: fix this later

// ✅ GOOD — actionable TODO
// TODO(JIRA-1234): Replace with batch query once the ORM supports it. Current N+1
// causes ~200ms latency on pages with 50+ items.

// ✅ GOOD — explains a workaround
// HACK: Safari doesn't support the CSS gap property in flex containers on iOS < 14.5.
// Using margin-based spacing instead. Remove when we drop iOS 14 support.
// See: https://bugs.webkit.org/show_bug.cgi?id=XXXXX

// ✅ GOOD — explains regex
// Matches email addresses: local-part@domain.tld
// Allows: letters, numbers, dots, hyphens, underscores in local part
// Requires: at least one dot in domain, 2-63 char TLD
const EMAIL_REGEX = /^[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,63}$/;
```

## 4. Function & Method Documentation

Every public function should document:
- **What it does** — one-line summary
- **Parameters** — name, type, description, constraints, defaults
- **Return value** — type, description, possible values
- **Errors/exceptions** — what can go wrong and when
- **Examples** — at least one usage example for non-trivial functions
- **Side effects** — database writes, API calls, file changes, event emissions

## 5. Stack-Specific Docstring Formats

### TypeScript / JavaScript (TSDoc / JSDoc)
```typescript
/**
 * Calculates the shipping cost for an order based on weight, destination, and shipping method.
 *
 * Uses the carrier's rate table for standard/express and falls back to flat-rate
 * pricing for international destinations not in the rate table.
 *
 * @param order - The order containing items with weight and dimensions
 * @param destination - Shipping address with country and postal code
 * @param method - Shipping method: 'standard' (5-7 days), 'express' (1-2 days), or 'overnight'
 * @returns The calculated shipping cost in cents (USD). Returns 0 for orders over $100 (free shipping promo)
 * @throws {InvalidAddressError} If the destination address cannot be validated
 * @throws {CarrierUnavailableError} If no carrier supports the destination country
 *
 * @example
 * const cost = await calculateShipping(order, { country: 'US', postal: '90210' }, 'standard');
 * // Returns: 599 (i.e., $5.99)
 *
 * @see {@link ShippingRateTable} for carrier rate configuration
 * @since 2.1.0
 */
async function calculateShipping(
  order: Order,
  destination: Address,
  method: ShippingMethod
): Promise<number> { ... }
```

### TypeScript Interfaces and Types
```typescript
/**
 * Represents a user account in the system.
 *
 * Created during registration and updated via the profile settings page.
 * Soft-deleted when the user requests account deletion (retained for 30 days per compliance policy).
 */
interface User {
  /** Unique identifier (UUIDv4), generated at creation */
  id: string;

  /** User's email address, used for login. Must be unique and verified. */
  email: string;

  /** Display name shown in the UI. 2-50 characters, no special characters. */
  name: string;

  /**
   * User's role determining access level.
   * - 'viewer': Read-only access to shared resources
   * - 'editor': Can create and modify own resources
   * - 'admin': Full access including user management
   */
  role: 'viewer' | 'editor' | 'admin';

  /** ISO 8601 timestamp of account creation */
  createdAt: string;

  /** ISO 8601 timestamp of last login, null if never logged in */
  lastLoginAt: string | null;
}
```

### Python (Google Style Docstrings)
```python
def calculate_shipping(order: Order, destination: Address, method: str = "standard") -> int:
    """Calculate shipping cost for an order based on weight and destination.

    Uses the carrier's rate table for domestic shipments and falls back
    to flat-rate pricing for international destinations not in the rate table.

    Args:
        order: The order containing items with weight and dimensions.
        destination: Shipping address with country and postal code.
        method: Shipping method. One of 'standard' (5-7 days),
            'express' (1-2 days), or 'overnight'. Defaults to 'standard'.

    Returns:
        Shipping cost in cents (USD). Returns 0 for orders over $100
        due to the free shipping promotion.

    Raises:
        InvalidAddressError: If the destination address cannot be validated
            against the carrier's address database.
        CarrierUnavailableError: If no carrier supports the destination country.

    Example:
        >>> cost = calculate_shipping(order, Address(country="US", postal="90210"))
        >>> print(cost)
        599  # $5.99

    Note:
        International rates are refreshed daily at midnight UTC from the
        carrier API. Cached rates may be up to 24 hours stale.
    """
```

### Python Classes
```python
class OrderProcessor:
    """Processes orders from cart submission through fulfillment.

    Handles the complete order lifecycle: validation, payment capture,
    inventory reservation, and shipping label generation. Uses the saga
    pattern to ensure all steps complete or roll back atomically.

    Attributes:
        payment_gateway: The payment provider client (Stripe/PayPal).
        inventory_service: Service for checking and reserving stock.
        notification_service: Sends order confirmation emails and SMS.

    Example:
        >>> processor = OrderProcessor(
        ...     payment_gateway=StripeClient(),
        ...     inventory_service=InventoryService(),
        ... )
        >>> result = await processor.process(cart, user)
        >>> print(result.order_id)
        'ord_abc123'

    Note:
        This class is NOT thread-safe. Use one instance per request
        in web applications.
    """
```

### Go (Godoc)
```go
// CalculateShipping returns the shipping cost in cents for the given order.
//
// It uses the carrier's rate table for domestic shipments and falls back
// to flat-rate pricing for international destinations. Returns 0 for
// orders over $100 (free shipping promotion).
//
// Returns ErrInvalidAddress if the destination cannot be validated.
// Returns ErrCarrierUnavailable if no carrier supports the destination.
//
// Example:
//
//	cost, err := CalculateShipping(order, Address{Country: "US", Postal: "90210"}, Standard)
//	if err != nil {
//	    log.Fatal(err)
//	}
//	fmt.Println(cost) // 599
func CalculateShipping(order Order, dest Address, method ShippingMethod) (int, error) { ... }

// OrderProcessor handles the complete order lifecycle from cart to fulfillment.
//
// It coordinates payment capture, inventory reservation, and shipping label
// generation using the saga pattern for atomic completion or rollback.
//
// OrderProcessor is NOT safe for concurrent use. Use one instance per request.
type OrderProcessor struct {
    // PaymentGateway handles payment capture and refunds.
    PaymentGateway PaymentClient

    // InventoryService checks availability and reserves stock.
    InventoryService InventoryClient
}
```

### Java (Javadoc)
```java
/**
 * Calculates the shipping cost for an order based on weight, destination, and method.
 *
 * <p>Uses the carrier's rate table for domestic shipments and falls back
 * to flat-rate pricing for international destinations not in the rate table.
 *
 * <p><b>Example:</b>
 * <pre>{@code
 * int cost = calculateShipping(order, new Address("US", "90210"), ShippingMethod.STANDARD);
 * // Returns: 599 ($5.99)
 * }</pre>
 *
 * @param order the order containing items with weight and dimensions
 * @param destination shipping address with country and postal code
 * @param method shipping method: STANDARD (5-7 days), EXPRESS (1-2 days), or OVERNIGHT
 * @return shipping cost in cents (USD); returns 0 for orders over $100
 * @throws InvalidAddressException if the destination address cannot be validated
 * @throws CarrierUnavailableException if no carrier supports the destination
 * @since 2.1.0
 * @see ShippingRateTable
 */
public int calculateShipping(Order order, Address destination, ShippingMethod method)
    throws InvalidAddressException, CarrierUnavailableException { ... }
```

### Ruby (YARD)
```ruby
# Calculates the shipping cost for an order based on weight and destination.
#
# Uses the carrier's rate table for domestic shipments and falls back
# to flat-rate pricing for international destinations.
#
# @param order [Order] the order containing items with weight and dimensions
# @param destination [Address] shipping address with country and postal code
# @param method [Symbol] shipping method: :standard, :express, or :overnight
# @return [Integer] shipping cost in cents (USD), 0 for orders over $100
# @raise [InvalidAddressError] if the destination cannot be validated
# @raise [CarrierUnavailableError] if no carrier supports the destination
#
# @example Calculate standard shipping
#   cost = calculate_shipping(order, Address.new(country: "US", postal: "90210"))
#   # => 599
#
# @see ShippingRateTable
# @since 2.1.0
def calculate_shipping(order, destination, method: :standard)
```

### PHP (PHPDoc)
```php
/**
 * Calculates the shipping cost for an order.
 *
 * Uses the carrier's rate table for domestic shipments and falls back
 * to flat-rate pricing for international destinations.
 *
 * @param Order   $order       The order containing items with weight and dimensions.
 * @param Address $destination Shipping address with country and postal code.
 * @param string  $method      Shipping method: 'standard', 'express', or 'overnight'.
 *
 * @return int Shipping cost in cents (USD). Returns 0 for orders over $100.
 *
 * @throws InvalidAddressException    If the destination cannot be validated.
 * @throws CarrierUnavailableException If no carrier supports the destination.
 *
 * @example
 * $cost = $this->calculateShipping($order, new Address('US', '90210'), 'standard');
 * // Returns: 599 ($5.99)
 *
 * @since 2.1.0
 */
public function calculateShipping(Order $order, Address $destination, string $method = 'standard'): int
```

### Rust (Rustdoc)
```rust
/// Calculates the shipping cost for an order based on weight and destination.
///
/// Uses the carrier's rate table for domestic shipments and falls back
/// to flat-rate pricing for international destinations not in the rate table.
///
/// # Arguments
///
/// * `order` - The order containing items with weight and dimensions
/// * `destination` - Shipping address with country and postal code
/// * `method` - Shipping method: `Standard`, `Express`, or `Overnight`
///
/// # Returns
///
/// Shipping cost in cents (USD). Returns `0` for orders over $100.
///
/// # Errors
///
/// Returns `ShippingError::InvalidAddress` if the destination cannot be validated.
/// Returns `ShippingError::CarrierUnavailable` if no carrier supports the destination.
///
/// # Examples
///
/// ```
/// let cost = calculate_shipping(&order, &Address::new("US", "90210"), Method::Standard)?;
/// assert_eq!(cost, 599); // $5.99
/// ```
///
/// # Panics
///
/// Panics if the rate table has not been initialized via [`init_rate_table`].
pub fn calculate_shipping(
    order: &Order,
    destination: &Address,
    method: ShippingMethod,
) -> Result<u32, ShippingError> { ... }
```

### Swift
```swift
/// Calculates the shipping cost for an order based on weight and destination.
///
/// Uses the carrier's rate table for domestic shipments and falls back
/// to flat-rate pricing for international destinations.
///
/// - Parameters:
///   - order: The order containing items with weight and dimensions.
///   - destination: Shipping address with country and postal code.
///   - method: Shipping method. Defaults to `.standard`.
/// - Returns: Shipping cost in cents (USD). Returns 0 for orders over $100.
/// - Throws: `ShippingError.invalidAddress` if the destination cannot be validated.
///
/// ```swift
/// let cost = try calculateShipping(order: order, destination: address, method: .standard)
/// // cost == 599 ($5.99)
/// ```
///
/// - Note: International rates are cached for up to 24 hours.
/// - Since: 2.1.0
func calculateShipping(order: Order, destination: Address, method: ShippingMethod = .standard) throws -> Int
```

### Kotlin (KDoc)
```kotlin
/**
 * Calculates the shipping cost for an order based on weight and destination.
 *
 * Uses the carrier's rate table for domestic shipments and falls back
 * to flat-rate pricing for international destinations.
 *
 * @param order The order containing items with weight and dimensions.
 * @param destination Shipping address with country and postal code.
 * @param method Shipping method. Defaults to [ShippingMethod.STANDARD].
 * @return Shipping cost in cents (USD). Returns 0 for orders over $100.
 * @throws InvalidAddressException if the destination cannot be validated.
 * @sample com.example.shipping.Samples.calculateStandard
 * @since 2.1.0
 * @see ShippingRateTable
 */
suspend fun calculateShipping(
    order: Order,
    destination: Address,
    method: ShippingMethod = ShippingMethod.STANDARD
): Int
```

## 6. Module & File Headers

Every non-trivial file should have a header explaining:
```typescript
/**
 * Order Processing Pipeline
 *
 * Handles the complete order lifecycle from cart submission to fulfillment.
 * This is the core business logic module — changes here affect revenue directly.
 *
 * Flow: Cart → Validation → Payment → Inventory → Shipping → Confirmation
 *
 * Key decisions:
 * - Uses saga pattern for distributed transactions (see ADR-015)
 * - Retry logic uses exponential backoff with jitter (max 3 attempts)
 * - Failed orders go to dead letter queue for manual review
 *
 * Dependencies:
 * - PaymentService (Stripe)
 * - InventoryService (internal gRPC)
 * - NotificationService (SendGrid)
 *
 * @module order-processing
 * @see docs/architecture/order-flow.md
 */
```

## 7. README Generation

When generating a README for a module, service, or project, include:
```markdown
# Module/Service Name

One-line description of what this does.

## Overview

2-3 sentences explaining the purpose, who uses it, and how it fits
into the larger system.

## Getting Started

### Prerequisites
- Required tools and versions
- Environment variables needed
- External services required

### Installation
Step-by-step setup instructions.

### Running Locally
How to start the service/module locally.

## Architecture

Brief description of how the code is organized:
- `src/api/` — HTTP route handlers
- `src/services/` — Business logic
- `src/db/` — Database queries and models
- `src/utils/` — Shared helper functions

## API Reference

### `POST /api/orders`
Creates a new order.

**Request:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| items | array | yes | Cart items with productId and quantity |
| shipping | object | yes | Shipping address |

**Response:** `201 Created`
Returns the created order with id and status.

**Errors:**
| Code | Description |
|------|-------------|
| 400 | Invalid request body |
| 402 | Payment failed |
| 409 | Item out of stock |

## Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| DATABASE_URL | PostgreSQL connection string | — | Yes |
| STRIPE_SECRET_KEY | Stripe API key | — | Yes |
| PORT | Server port | 3000 | No |

## Testing

How to run tests:
- `npm test` — unit tests
- `npm run test:integration` — integration tests
- `npm run test:e2e` — end-to-end tests

## Deployment

How this gets deployed and any special considerations.

## Troubleshooting

Common issues and how to resolve them.
```

## 8. API Documentation

When documenting APIs, include for each endpoint:

- **Method and path** — `GET /api/users/:id`
- **Description** — what it does and when to use it
- **Authentication** — required auth method and permissions
- **Request parameters** — path params, query params, headers, body
- **Request example** — complete curl or code example
- **Response format** — schema with types and descriptions
- **Response examples** — success and error responses
- **Rate limits** — if applicable
- **Versioning** — which API version introduced this
```markdown
### Get User Profile

Retrieves the profile for a specific user. Returns public fields for
other users and full profile for the authenticated user's own profile.

**Endpoint:** `GET /api/v1/users/:id`

**Authentication:** Bearer token required. Scope: `users:read`

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| id | string (UUID) | The user's unique identifier |

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| include | string | — | Comma-separated related resources: `posts`, `followers` |

**Request Example:**
\`\`\`bash
curl -H "Authorization: Bearer <token>" \
  https://api.example.com/v1/users/abc-123?include=posts
\`\`\`

**Success Response:** `200 OK`
\`\`\`json
{
  "id": "abc-123",
  "name": "Alice Smith",
  "email": "alice@example.com",
  "role": "editor",
  "createdAt": "2024-01-15T10:30:00Z"
}
\`\`\`

**Error Responses:**
| Status | Code | Description |
|--------|------|-------------|
| 401 | UNAUTHORIZED | Missing or invalid bearer token |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | USER_NOT_FOUND | No user exists with the given ID |
| 429 | RATE_LIMITED | Too many requests (limit: 100/min) |
```

## 9. Architecture Documentation

When documenting system architecture, include:
```markdown
# System Architecture

## High-Level Overview

ASCII diagram showing major components and their relationships:

┌──────────┐     ┌──────────┐     ┌──────────┐
│  Client   │────▶│ API      │────▶│ Database │
│  (React)  │◀────│ (Express)│◀────│ (Postgres)│
└──────────┘     └────┬─────┘     └──────────┘
                      │
                 ┌────▼─────┐
                 │  Redis    │
                 │  (Cache)  │
                 └──────────┘

## Data Flow

Describe how data moves through the system for key operations:

### Order Creation Flow
1. Client submits order → API validates request
2. API checks inventory → reserves items
3. API charges payment → Stripe webhook confirms
4. API creates order record → sends confirmation email
5. Background job generates shipping label

## Key Design Decisions

| Decision | Choice | Rationale | Trade-offs |
|----------|--------|-----------|------------|
| Database | PostgreSQL | ACID compliance for financial data | Higher ops overhead than NoSQL |
| Cache | Redis | Sub-ms reads for session data | Additional infrastructure to manage |
| Queue | BullMQ | Node.js native, Redis-backed | Single-language ecosystem lock |

## Component Responsibilities

### API Layer (`src/api/`)
Handles HTTP requests, input validation, and response formatting.
Does NOT contain business logic.

### Service Layer (`src/services/`)
Contains all business logic. Orchestrates operations across
multiple repositories and external services.

### Repository Layer (`src/db/`)
Handles all database operations. Only layer that imports the ORM.
Returns plain objects, not ORM models.
```

## 10. Documentation Quality Rules

### DO
- Explain WHY, not just WHAT
- Keep docs close to code (docstrings > wiki pages)
- Update docs when code changes (in the same PR)
- Use examples — they're worth 1000 words
- Document edge cases and gotchas
- Link to related docs, ADRs, and tickets
- Use consistent format across the project

### DON'T
- Don't restate what the code already says
- Don't write novels — be concise
- Don't document private/internal implementation details excessively
- Don't let docs get stale — outdated docs are worse than no docs
- Don't document obvious parameters (`@param name - The name`)
- Don't use jargon without explaining it
- Don't copy-paste docs between similar functions (extract shared docs)

## Output Format

When documenting code, provide:

1. **Documentation added** — list of files and what was documented
2. **Format used** — which docstring/comment style and why
3. **Coverage** — what percentage of public APIs are now documented
4. **README generated** — if applicable, the complete README
5. **Suggestions** — areas where architecture docs or ADRs would help
6. **Existing docs updated** — any outdated docs that were corrected

## Summary

End every documentation task with:
1. **What was documented** — files and functions covered
2. **Documentation gaps remaining** — what still needs docs
3. **Recommendations** — suggested docs to add next (README, API docs, architecture)
4. **Consistency notes** — any formatting inconsistencies found in existing docs
