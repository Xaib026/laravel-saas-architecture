# 03 - SOLID Principles Applied to Laravel

## Building Maintainable, Extensible Code

SOLID principles are the foundation of good software design. This guide shows how to apply them in Laravel.

---

## S - Single Responsibility Principle

### Definition
A class should have only ONE reason to change. One responsibility = one reason to change.

### ❌ Bad Example

```php
class UserController
{
    public function store(Request $request)
    {
        // Validation
        $validated = $request->validate([...]);

        // Create user
        $user = User::create($validated);

        // Send email
        Mail::to($user->email)->send(new WelcomeEmail($user));

        // Log event
        Log::info('User created', ['user_id' => $user->id]);

        // Generate JWT token
        $token = auth()->attempt($validated);

        // Track analytics
        Analytics::track('user.created', ['user_id' => $user->id]);

        return response()->json(['token' => $token]);
    }
}
```

**Problems**:
- Controller does validation, creation, emailing, logging, auth, analytics
- 6 reasons to change this class
- Hard to test
- Hard to reuse logic

### ✅ Good Example

```php
// 1. Controller - only handles HTTP
class UserController
{
    public function __construct(private RegisterUserUseCase $useCase) {}

    public function store(StoreUserRequest $request)
    {
        $user = $this->useCase->execute(
            RegisterUserDTO::fromRequest($request)
        );

        return response()->json(UserResource::make($user), 201);
    }
}

// 2. Request - only handles validation
class StoreUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8',
        ];
    }
}

// 3. Use Case - only handles business logic
class RegisterUserUseCase
{
    public function __construct(
        private UserRepository $userRepo,
        private EmailService $emailService,
    ) {}

    public function execute(RegisterUserDTO $dto): User
    {
        $user = User::create($dto->toArray());
        $this->userRepo->save($user);
        event(new UserRegistered($user));
        return $user;
    }
}

// 4. Listeners - handle specific concerns
class SendWelcomeEmailListener
{
    public function handle(UserRegistered $event)
    {
        Mail::to($event->user->email)->queue(new WelcomeEmail());
    }
}

class LogUserRegistrationListener
{
    public function handle(UserRegistered $event)
    {
        Log::info('User registered', ['user_id' => $event->user->id]);
    }
}

class TrackUserAnalyticsListener
{
    public function handle(UserRegistered $event)
    {
        Analytics::track('user.created', ['user_id' => $event->user->id]);
    }
}
```

### Benefits
- ✅ Each class has ONE reason to change
- ✅ Easier to understand
- ✅ Easier to test
- ✅ Easier to reuse
- ✅ Easier to maintain

---

## O - Open/Closed Principle

### Definition
Classes should be **open for extension** but **closed for modification**.

Add new features WITHOUT changing existing code.

### ❌ Bad Example

```php
class PaymentProcessor
{
    public function process(Payment $payment): bool
    {
        // Adding new payment methods requires MODIFYING this class
        if ($payment->method === 'stripe') {
            return $this->processStripe($payment);
        } elseif ($payment->method === 'paypal') {
            return $this->processPayPal($payment);
        } elseif ($payment->method === 'square') {
            return $this->processSquare($payment);
        }
        return false;
    }

    private function processStripe(Payment $payment): bool { /* ... */ }
    private function processPayPal(Payment $payment): bool { /* ... */ }
    private function processSquare(Payment $payment): bool { /* ... */ }
}
```

**Problem**: Every new payment method requires modifying the class (CLOSED for extension)

### ✅ Good Example

```php
// Define contract
interface PaymentGateway
{
    public function charge(Money $amount, string $token): PaymentResult;
    public function refund(string $transactionId): RefundResult;
}

// Implementations (can add unlimited without modifying existing code)
class StripePaymentGateway implements PaymentGateway
{
    public function charge(Money $amount, string $token): PaymentResult { /* ... */ }
    public function refund(string $transactionId): RefundResult { /* ... */ }
}

class PayPalPaymentGateway implements PaymentGateway
{
    public function charge(Money $amount, string $token): PaymentResult { /* ... */ }
    public function refund(string $transactionId): RefundResult { /* ... */ }
}

class SquarePaymentGateway implements PaymentGateway
{
    public function charge(Money $amount, string $token): PaymentResult { /* ... */ }
    public function refund(string $transactionId): RefundResult { /* ... */ }
}

// Processor (CLOSED for modification)
class PaymentProcessor
{
    public function __construct(private PaymentGateway $gateway) {}

    public function process(Payment $payment): bool
    {
        return $this->gateway->charge(
            Money::create($payment->amount),
            $payment->token
        )->isSuccessful();
    }
}

// Register different implementations
route()->post('/pay/stripe', fn(Request $r) => 
    app(PaymentProcessor::class, ['gateway' => new StripePaymentGateway()])
);
```

### Benefits
- ✅ Add new payment methods WITHOUT modifying existing code
- ✅ Existing code is stable
- ✅ New features are extensions
- ✅ Reduced risk of breaking changes

---

## L - Liskov Substitution Principle

### Definition
Subtypes MUST be substitutable for their base types without breaking functionality.

If class B extends class A, you should be able to use B everywhere A is used.

### ❌ Bad Example

```php
class Bird
{
    public function fly(): string
    {
        return 'Flying...';
    }
}

class Penguin extends Bird
{
    public function fly(): string
    {
        // Penguins can't fly!
        throw new Exception('Cannot fly');
    }
}

// Usage
$bird = new Penguin();
$bird->fly(); // Throws exception - violates contract!
```

**Problem**: Penguin breaks the Bird contract. You can't substitute Penguin for Bird.

### ✅ Good Example

```php
interface Bird
{
    public function move(): string;
}

class Sparrow implements Bird
{
    public function move(): string
    {
        return 'Flying...';
    }
}

class Penguin implements Bird
{
    public function move(): string
    {
        return 'Swimming...';
    }
}

// Usage
function makeMove(Bird $bird): void
{
    echo $bird->move();
}

makeMove(new Sparrow()); // "Flying..."
makeMove(new Penguin()); // "Swimming..."
// Both work! Liskov satisfied.
```

### SaaS Example

```php
interface SubscriptionPricingCalculator
{
    public function calculate(Subscription $subscription): Money;
}

// Both implementations honor the contract
class StandardPricingCalculator implements SubscriptionPricingCalculator
{
    public function calculate(Subscription $subscription): Money
    {
        return $subscription->plan()->price();
    }
}

class DiscountedPricingCalculator implements SubscriptionPricingCalculator
{
    public function calculate(Subscription $subscription): Money
    {
        $basePrice = $subscription->plan()->price();
        // Apply discount but still return Money
        return $basePrice->multiply(0.9); // 10% off
    }
}

// This works with ANY implementation
class BillingService
{
    public function __construct(
        private SubscriptionPricingCalculator $calculator
    ) {}

    public function generateInvoice(Subscription $sub): Invoice
    {
        $amount = $this->calculator->calculate($sub);
        return Invoice::create(['amount' => $amount]);
    }
}
```

---

## I - Interface Segregation Principle

### Definition
Clients should NOT depend on interfaces they don't use.

Make small, focused interfaces, not fat ones.

### ❌ Bad Example

```php
// Fat interface - forces implementers to implement what they don't need
interface Repository
{
    public function find(string $id);
    public function findAll();
    public function create(array $data);
    public function update(string $id, array $data);
    public function delete(string $id);
    public function paginate(int $page);
    public function export(): string;
    public function import(string $data);
    public function backup();
    public function restore();
}

// This class must implement EVERYTHING even if read-only
class ReadOnlyUserRepository implements Repository
{
    public function find(string $id) { /* ... */ }
    public function findAll() { /* ... */ }
    public function create(array $data) 
    { 
        throw new Exception('Read only'); // Not used!
    }
    public function update(string $id, array $data) 
    { 
        throw new Exception('Read only'); // Not used!
    }
    // ... more exceptions ...
}
```

### ✅ Good Example

```php
// Segregated interfaces
interface Readable
{
    public function find(string $id);
    public function findAll();
}

interface Writable
{
    public function create(array $data);
    public function update(string $id, array $data);
    public function delete(string $id);
}

interface Exportable
{
    public function export(): string;
}

interface Importable
{
    public function import(string $data);
}

// Implement only what's needed
class ReadOnlyUserRepository implements Readable
{
    public function find(string $id) { /* ... */ }
    public function findAll() { /* ... */ }
}

class FullUserRepository implements Readable, Writable, Exportable
{
    public function find(string $id) { /* ... */ }
    public function findAll() { /* ... */ }
    public function create(array $data) { /* ... */ }
    public function update(string $id, array $data) { /* ... */ }
    public function delete(string $id) { /* ... */ }
    public function export(): string { /* ... */ }
}

// Usage
class UserService
{
    // Only depends on what it uses
    public function __construct(private Readable $userRepo) {}

    public function getUser(string $id)
    {
        return $this->userRepo->find($id);
    }
}
```

### Benefits
- ✅ Classes implement only what they need
- ✅ No fake implementations
- ✅ Smaller, focused interfaces
- ✅ Better flexibility

---

## D - Dependency Inversion Principle

### Definition
Depend on abstractions (interfaces), NOT concrete implementations.

High-level modules should NOT depend on low-level modules. Both should depend on abstractions.

### ❌ Bad Example

```php
class UserService
{
    private MySQLUserRepository $repository; // TIGHTLY COUPLED

    public function __construct()
    {
        $this->repository = new MySQLUserRepository(); // Creates dependency
    }

    public function getUser(string $id)
    {
        return $this->repository->find($id);
    }
}

// Can't use different database without changing UserService
```

### ✅ Good Example

```php
// Depend on interface, not implementation
interface UserRepository
{
    public function find(string $id);
}

class UserService
{
    public function __construct(private UserRepository $repository) {}

    public function getUser(string $id)
    {
        return $this->repository->find($id);
    }
}

// Inject implementations at runtime
app()->bind(UserRepository::class, MySQLUserRepository::class);
// or
app()->bind(UserRepository::class, PostgresUserRepository::class);
// or
app()->bind(UserRepository::class, MongoUserRepository::class);

// UserService works with ALL implementations
```

### SaaS Example

```php
// Bad: Depends on concrete Stripe class
class SubscriptionService
{
    private StripePaymentGateway $stripe; // CONCRETE

    public function charge(Subscription $sub): bool
    {
        return $this->stripe->charge($sub->amount);
    }
}

// Good: Depends on interface
interface PaymentGateway
{
    public function charge(Money $amount): PaymentResult;
}

class SubscriptionService
{
    public function __construct(private PaymentGateway $gateway) {}

    public function charge(Subscription $sub): bool
    {
        return $this->gateway->charge($sub->amount)->isSuccessful();
    }
}

// Now you can inject ANY payment gateway
app()->bind(PaymentGateway::class, StripePaymentGateway::class);
app()->bind(PaymentGateway::class, PayPalPaymentGateway::class);
app()->bind(PaymentGateway::class, SquarePaymentGateway::class);
```

---

## SOLID Principles Checklist

### Single Responsibility
- [ ] Each class has ONE reason to change
- [ ] Class has ONE responsibility
- [ ] Code is focused and simple

### Open/Closed
- [ ] Can add new features without modifying existing code
- [ ] Use interfaces for extension
- [ ] Use composition over inheritance

### Liskov Substitution
- [ ] Subtypes honor parent contract
- [ ] Can substitute implementations freely
- [ ] No special cases or exceptions

### Interface Segregation
- [ ] Interfaces are focused and small
- [ ] Classes implement only what they need
- [ ] No forced "dummy" implementations

### Dependency Inversion
- [ ] Depend on interfaces, not implementations
- [ ] Use constructor injection
- [ ] Use service container for bindings

---

## How SOLID Principles Help Scale SaaS

| Principle | Benefit for SaaS |
|-----------|------------------|
| SRP | Add features without breaking existing code |
| OCP | Extend for new payment methods/plans without changes |
| LSP | Swap implementations (caching, databases) |
| ISP | Services only depend on what they need |
| DIP | Easy to test with mocked dependencies |

---

Next: [04-CLEAN-CODE.md](./04-CLEAN-CODE.md)
