---
name: write-tests
description: Generates comprehensive tests for a given file, function, class, or module. Covers unit tests, integration tests, edge cases, error paths, and mocking. Use when the user says "write tests", "add tests", "test this", "create tests for", "generate test cases", "cover this with tests", or "improve test coverage".
allowed-tools: Read, Grep, Glob, Bash
---

# Write Tests Skill

When writing tests, follow this structured process. Good tests verify behavior, catch regressions, and serve as living documentation.

## 1. Discovery — Understand What to Test

Before writing any tests, analyze the code:

### Detect Testing Setup
```bash
# Detect framework and config
cat package.json 2>/dev/null | grep -E "jest|vitest|mocha|ava|playwright|cypress|testing-library"
cat jest.config.* vitest.config.* 2>/dev/null | head -30
cat pyproject.toml 2>/dev/null | grep -A 20 "\[tool.pytest"
cat pytest.ini conftest.py 2>/dev/null | head -20
cat .rspec spec/spec_helper.rb 2>/dev/null | head -20
cat phpunit.xml 2>/dev/null | head -20

# Find existing test examples to match patterns
find . -name "*.test.*" -o -name "*.spec.*" -o -name "test_*" -o -name "*_test.*" | head -10

# Read an existing test file to understand conventions
# (pick the most recent or most relevant one)
```

### Analyze the Code Under Test
```bash
# Read the file to understand its exports and dependencies
cat [target-file]

# Find what it imports (these might need mocking)
grep -E "^import|^from|require\(" [target-file]

# Find what exports it provides (these are what we test)
grep -E "^export|module\.exports" [target-file]

# Find related types/interfaces
grep -E "interface |type |class " [target-file]
```

### Identify Test Cases
For each function/method/class, identify:

1. **Happy path** — normal successful operation
2. **Edge cases** — boundary values, empty inputs, limits
3. **Error cases** — invalid inputs, failures, exceptions
4. **Integration points** — database, API, file system calls
5. **State transitions** — before/after state changes
6. **Concurrency** — race conditions, parallel execution
7. **Security** — malicious inputs, injection attempts

## 2. Test Structure Principles

### Arrange-Act-Assert (AAA)
```
// Arrange — set up the test data and conditions
// Act — execute the code under test
// Assert — verify the results
```

### Test Naming
Use descriptive names that explain the scenario:
```
// Pattern: [unit] [scenario] [expected result]

// ✅ GOOD
"createUser returns user with generated UUID when valid email provided"
"createUser throws ValidationError when email is empty string"
"createUser hashes password with bcrypt before saving"

// 🔴 BAD
"createUser works"
"test 1"
"should work correctly"
```

### Test Organization
```
describe('[Module/Class/Function]', () => {
  describe('[method or scenario group]', () => {
    it('[specific behavior]', () => { ... });
  });
});
```

### What Makes a Good Test
- **Independent** — no test depends on another test's state
- **Deterministic** — same result every time (no random, no timing)
- **Fast** — unit tests < 100ms each
- **Readable** — a test is documentation; anyone should understand it
- **Focused** — one assertion concept per test (can have multiple expect statements)
- **Realistic** — test data resembles real data, not `"test"` and `123`

## 3. Stack-Specific Test Generation

### TypeScript / JavaScript — Jest
```typescript
import { describe, it, expect, jest, beforeEach, afterEach } from '@jest/globals';
import { OrderService } from '../services/orderService';
import { OrderRepository } from '../db/repositories/orderRepository';
import { PaymentService } from '../lib/paymentService';
import { NotificationService } from '../lib/notificationService';
import type { CreateOrderInput, Order } from '../models/order';

// Mock dependencies
jest.mock('../db/repositories/orderRepository');
jest.mock('../lib/paymentService');
jest.mock('../lib/notificationService');

describe('OrderService', () => {
  let orderService: OrderService;
  let mockOrderRepo: jest.Mocked<OrderRepository>;
  let mockPaymentService: jest.Mocked<PaymentService>;
  let mockNotificationService: jest.Mocked<NotificationService>;

  // --- Setup & Teardown ---

  beforeEach(() => {
    mockOrderRepo = new OrderRepository() as jest.Mocked<OrderRepository>;
    mockPaymentService = new PaymentService() as jest.Mocked<PaymentService>;
    mockNotificationService = new NotificationService() as jest.Mocked<NotificationService>;
    orderService = new OrderService(mockOrderRepo, mockPaymentService, mockNotificationService);
  });

  afterEach(() => {
    jest.restoreAllMocks();
  });

  // --- Test Data Factories ---

  const createValidOrderInput = (overrides?: Partial<CreateOrderInput>): CreateOrderInput => ({
    userId: 'user-abc-123',
    items: [
      { productId: 'prod-001', quantity: 2, priceInCents: 2999 },
      { productId: 'prod-002', quantity: 1, priceInCents: 4999 },
    ],
    shippingAddress: {
      street: '123 Main St',
      city: 'Springfield',
      state: 'IL',
      zip: '62701',
      country: 'US',
    },
    ...overrides,
  });

  const createMockOrder = (overrides?: Partial<Order>): Order => ({
    id: 'order-xyz-789',
    userId: 'user-abc-123',
    status: 'pending',
    totalInCents: 10997,
    items: [],
    createdAt: new Date('2026-01-15T10:00:00Z'),
    updatedAt: new Date('2026-01-15T10:00:00Z'),
    ...overrides,
  });

  // --- Tests ---

  describe('createOrder', () => {
    // Happy Path
    describe('when given valid input', () => {
      it('creates an order with correct total calculated from items', async () => {
        const input = createValidOrderInput();
        mockOrderRepo.create.mockResolvedValue(createMockOrder({ totalInCents: 10997 }));
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });

        const result = await orderService.createOrder(input);

        expect(mockOrderRepo.create).toHaveBeenCalledWith(
          expect.objectContaining({
            userId: 'user-abc-123',
            totalInCents: 10997, // (2999 * 2) + (4999 * 1)
          })
        );
        expect(result.totalInCents).toBe(10997);
      });

      it('authorizes payment before saving the order', async () => {
        const input = createValidOrderInput();
        mockOrderRepo.create.mockResolvedValue(createMockOrder());
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });

        await orderService.createOrder(input);

        // Verify payment authorization was called before order creation
        const paymentCallOrder = mockPaymentService.authorize.mock.invocationCallOrder[0];
        const repoCallOrder = mockOrderRepo.create.mock.invocationCallOrder[0];
        expect(paymentCallOrder).toBeLessThan(repoCallOrder);
      });

      it('sends order confirmation notification', async () => {
        const input = createValidOrderInput();
        mockOrderRepo.create.mockResolvedValue(createMockOrder());
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });

        await orderService.createOrder(input);

        expect(mockNotificationService.sendOrderConfirmation).toHaveBeenCalledWith(
          expect.objectContaining({
            userId: 'user-abc-123',
            orderId: 'order-xyz-789',
          })
        );
      });
    });

    // Edge Cases
    describe('edge cases', () => {
      it('handles single item order', async () => {
        const input = createValidOrderInput({
          items: [{ productId: 'prod-001', quantity: 1, priceInCents: 999 }],
        });
        mockOrderRepo.create.mockResolvedValue(createMockOrder({ totalInCents: 999 }));
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });

        const result = await orderService.createOrder(input);

        expect(result.totalInCents).toBe(999);
      });

      it('handles maximum quantity per item', async () => {
        const input = createValidOrderInput({
          items: [{ productId: 'prod-001', quantity: 99, priceInCents: 100 }],
        });
        mockOrderRepo.create.mockResolvedValue(createMockOrder({ totalInCents: 9900 }));
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });

        const result = await orderService.createOrder(input);

        expect(result.totalInCents).toBe(9900);
      });

      it('applies free shipping for orders over 10000 cents ($100)', async () => {
        const input = createValidOrderInput({
          items: [{ productId: 'prod-001', quantity: 1, priceInCents: 15000 }],
        });
        mockOrderRepo.create.mockResolvedValue(createMockOrder({ totalInCents: 15000 }));
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });

        const result = await orderService.createOrder(input);

        expect(mockOrderRepo.create).toHaveBeenCalledWith(
          expect.objectContaining({ shippingCostInCents: 0 })
        );
      });
    });

    // Error Cases
    describe('error handling', () => {
      it('throws ValidationError when items array is empty', async () => {
        const input = createValidOrderInput({ items: [] });

        await expect(orderService.createOrder(input)).rejects.toThrow('Order must contain at least one item');
      });

      it('throws ValidationError when quantity is zero or negative', async () => {
        const input = createValidOrderInput({
          items: [{ productId: 'prod-001', quantity: 0, priceInCents: 999 }],
        });

        await expect(orderService.createOrder(input)).rejects.toThrow('Quantity must be at least 1');
      });

      it('throws ValidationError when price is negative', async () => {
        const input = createValidOrderInput({
          items: [{ productId: 'prod-001', quantity: 1, priceInCents: -100 }],
        });

        await expect(orderService.createOrder(input)).rejects.toThrow('Price must be a positive number');
      });

      it('does not save order when payment authorization fails', async () => {
        const input = createValidOrderInput();
        mockPaymentService.authorize.mockRejectedValue(new Error('Card declined'));

        await expect(orderService.createOrder(input)).rejects.toThrow('Card declined');
        expect(mockOrderRepo.create).not.toHaveBeenCalled();
      });

      it('rolls back payment when order save fails', async () => {
        const input = createValidOrderInput();
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });
        mockOrderRepo.create.mockRejectedValue(new Error('Database connection lost'));

        await expect(orderService.createOrder(input)).rejects.toThrow('Database connection lost');
        expect(mockPaymentService.refund).toHaveBeenCalledWith('txn-001');
      });

      it('does not send notification when order creation fails', async () => {
        const input = createValidOrderInput();
        mockPaymentService.authorize.mockRejectedValue(new Error('Payment failed'));

        await expect(orderService.createOrder(input)).rejects.toThrow();
        expect(mockNotificationService.sendOrderConfirmation).not.toHaveBeenCalled();
      });

      it('still completes order when notification fails (non-critical)', async () => {
        const input = createValidOrderInput();
        mockOrderRepo.create.mockResolvedValue(createMockOrder());
        mockPaymentService.authorize.mockResolvedValue({ transactionId: 'txn-001' });
        mockNotificationService.sendOrderConfirmation.mockRejectedValue(new Error('Email service down'));

        // Should not throw — notification failure is non-critical
        const result = await orderService.createOrder(input);
        expect(result.id).toBe('order-xyz-789');
      });
    });
  });

  describe('cancelOrder', () => {
    it('updates order status to cancelled', async () => {
      mockOrderRepo.findById.mockResolvedValue(createMockOrder({ status: 'pending' }));
      mockOrderRepo.update.mockResolvedValue(createMockOrder({ status: 'cancelled' }));

      const result = await orderService.cancelOrder('order-xyz-789', 'user-abc-123');

      expect(result.status).toBe('cancelled');
    });

    it('refunds payment when cancelling a paid order', async () => {
      mockOrderRepo.findById.mockResolvedValue(
        createMockOrder({ status: 'paid', paymentTransactionId: 'txn-001' })
      );
      mockOrderRepo.update.mockResolvedValue(createMockOrder({ status: 'cancelled' }));

      await orderService.cancelOrder('order-xyz-789', 'user-abc-123');

      expect(mockPaymentService.refund).toHaveBeenCalledWith('txn-001');
    });

    it('throws ForbiddenError when user does not own the order', async () => {
      mockOrderRepo.findById.mockResolvedValue(createMockOrder({ userId: 'different-user' }));

      await expect(
        orderService.cancelOrder('order-xyz-789', 'user-abc-123')
      ).rejects.toThrow('You do not have permission to cancel this order');
    });

    it('throws NotFoundError when order does not exist', async () => {
      mockOrderRepo.findById.mockResolvedValue(null);

      await expect(
        orderService.cancelOrder('nonexistent-id', 'user-abc-123')
      ).rejects.toThrow('Order not found');
    });

    it('throws ConflictError when order is already shipped', async () => {
      mockOrderRepo.findById.mockResolvedValue(createMockOrder({ status: 'shipped' }));

      await expect(
        orderService.cancelOrder('order-xyz-789', 'user-abc-123')
      ).rejects.toThrow('Cannot cancel an order that has already been shipped');
    });

    it('throws ConflictError when order is already cancelled', async () => {
      mockOrderRepo.findById.mockResolvedValue(createMockOrder({ status: 'cancelled' }));

      await expect(
        orderService.cancelOrder('order-xyz-789', 'user-abc-123')
      ).rejects.toThrow('Order is already cancelled');
    });
  });
});
```

### TypeScript / JavaScript — Vitest
```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { calculateShipping } from '../services/shipping';
import type { Order, Address } from '../models/order';

describe('calculateShipping', () => {
  const createOrder = (overrides?: Partial<Order>): Order => ({
    id: 'order-001',
    totalWeightGrams: 500,
    totalInCents: 5000,
    items: [{ productId: 'p1', quantity: 1, priceInCents: 5000 }],
    ...overrides,
  });

  const createAddress = (overrides?: Partial<Address>): Address => ({
    country: 'US',
    state: 'CA',
    zip: '90210',
    ...overrides,
  });

  it('calculates domestic standard shipping based on weight', () => {
    const order = createOrder({ totalWeightGrams: 1000 });
    const address = createAddress();

    const cost = calculateShipping(order, address, 'standard');

    expect(cost).toBe(599); // base rate for 1kg domestic
  });

  it('returns zero for orders over $100 (free shipping)', () => {
    const order = createOrder({ totalInCents: 15000 });
    const address = createAddress();

    const cost = calculateShipping(order, address, 'standard');

    expect(cost).toBe(0);
  });

  it('charges higher rate for express shipping', () => {
    const order = createOrder({ totalWeightGrams: 1000 });
    const address = createAddress();

    const standard = calculateShipping(order, address, 'standard');
    const express = calculateShipping(order, address, 'express');

    expect(express).toBeGreaterThan(standard);
  });

  it('charges international rate for non-US addresses', () => {
    const order = createOrder({ totalWeightGrams: 1000 });
    const domestic = createAddress({ country: 'US' });
    const international = createAddress({ country: 'DE' });

    const domesticCost = calculateShipping(order, domestic, 'standard');
    const internationalCost = calculateShipping(order, international, 'standard');

    expect(internationalCost).toBeGreaterThan(domesticCost);
  });

  it('throws InvalidAddressError for unsupported countries', () => {
    const order = createOrder();
    const address = createAddress({ country: 'XX' });

    expect(() => calculateShipping(order, address, 'standard')).toThrow('Unsupported shipping destination');
  });

  it('handles zero-weight orders (digital products)', () => {
    const order = createOrder({ totalWeightGrams: 0 });
    const address = createAddress();

    const cost = calculateShipping(order, address, 'standard');

    expect(cost).toBe(0);
  });
});
```

### React Component — Testing Library
```typescript
import { render, screen, waitFor, within } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { LoginForm } from '../components/LoginForm';

// Mock the auth service
const mockLogin = vi.fn();
vi.mock('../services/auth', () => ({
  useAuth: () => ({
    login: mockLogin,
    isLoading: false,
    error: null,
  }),
}));

describe('LoginForm', () => {
  const user = userEvent.setup();

  beforeEach(() => {
    vi.clearAllMocks();
  });

  // --- Rendering ---

  it('renders email and password fields', () => {
    render(<LoginForm />);

    expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /sign in/i })).toBeInTheDocument();
  });

  it('renders forgot password link', () => {
    render(<LoginForm />);

    expect(screen.getByRole('link', { name: /forgot password/i })).toHaveAttribute(
      'href',
      '/forgot-password'
    );
  });

  // --- Form Interaction ---

  it('allows typing in email and password fields', async () => {
    render(<LoginForm />);

    const emailInput = screen.getByLabelText(/email/i);
    const passwordInput = screen.getByLabelText(/password/i);

    await user.type(emailInput, 'alice@example.com');
    await user.type(passwordInput, 'securePass123');

    expect(emailInput).toHaveValue('alice@example.com');
    expect(passwordInput).toHaveValue('securePass123');
  });

  it('submits form with email and password', async () => {
    mockLogin.mockResolvedValue({ success: true });
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'securePass123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(mockLogin).toHaveBeenCalledWith({
      email: 'alice@example.com',
      password: 'securePass123',
    });
  });

  it('submits form on Enter key press', async () => {
    mockLogin.mockResolvedValue({ success: true });
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'securePass123{enter}');

    expect(mockLogin).toHaveBeenCalled();
  });

  // --- Validation ---

  it('shows error when email is empty on submit', async () => {
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/password/i), 'securePass123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByText(/email is required/i)).toBeInTheDocument();
    expect(mockLogin).not.toHaveBeenCalled();
  });

  it('shows error when password is empty on submit', async () => {
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByText(/password is required/i)).toBeInTheDocument();
    expect(mockLogin).not.toHaveBeenCalled();
  });

  it('shows error for invalid email format', async () => {
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'not-an-email');
    await user.type(screen.getByLabelText(/password/i), 'securePass123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByText(/enter a valid email/i)).toBeInTheDocument();
    expect(mockLogin).not.toHaveBeenCalled();
  });

  // --- Loading State ---

  it('disables submit button while loading', async () => {
    mockLogin.mockImplementation(() => new Promise(() => {})); // never resolves
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'securePass123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() => {
      expect(screen.getByRole('button', { name: /signing in/i })).toBeDisabled();
    });
  });

  it('shows loading spinner during submission', async () => {
    mockLogin.mockImplementation(() => new Promise(() => {}));
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'securePass123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() => {
      expect(screen.getByTestId('loading-spinner')).toBeInTheDocument();
    });
  });

  // --- Error Handling ---

  it('displays server error message on failed login', async () => {
    mockLogin.mockRejectedValue(new Error('Invalid email or password'));
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'wrongPassword');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/invalid email or password/i);
    });
  });

  it('displays generic error on network failure', async () => {
    mockLogin.mockRejectedValue(new Error('Network error'));
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'securePass123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/something went wrong/i);
    });
  });

  it('clears error when user starts typing again', async () => {
    mockLogin.mockRejectedValue(new Error('Invalid email or password'));
    render(<LoginForm />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'wrong');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    await waitFor(() => {
      expect(screen.getByRole('alert')).toBeInTheDocument();
    });

    // Start typing again — error should clear
    await user.type(screen.getByLabelText(/password/i), 'x');
    expect(screen.queryByRole('alert')).not.toBeInTheDocument();
  });

  // --- Accessibility ---

  it('focuses email field on mount', () => {
    render(<LoginForm />);

    expect(screen.getByLabelText(/email/i)).toHaveFocus();
  });

  it('moves focus to first error field on validation failure', async () => {
    render(<LoginForm />);

    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByLabelText(/email/i)).toHaveFocus();
  });

  it('password field has type="password"', () => {
    render(<LoginForm />);

    expect(screen.getByLabelText(/password/i)).toHaveAttribute('type', 'password');
  });
});
```

### Python — Pytest
```python
import pytest
from datetime import datetime, timedelta
from unittest.mock import AsyncMock, MagicMock, patch
from decimal import Decimal

from app.services.order_service import OrderService
from app.models.order import Order, OrderStatus, CreateOrderInput, OrderItem
from app.exceptions import ValidationError, NotFoundError, ForbiddenError, ConflictError


# --- Fixtures ---

@pytest.fixture
def mock_order_repo():
    repo = AsyncMock()
    return repo


@pytest.fixture
def mock_payment_service():
    service = AsyncMock()
    return service


@pytest.fixture
def mock_notification_service():
    service = AsyncMock()
    return service


@pytest.fixture
def order_service(mock_order_repo, mock_payment_service, mock_notification_service):
    return OrderService(
        order_repo=mock_order_repo,
        payment_service=mock_payment_service,
        notification_service=mock_notification_service,
    )


@pytest.fixture
def valid_order_input():
    return CreateOrderInput(
        user_id="user-abc-123",
        items=[
            OrderItem(product_id="prod-001", quantity=2, price_in_cents=2999),
            OrderItem(product_id="prod-002", quantity=1, price_in_cents=4999),
        ],
        shipping_address={
            "street": "123 Main St",
            "city": "Springfield",
            "state": "IL",
            "zip": "62701",
            "country": "US",
        },
    )


@pytest.fixture
def mock_order():
    return Order(
        id="order-xyz-789",
        user_id="user-abc-123",
        status=OrderStatus.PENDING,
        total_in_cents=10997,
        items=[],
        created_at=datetime(2026, 1, 15, 10, 0, 0),
        updated_at=datetime(2026, 1, 15, 10, 0, 0),
    )


# --- Create Order Tests ---

class TestCreateOrder:
    """Tests for OrderService.create_order"""

    async def test_creates_order_with_correct_total(
        self, order_service, valid_order_input, mock_order, mock_order_repo, mock_payment_service
    ):
        mock_order_repo.create.return_value = mock_order
        mock_payment_service.authorize.return_value = {"transaction_id": "txn-001"}

        result = await order_service.create_order(valid_order_input)

        assert result.total_in_cents == 10997  # (2999 * 2) + (4999 * 1)

    async def test_authorizes_payment_before_saving(
        self, order_service, valid_order_input, mock_order, mock_order_repo, mock_payment_service
    ):
        call_order = []
        mock_payment_service.authorize.side_effect = lambda *a, **kw: call_order.append("payment")
        mock_order_repo.create.side_effect = lambda *a, **kw: (call_order.append("save"), mock_order)[1]

        await order_service.create_order(valid_order_input)

        assert call_order == ["payment", "save"]

    async def test_sends_confirmation_notification(
        self, order_service, valid_order_input, mock_order, mock_order_repo,
        mock_payment_service, mock_notification_service
    ):
        mock_order_repo.create.return_value = mock_order
        mock_payment_service.authorize.return_value = {"transaction_id": "txn-001"}

        await order_service.create_order(valid_order_input)

        mock_notification_service.send_order_confirmation.assert_called_once()

    # Edge Cases

    async def test_applies_free_shipping_for_orders_over_100_dollars(
        self, order_service, mock_order, mock_order_repo, mock_payment_service
    ):
        input_data = CreateOrderInput(
            user_id="user-abc-123",
            items=[OrderItem(product_id="prod-001", quantity=1, price_in_cents=15000)],
            shipping_address={"country": "US", "zip": "90210"},
        )
        mock_order_repo.create.return_value = mock_order
        mock_payment_service.authorize.return_value = {"transaction_id": "txn-001"}

        await order_service.create_order(input_data)

        create_call_args = mock_order_repo.create.call_args[0][0]
        assert create_call_args.shipping_cost_in_cents == 0

    # Error Cases

    async def test_raises_validation_error_when_items_empty(self, order_service):
        input_data = CreateOrderInput(
            user_id="user-abc-123",
            items=[],
            shipping_address={"country": "US"},
        )

        with pytest.raises(ValidationError, match="at least one item"):
            await order_service.create_order(input_data)

    async def test_raises_validation_error_when_quantity_is_zero(self, order_service):
        input_data = CreateOrderInput(
            user_id="user-abc-123",
            items=[OrderItem(product_id="prod-001", quantity=0, price_in_cents=999)],
            shipping_address={"country": "US"},
        )

        with pytest.raises(ValidationError, match="Quantity must be at least 1"):
            await order_service.create_order(input_data)

    async def test_does_not_save_order_when_payment_fails(
        self, order_service, valid_order_input, mock_order_repo, mock_payment_service
    ):
        mock_payment_service.authorize.side_effect = Exception("Card declined")

        with pytest.raises(Exception, match="Card declined"):
            await order_service.create_order(valid_order_input)

        mock_order_repo.create.assert_not_called()

    async def test_rolls_back_payment_when_save_fails(
        self, order_service, valid_order_input, mock_order_repo, mock_payment_service
    ):
        mock_payment_service.authorize.return_value = {"transaction_id": "txn-001"}
        mock_order_repo.create.side_effect = Exception("Database connection lost")

        with pytest.raises(Exception, match="Database connection lost"):
            await order_service.create_order(valid_order_input)

        mock_payment_service.refund.assert_called_once_with("txn-001")


# --- Cancel Order Tests ---

class TestCancelOrder:
    """Tests for OrderService.cancel_order"""

    async def test_updates_status_to_cancelled(
        self, order_service, mock_order, mock_order_repo
    ):
        mock_order_repo.find_by_id.return_value = mock_order
        cancelled_order = Order(**{**mock_order.__dict__, "status": OrderStatus.CANCELLED})
        mock_order_repo.update.return_value = cancelled_order

        result = await order_service.cancel_order("order-xyz-789", "user-abc-123")

        assert result.status == OrderStatus.CANCELLED

    async def test_raises_not_found_for_nonexistent_order(
        self, order_service, mock_order_repo
    ):
        mock_order_repo.find_by_id.return_value = None

        with pytest.raises(NotFoundError, match="Order not found"):
            await order_service.cancel_order("nonexistent", "user-abc-123")

    async def test_raises_forbidden_when_user_does_not_own_order(
        self, order_service, mock_order, mock_order_repo
    ):
        mock_order.user_id = "different-user"
        mock_order_repo.find_by_id.return_value = mock_order

        with pytest.raises(ForbiddenError):
            await order_service.cancel_order("order-xyz-789", "user-abc-123")

    async def test_raises_conflict_when_order_already_shipped(
        self, order_service, mock_order, mock_order_repo
    ):
        mock_order.status = OrderStatus.SHIPPED
        mock_order_repo.find_by_id.return_value = mock_order

        with pytest.raises(ConflictError, match="already been shipped"):
            await order_service.cancel_order("order-xyz-789", "user-abc-123")

    @pytest.mark.parametrize(
        "invalid_status",
        [OrderStatus.SHIPPED, OrderStatus.DELIVERED, OrderStatus.CANCELLED],
    )
    async def test_raises_conflict_for_non_cancellable_statuses(
        self, order_service, mock_order, mock_order_repo, invalid_status
    ):
        mock_order.status = invalid_status
        mock_order_repo.find_by_id.return_value = mock_order

        with pytest.raises(ConflictError):
            await order_service.cancel_order("order-xyz-789", "user-abc-123")
```

### Go — Standard Testing
```go
package order_test

import (
	"context"
	"errors"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
	"github.com/stretchr/testify/require"
	"myapp/internal/order"
)

// --- Mocks ---

type MockOrderRepo struct {
	mock.Mock
}

func (m *MockOrderRepo) Create(ctx context.Context, o *order.Order) (*order.Order, error) {
	args := m.Called(ctx, o)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*order.Order), args.Error(1)
}

func (m *MockOrderRepo) FindByID(ctx context.Context, id string) (*order.Order, error) {
	args := m.Called(ctx, id)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*order.Order), args.Error(1)
}

type MockPaymentService struct {
	mock.Mock
}

func (m *MockPaymentService) Authorize(ctx context.Context, amount int, currency string) (string, error) {
	args := m.Called(ctx, amount, currency)
	return args.String(0), args.Error(1)
}

func (m *MockPaymentService) Refund(ctx context.Context, txnID string) error {
	args := m.Called(ctx, txnID)
	return args.Error(0)
}

// --- Helpers ---

func newValidInput() order.CreateInput {
	return order.CreateInput{
		UserID: "user-abc-123",
		Items: []order.ItemInput{
			{ProductID: "prod-001", Quantity: 2, PriceInCents: 2999},
			{ProductID: "prod-002", Quantity: 1, PriceInCents: 4999},
		},
		ShippingCountry: "US",
	}
}

func newMockOrder() *order.Order {
	return &order.Order{
		ID:            "order-xyz-789",
		UserID:        "user-abc-123",
		Status:        order.StatusPending,
		TotalInCents:  10997,
	}
}

// --- Tests ---

func TestCreateOrder(t *testing.T) {
	t.Run("creates order with correct total", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		payment.On("Authorize", mock.Anything, 10997, "USD").Return("txn-001", nil)
		repo.On("Create", mock.Anything, mock.AnythingOfType("*order.Order")).Return(newMockOrder(), nil)

		result, err := svc.CreateOrder(context.Background(), newValidInput())

		require.NoError(t, err)
		assert.Equal(t, 10997, result.TotalInCents)
		repo.AssertExpectations(t)
		payment.AssertExpectations(t)
	})

	t.Run("returns error when items slice is empty", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		input := newValidInput()
		input.Items = []order.ItemInput{}

		_, err := svc.CreateOrder(context.Background(), input)

		require.Error(t, err)
		assert.Contains(t, err.Error(), "at least one item")
		repo.AssertNotCalled(t, "Create")
	})

	t.Run("does not save order when payment fails", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		payment.On("Authorize", mock.Anything, mock.Anything, mock.Anything).
			Return("", errors.New("card declined"))

		_, err := svc.CreateOrder(context.Background(), newValidInput())

		require.Error(t, err)
		assert.Contains(t, err.Error(), "card declined")
		repo.AssertNotCalled(t, "Create")
	})

	t.Run("refunds payment when save fails", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		payment.On("Authorize", mock.Anything, mock.Anything, mock.Anything).Return("txn-001", nil)
		repo.On("Create", mock.Anything, mock.Anything).Return(nil, errors.New("db error"))
		payment.On("Refund", mock.Anything, "txn-001").Return(nil)

		_, err := svc.CreateOrder(context.Background(), newValidInput())

		require.Error(t, err)
		payment.AssertCalled(t, "Refund", mock.Anything, "txn-001")
	})

	t.Run("applies free shipping for orders over 10000 cents", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		input := order.CreateInput{
			UserID: "user-abc-123",
			Items: []order.ItemInput{
				{ProductID: "prod-001", Quantity: 1, PriceInCents: 15000},
			},
			ShippingCountry: "US",
		}

		payment.On("Authorize", mock.Anything, mock.Anything, mock.Anything).Return("txn-001", nil)
		repo.On("Create", mock.Anything, mock.MatchedBy(func(o *order.Order) bool {
			return o.ShippingCostInCents == 0
		})).Return(newMockOrder(), nil)

		_, err := svc.CreateOrder(context.Background(), input)

		require.NoError(t, err)
		repo.AssertExpectations(t)
	})
}

func TestCancelOrder(t *testing.T) {
	t.Run("cancels pending order successfully", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		pending := newMockOrder()
		pending.Status = order.StatusPending
		repo.On("FindByID", mock.Anything, "order-xyz-789").Return(pending, nil)
		repo.On("Update", mock.Anything, mock.Anything).Return(nil)

		err := svc.CancelOrder(context.Background(), "order-xyz-789", "user-abc-123")

		require.NoError(t, err)
	})

	t.Run("returns error when order not found", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		repo.On("FindByID", mock.Anything, "nonexistent").Return(nil, order.ErrNotFound)

		err := svc.CancelOrder(context.Background(), "nonexistent", "user-abc-123")

		require.ErrorIs(t, err, order.ErrNotFound)
	})

	t.Run("returns error when user does not own order", func(t *testing.T) {
		t.Parallel()
		repo := new(MockOrderRepo)
		payment := new(MockPaymentService)
		svc := order.NewService(repo, payment)

		o := newMockOrder()
		o.UserID = "different-user"
		repo.On("FindByID", mock.Anything, "order-xyz-789").Return(o, nil)

		err := svc.CancelOrder(context.Background(), "order-xyz-789", "user-abc-123")

		require.ErrorIs(t, err, order.ErrForbidden)
	})
}

// --- Table-Driven Tests ---

func TestCalculateTotal(t *testing.T) {
	tests := []struct {
		name     string
		items    []order.ItemInput
		expected int
	}{
		{
			name:     "single item",
			items:    []order.ItemInput{{Quantity: 1, PriceInCents: 999}},
			expected: 999,
		},
		{
			name:     "multiple items",
			items:    []order.ItemInput{{Quantity: 2, PriceInCents: 500}, {Quantity: 1, PriceInCents: 300}},
			expected: 1300,
		},
		{
			name:     "empty items",
			items:    []order.ItemInput{},
			expected: 0,
		},
		{
			name:     "large quantity",
			items:    []order.ItemInput{{Quantity: 99, PriceInCents: 100}},
			expected: 9900,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()
			result := order.CalculateTotal(tt.items)
			assert.Equal(t, tt.expected, result)
		})
	}
}
```

### Ruby — RSpec
```ruby
require 'rails_helper'

RSpec.describe OrderService do
  let(:order_repo) { instance_double(OrderRepository) }
  let(:payment_service) { instance_double(PaymentService) }
  let(:notification_service) { instance_double(NotificationService) }
  let(:service) { described_class.new(order_repo:, payment_service:, notification_service:) }

  let(:valid_input) do
    {
      user_id: "user-abc-123",
      items: [
        { product_id: "prod-001", quantity: 2, price_in_cents: 2999 },
        { product_id: "prod-002", quantity: 1, price_in_cents: 4999 }
      ],
      shipping_address: { country: "US", zip: "90210" }
    }
  end

  let(:mock_order) do
    build(:order,
      id: "order-xyz-789",
      user_id: "user-abc-123",
      status: :pending,
      total_in_cents: 10_997
    )
  end

  describe "#create_order" do
    context "with valid input" do
      before do
        allow(payment_service).to receive(:authorize).and_return("txn-001")
        allow(order_repo).to receive(:create).and_return(mock_order)
        allow(notification_service).to receive(:send_order_confirmation)
      end

      it "creates order with correct total" do
        result = service.create_order(valid_input)

        expect(result.total_in_cents).to eq(10_997)
      end

      it "authorizes payment before saving" do
        service.create_order(valid_input)

        expect(payment_service).to have_received(:authorize).ordered
        expect(order_repo).to have_received(:create).ordered
      end

      it "sends confirmation notification" do
        service.create_order(valid_input)

        expect(notification_service).to have_received(:send_order_confirmation)
          .with(hash_including(order_id: "order-xyz-789"))
      end
    end

    context "with empty items" do
      it "raises ValidationError" do
        input = valid_input.merge(items: [])

        expect { service.create_order(input) }
          .to raise_error(ValidationError, /at least one item/)
      end
    end

    context "when payment fails" do
      before do
        allow(payment_service).to receive(:authorize).and_raise("Card declined")
      end

      it "does not save the order" do
        expect { service.create_order(valid_input) }.to raise_error(RuntimeError)
        expect(order_repo).not_to have_received(:create)
      end
    end

    context "when save fails after payment" do
      before do
        allow(payment_service).to receive(:authorize).and_return("txn-001")
        allow(payment_service).to receive(:refund)
        allow(order_repo).to receive(:create).and_raise("DB error")
      end

      it "refunds the payment" do
        expect { service.create_order(valid_input) }.to raise_error(RuntimeError)
        expect(payment_service).to have_received(:refund).with("txn-001")
      end
    end
  end

  describe "#cancel_order" do
    context "when order exists and user owns it" do
      let(:pending_order) { build(:order, user_id: "user-abc-123", status: :pending) }

      before do
        allow(order_repo).to receive(:find_by_id).and_return(pending_order)
        allow(order_repo).to receive(:update)
      end

      it "updates status to cancelled" do
        service.cancel_order("order-xyz-789", "user-abc-123")

        expect(order_repo).to have_received(:update)
          .with(hash_including(status: :cancelled))
      end
    end

    context "when order does not exist" do
      before { allow(order_repo).to receive(:find_by_id).and_return(nil) }

      it "raises NotFoundError" do
        expect { service.cancel_order("nonexistent", "user-abc-123") }
          .to raise_error(NotFoundError)
      end
    end

    context "when user does not own the order" do
      let(:other_order) { build(:order, user_id: "different-user") }
      before { allow(order_repo).to receive(:find_by_id).and_return(other_order) }

      it "raises ForbiddenError" do
        expect { service.cancel_order("order-xyz-789", "user-abc-123") }
          .to raise_error(ForbiddenError)
      end
    end

    %i[shipped delivered cancelled].each do |status|
      context "when order status is #{status}" do
        let(:order) { build(:order, user_id: "user-abc-123", status:) }
        before { allow(order_repo).to receive(:find_by_id).and_return(order) }

        it "raises ConflictError" do
          expect { service.cancel_order("order-xyz-789", "user-abc-123") }
            .to raise_error(ConflictError)
        end
      end
    end
  end
end
```

### PHP — PHPUnit / Pest
```php
<?php

use Tests\TestCase;
use App\Services\OrderService;
use App\Models\Order;
use App\Exceptions\ValidationException;
use App\Exceptions\ForbiddenException;
use Illuminate\Foundation\Testing\RefreshDatabase;

class OrderServiceTest extends TestCase
{
    use RefreshDatabase;

    private OrderService $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = app(OrderService::class);
    }

    private function validInput(array $overrides = []): array
    {
        return array_merge([
            'user_id' => 'user-abc-123',
            'items' => [
                ['product_id' => 'prod-001', 'quantity' => 2, 'price_in_cents' => 2999],
                ['product_id' => 'prod-002', 'quantity' => 1, 'price_in_cents' => 4999],
            ],
            'shipping_address' => ['country' => 'US', 'zip' => '90210'],
        ], $overrides);
    }

    /** @test */
    public function it_creates_order_with_correct_total(): void
    {
        $result = $this->service->createOrder($this->validInput());

        $this->assertEquals(10997, $result->total_in_cents);
    }

    /** @test */
    public function it_throws_validation_error_when_items_empty(): void
    {
        $this->expectException(ValidationException::class);
        $this->expectExceptionMessage('at least one item');

        $this->service->createOrder($this->validInput(['items' => []]));
    }

    /** @test */
    public function it_throws_forbidden_when_cancelling_other_users_order(): void
    {
        $order = Order::factory()->create(['user_id' => 'different-user']);

        $this->expectException(ForbiddenException::class);

        $this->service->cancelOrder($order->id, 'user-abc-123');
    }

    /** @test */
    public function it_applies_free_shipping_over_100_dollars(): void
    {
        $input = $this->validInput([
            'items' => [
                ['product_id' => 'prod-001', 'quantity' => 1, 'price_in_cents' => 15000],
            ],
        ]);

        $result = $this->service->createOrder($input);

        $this->assertEquals(0, $result->shipping_cost_in_cents);
    }
}
```

## 4. API / Integration Test Patterns

### REST API Tests — Node.js (Supertest)
```typescript
import request from 'supertest';
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { app } from '../app';
import { db } from '../db';
import { createTestUser, generateAuthToken } from './helpers';

describe('POST /api/v1/orders', () => {
  let authToken: string;
  let testUser: { id: string; email: string };

  beforeAll(async () => {
    await db.migrate.latest();
  });

  beforeEach(async () => {
    await db('orders').truncate();
    testUser = await createTestUser();
    authToken = generateAuthToken(testUser.id);
  });

  afterAll(async () => {
    await db.destroy();
  });

  it('creates an order and returns 201', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        items: [{ productId: 'prod-001', quantity: 2 }],
        shippingAddress: { country: 'US', zip: '90210' },
      });

    expect(response.status).toBe(201);
    expect(response.body).toMatchObject({
      id: expect.any(String),
      status: 'pending',
      items: expect.arrayContaining([
        expect.objectContaining({ productId: 'prod-001', quantity: 2 }),
      ]),
    });
  });

  it('returns 401 without auth token', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .send({ items: [{ productId: 'prod-001', quantity: 1 }] });

    expect(response.status).toBe(401);
    expect(response.body.error).toBe('UNAUTHORIZED');
  });

  it('returns 400 with empty items', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ items: [], shippingAddress: { country: 'US' } });

    expect(response.status).toBe(400);
    expect(response.body.error).toMatch(/at least one item/i);
  });

  it('returns 400 with invalid product ID', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        items: [{ productId: 'nonexistent', quantity: 1 }],
        shippingAddress: { country: 'US' },
      });

    expect(response.status).toBe(400);
    expect(response.body.error).toMatch(/product not found/i);
  });

  it('returns 429 when rate limit exceeded', async () => {
    // Send requests up to the limit
    for (let i = 0; i < 100; i++) {
      await request(app)
        .post('/api/v1/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ items: [{ productId: 'prod-001', quantity: 1 }] });
    }

    const response = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ items: [{ productId: 'prod-001', quantity: 1 }] });

    expect(response.status).toBe(429);
  });
});
```

### REST API Tests — Python (FastAPI / httpx)
```python
import pytest
from httpx import AsyncClient
from app.main import app
from app.db import get_db, reset_db
from tests.factories import create_test_user, generate_auth_token


@pytest.fixture(autouse=True)
async def clean_db():
    await reset_db()
    yield


@pytest.fixture
async def client():
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client


@pytest.fixture
async def auth_headers():
    user = await create_test_user()
    token = generate_auth_token(user.id)
    return {"Authorization": f"Bearer {token}"}


class TestCreateOrder:
    async def test_creates_order_returns_201(self, client, auth_headers):
        response = await client.post(
            "/api/v1/orders",
            headers=auth_headers,
            json={
                "items": [{"product_id": "prod-001", "quantity": 2}],
                "shipping_address": {"country": "US", "zip": "90210"},
            },
        )

        assert response.status_code == 201
        data = response.json()
        assert data["status"] == "pending"
        assert len(data["items"]) == 1

    async def test_returns_401_without_token(self, client):
        response = await client.post(
            "/api/v1/orders",
            json={"items": [{"product_id": "prod-001", "quantity": 1}]},
        )

        assert response.status_code == 401

    async def test_returns_400_with_empty_items(self, client, auth_headers):
        response = await client.post(
            "/api/v1/orders",
            headers=auth_headers,
            json={"items": [], "shipping_address": {"country": "US"}},
        )

        assert response.status_code == 400
        assert "at least one item" in response.json()["detail"].lower()
```

## 5. Test Checklist by Code Type

### For a Service/Business Logic Function
- [ ] Happy path with valid input
- [ ] Each validation rule (one test per rule)
- [ ] Null/undefined/empty input
- [ ] Boundary values (min, max, zero, negative)
- [ ] Error thrown by each dependency
- [ ] Return value shape and types
- [ ] Side effects (DB writes, API calls, events emitted)
- [ ] Idempotency (if applicable)
- [ ] Concurrency (if applicable)

### For an API Endpoint
- [ ] Success response (status code, body shape)
- [ ] Authentication required (401 without token)
- [ ] Authorization (403 for wrong role)
- [ ] Input validation (400 for each invalid field)
- [ ] Resource not found (404)
- [ ] Conflict/duplicate (409)
- [ ] Rate limiting (429)
- [ ] Request body too large (413)
- [ ] Correct content-type header
- [ ] Pagination (if list endpoint)
- [ ] Filtering and sorting (if applicable)

### For a UI Component
- [ ] Renders without errors
- [ ] Displays correct initial state
- [ ] Responds to user interactions (click, type, submit)
- [ ] Shows loading state
- [ ] Shows error state
- [ ] Shows empty state
- [ ] Form validation (each rule)
- [ ] Keyboard navigation / accessibility
- [ ] Conditional rendering (shows/hides elements)
- [ ] Calls correct callbacks with correct arguments

### For a Database Query/Repository
- [ ] Returns correct data for valid query
- [ ] Returns empty for no matches
- [ ] Handles multiple results
- [ ] Pagination works correctly
- [ ] Filters apply correctly
- [ ] Sorting works correctly
- [ ] Handles null/optional fields
- [ ] Transaction commits on success
- [ ] Transaction rolls back on failure
- [ ] Concurrent access doesn't corrupt data

### For a Utility/Helper Function
- [ ] Normal input → correct output
- [ ] Edge case inputs (empty string, zero, negative, max int)
- [ ] Type coercion (string "123" vs number 123)
- [ ] Unicode and special characters
- [ ] Very large input
- [ ] Null/undefined input
- [ ] Return type is consistent

## 6. Test Data Strategies

### Factory Functions (Recommended)
```typescript
// Create reusable, customizable test data
const createUser = (overrides?: Partial<User>): User => ({
  id: `user-${Math.random().toString(36).slice(2, 8)}`,
  email: `test-${Date.now()}@example.com`,
  name: 'Test User',
  role: 'editor',
  createdAt: new Date(),
  ...overrides,
});

// Usage
const admin = createUser({ role: 'admin' });
const inactive = createUser({ status: 'inactive', lastLoginAt: null });
```

### Fixtures for Complex Data
```typescript
// fixtures/orders.ts
export const sampleOrders = {
  simple: { /* ... */ },
  withDiscount: { /* ... */ },
  international: { /* ... */ },
  maxItems: { /* ... */ },
};
```

### Rules for Test Data
- Use realistic data, not "test" and "123"
- Generate unique IDs to prevent collisions
- Keep factories close to test files or in a shared helpers directory
- Override only what's relevant to the test
- Never depend on specific database IDs

## Output Format

When generating tests, provide:

### 1. Test Plan
```
File: src/services/orderService.ts
Functions to test: createOrder, cancelOrder, getOrderById
Total test cases: 24

createOrder (12 tests):
  ✅ Happy path: 3 tests
  ⚠️ Edge cases: 4 tests
  ❌ Error handling: 5 tests

cancelOrder (8 tests):
  ✅ Happy path: 2 tests
  ⚠️ Edge cases: 2 tests
  ❌ Error handling: 4 tests

getOrderById (4 tests):
  ✅ Happy path: 1 test
  ❌ Error handling: 3 tests
```

### 2. Complete Test File
The full, runnable test file matching the project's conventions.

### 3. Setup Instructions
Any additional setup needed:
- New dev dependencies to install
- Test configuration changes
- Mock files to create
- Fixture data to add

### 4. Run Command
```
npm test -- --testPathPattern=orderService
pytest tests/test_order_service.py -v
go test ./internal/order/ -v -run TestCreateOrder
```

## Adaptation Rules

- **Match existing conventions** — read existing test files first and follow the same patterns
- **Match naming** — use the same naming convention as existing tests (test_, .test., .spec., _test)
- **Match structure** — co-located tests vs test directory, match what exists
- **Match assertion style** — expect vs assert, toBe vs toEqual, follow existing patterns
- **Use existing helpers** — check for test utilities, factories, or fixtures already in the project
- **Don't over-mock** — test real behavior when possible, mock only external dependencies
- **Run the tests** — always verify the generated tests actually pass before presenting them

## Summary

End every test generation with:
1. **Tests created** — count and file location
2. **Coverage added** — which functions/paths are now covered
3. **Gaps remaining** — what's still untested
4. **Run command** — exact command to run the new tests
5. **Dependencies needed** — any packages to install
