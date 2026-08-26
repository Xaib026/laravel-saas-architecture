# 02 - Design Patterns for Laravel SaaS

## Essential Design Patterns for Scalable Applications

Design patterns are reusable solutions to common problems. This guide covers the most important patterns for building scalable SaaS applications in Laravel.

---

## 1. Repository Pattern

### Purpose
Decouple business logic from data access. Allows switching databases without changing domain code.

### Implementation

```php
// Domain/Users/UserRepository.php - Interface
namespace App\Domain\Users;

interface UserRepository
{
    public function findById(string $id): ?User;
    public function findByEmail(string $email): ?User;
    public function findActive(): Collection;
    public function save(User $user): void;
    public function delete(User $user): void;
}

// Infrastructure/Repositories/EloquentUserRepository.php - Implementation
namespace App\Infrastructure\Repositories;

use App\Domain\Users\User as DomainUser;
use App\Domain\Users\UserRepository;
use App\Models\User as UserModel;

class EloquentUserRepository implements UserRepository
{
    public function findById(string $id): ?DomainUser
    {
        $model = UserModel::find($id);
        return $model ? $this->toDomain($model) : null;
    }

    public function findByEmail(string $email): ?DomainUser
    {
        $model = UserModel::where('email', $email)->first();
        return $model ? $this->toDomain($model) : null;
    }

    public function findActive(): Collection
    {
        return UserModel::active()
            ->get()
            ->map(fn($model) => $this->toDomain($model));
    }

    public function save(DomainUser $user): void
    {
        UserModel::updateOrCreate(
            ['id' => $user->id()],
            $user->toArray()
        );
    }

    public function delete(DomainUser $user): void
    {
        UserModel::destroy($user->id());
    }

    private function toDomain(UserModel $model): DomainUser
    {
        return new DomainUser(
            id: $model->id,
            email: $model->email,
            name: $model->name,
            // ... other fields
        );
    }
}

// Usage in Service Provider
app()->bind(UserRepository::class, EloquentUserRepository::class);
```

### Benefits
- ✅ Easy to test (mock repository)
- ✅ Easy to switch databases
- ✅ Single source of truth for queries
- ✅ Business logic independent of database

---

## 2. Service Layer Pattern

### Purpose
Encapsulate business logic, orchestrate multiple repositories, handle transactions.

### Implementation

```php
// Application/Services/SubscriptionService.php
namespace App\Application\Services;

use App\Application\DTO\CreateSubscriptionDTO;
use App\Domain\Subscriptions\Subscription;
use App\Domain\Subscriptions\SubscriptionRepository;
use App\Domain\Users\UserRepository;
use Illuminate\Support\Facades\DB;

class SubscriptionService
{
    public function __construct(
        private SubscriptionRepository $subscriptionRepo,
        private UserRepository $userRepo,
        private PaymentService $paymentService,
    ) {}

    public function createSubscription(CreateSubscriptionDTO $dto): Subscription
    {
        return DB::transaction(function () use ($dto) {
            // Validate user
            $user = $this->userRepo->findById($dto->userId);
            if (!$user) {
                throw new UserNotFoundException();
            }

            // Create subscription
            $subscription = Subscription::create(
                userId: $dto->userId,
                planId: $dto->planId,
                startsAt: now(),
            );

            // Process payment
            $payment = $this->paymentService->processPayment(
                userId: $dto->userId,
                amount: $dto->amount,
                planId: $dto->planId,
            );

            if (!$payment->isSuccessful()) {
                throw new PaymentFailedException();
            }

            // Save subscription
            $this->subscriptionRepo->save($subscription);

            // Dispatch event (handled by listeners)
            event(new SubscriptionCreated($subscription));

            return $subscription;
        });
    }
}
```

### Best Practices
- ✅ One responsibility per service
- ✅ Inject dependencies via constructor
- ✅ Use transactions for critical operations
- ✅ Throw domain exceptions
- ✅ Dispatch events for side effects

---

## 3. Data Transfer Object (DTO) Pattern

### Purpose
Safely transfer data between layers with validation and type safety.

### Implementation

```php
// Application/DTO/CreateSubscriptionDTO.php
namespace App\Application\DTO;

use App\Shared\Exceptions\ValidationException;

final class CreateSubscriptionDTO
{
    public function __construct(
        public readonly string $userId,
        public readonly string $planId,
        public readonly string $billingEmail,
        public readonly float $amount,
    ) {
        $this->validate();
    }

    private function validate(): void
    {
        if (empty($this->userId)) {
            throw new ValidationException('User ID is required');
        }

        if (empty($this->planId)) {
            throw new ValidationException('Plan ID is required');
        }

        if ($this->amount <= 0) {
            throw new ValidationException('Amount must be positive');
        }
    }

    public static function fromRequest(Request $request): self
    {
        return new self(
            userId: auth()->id(),
            planId: $request->input('plan_id'),
            billingEmail: $request->input('billing_email'),
            amount: $request->input('amount'),
        );
    }
}

// Usage
$dto = CreateSubscriptionDTO::fromRequest($request);
$subscription = $useCase->execute($dto);
```

### Benefits
- ✅ Immutable data transfer
- �� Validation in one place
- ✅ Type-safe (PHP 8+)
- ✅ Self-documenting
- ✅ Easy to extend

---

## 4. Value Object Pattern

### Purpose
Represent domain concepts as immutable objects (Email, Money, UUID, etc.)

### Implementation

```php
// Domain/Shared/ValueObjects/Email.php
namespace App\Domain\Shared\ValueObjects;

use App\Domain\Shared\Exceptions\InvalidEmailException;

final class Email
{
    private function __construct(private readonly string $value) {}

    public static function create(string $value): self
    {
        $value = strtolower(trim($value));

        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException("{$value} is not a valid email");
        }

        return new self($value);
    }

    public function value(): string
    {
        return $this->value;
    }

    public function equals(Email $other): bool
    {
        return $this->value === $other->value();
    }

    public function __toString(): string
    {
        return $this->value;
    }
}

// Usage
$email = Email::create('user@example.com');
echo $email; // user@example.com
$email->equals(Email::create('user@example.com')); // true
```

### Common Value Objects for SaaS
- Email
- Money
- UUID
- Phone Number
- Address
- Date Range
- Subscription Status

---

## 5. Factory Pattern

### Purpose
Encapsulate complex object creation logic.

### Implementation

```php
// Infrastructure/Factories/UserFactory.php
namespace App\Infrastructure\Factories;

use App\Domain\Users\User;
use App\Domain\Shared\ValueObjects\Email;
use Illuminate\Support\Str;

class UserFactory
{
    public function createFromSignUp(array $data): User
    {
        return new User(
            id: Str::uuid(),
            email: Email::create($data['email']),
            name: $data['name'],
            password: bcrypt($data['password']),
            teamId: Str::uuid(), // Default team
            emailVerifiedAt: null,
            createdAt: now(),
        );
    }

    public function createFromSocialAuth(array $profile): User
    {
        return new User(
            id: Str::uuid(),
            email: Email::create($profile['email']),
            name: $profile['name'],
            password: null, // Social auth
            teamId: Str::uuid(),
            emailVerifiedAt: now(), // Auto-verified
            socialProvider: $profile['provider'],
            socialId: $profile['id'],
            createdAt: now(),
        );
    }
}

// Usage
$factory = app(UserFactory::class);
$user = $factory->createFromSignUp([
    'email' => 'user@example.com',
    'name' => 'John Doe',
    'password' => 'secret',
]);
```

---

## 6. Strategy Pattern

### Purpose
Encapsulate different algorithms that can be swapped at runtime.

### Implementation

```php
// Domain/Subscriptions/PricingStrategy.php - Interface
interface PricingStrategy
{
    public function calculate(Subscription $subscription): Money;
    public function supports(Plan $plan): bool;
}

// Infrastructure/Pricing/MonthlyPricingStrategy.php
class MonthlyPricingStrategy implements PricingStrategy
{
    public function calculate(Subscription $subscription): Money
    {
        return Money::create($subscription->plan()->monthlyPrice());
    }

    public function supports(Plan $plan): bool
    {
        return $plan->billingCycle() === BillingCycle::MONTHLY;
    }
}

// Infrastructure/Pricing/AnnualPricingStrategy.php
class AnnualPricingStrategy implements PricingStrategy
{
    public function calculate(Subscription $subscription): Money
    {
        $annualPrice = $subscription->plan()->annualPrice();
        // Apply discount for annual billing
        $discount = $annualPrice * 0.10;
        return Money::create($annualPrice - $discount);
    }

    public function supports(Plan $plan): bool
    {
        return $plan->billingCycle() === BillingCycle::ANNUAL;
    }
}

// Usage
class PricingCalculator
{
    public function __construct(private array $strategies) {}

    public function calculatePrice(Subscription $subscription): Money
    {
        foreach ($this->strategies as $strategy) {
            if ($strategy->supports($subscription->plan())) {
                return $strategy->calculate($subscription);
            }
        }
        throw new UnsupportedBillingCycleException();
    }
}
```

---

## 7. Observer/Event Pattern

### Purpose
Decouple event producers from handlers. Enable extensible event handling.

### Implementation

```php
// Domain/Users/Events/UserRegistered.php
namespace App\Domain\Users\Events;

use App\Domain\Users\User;

class UserRegistered
{
    public function __construct(public readonly User $user) {}
}

// Infrastructure/Listeners/SendWelcomeEmailListener.php
namespace App\Infrastructure\Listeners;

use App\Domain\Users\Events\UserRegistered;
use App\Infrastructure\Mail\WelcomeEmail;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmailListener
{
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email())
            ->queue(new WelcomeEmail($event->user));
    }
}

// Infrastructure/Listeners/LogUserRegistrationListener.php
class LogUserRegistrationListener
{
    public function handle(UserRegistered $event): void
    {
        Log::info('User registered', [
            'user_id' => $event->user->id(),
            'email' => $event->user->email(),
        ]);
    }
}

// Register in EventServiceProvider
protected $listen = [
    UserRegistered::class => [
        SendWelcomeEmailListener::class,
        LogUserRegistrationListener::class,
    ],
];

// Dispatch event
event(new UserRegistered($user));
```

### Benefits
- ✅ Decoupled event handling
- ✅ Easy to add new listeners
- ✅ Async processing possible
- ✅ Maintains single responsibility

---

## 8. Decorator Pattern

### Purpose
Add functionality to objects without modifying their structure.

### Implementation

```php
// Infrastructure/Repositories/CachedUserRepository.php
class CachedUserRepository implements UserRepository
{
    public function __construct(
        private UserRepository $repository,
        private Cache $cache,
    ) {}

    public function findById(string $id): ?User
    {
        return $this->cache->remember(
            "user:{$id}",
            now()->addHours(24),
            fn() => $this->repository->findById($id)
        );
    }

    public function findByEmail(string $email): ?User
    {
        return $this->cache->remember(
            "user:email:{$email}",
            now()->addHours(24),
            fn() => $this->repository->findByEmail($email)
        );
    }

    public function save(User $user): void
    {
        $this->repository->save($user);
        // Invalidate cache
        $this->cache->forget("user:{$user->id()}");
        $this->cache->forget("user:email:{$user->email()}");
    }
}

// Register in ServiceProvider
app()->bind(UserRepository::class, function () {
    $eloquentRepo = new EloquentUserRepository();
    return new CachedUserRepository($eloquentRepo, cache());
});
```

---

## 9. Adapter Pattern

### Purpose
Make incompatible interfaces work together.

### Implementation

```php
// Domain/Payments/PaymentProcessor.php - Interface
interface PaymentProcessor
{
    public function charge(Money $amount, string $token): PaymentResult;
    public function refund(string $transactionId, Money $amount): RefundResult;
}

// Infrastructure/External/StripePaymentAdapter.php
class StripePaymentAdapter implements PaymentProcessor
{
    public function __construct(private Stripe\StripeClient $stripe) {}

    public function charge(Money $amount, string $token): PaymentResult
    {
        try {
            $charge = $this->stripe->charges->create([
                'amount' => $amount->inCents(),
                'currency' => $amount->currency(),
                'source' => $token,
            ]);

            return PaymentResult::success($charge->id);
        } catch (\Exception $e) {
            return PaymentResult::failed($e->getMessage());
        }
    }

    public function refund(string $transactionId, Money $amount): RefundResult
    {
        // Implementation
    }
}
```

---

## 10. Singleton Pattern (Use Carefully)

### Purpose
Ensure only one instance of a class exists globally.

### Implementation

```php
// Shared/Services/ConfigService.php
class ConfigService
{
    private static ?self $instance = null;
    private array $config = [];

    private function __construct() {}

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function get(string $key): mixed
    {
        return $this->config[$key] ?? null;
    }
}

// Better: Use Laravel's Service Container (not a true singleton but effective)
app()->singleton(ConfigService::class, fn() => new ConfigService());
```

---

## Pattern Selection Guide

| Problem | Pattern | Example |
|---------|---------|----------|
| Multiple data sources | Repository | Switching MySQL to PostgreSQL |
| Complex logic flow | Service Layer | Subscription creation with validation |
| Object creation | Factory | Creating different user types |
| Immutable data | Value Object | Email, Money, UUID |
| Multiple algorithms | Strategy | Different pricing strategies |
| Decouple events | Observer/Event | Sending emails on user signup |
| Add functionality | Decorator | Adding caching to repository |
| Incompatible interfaces | Adapter | Integrating third-party payment gateways |

---

## Common Mistakes

❌ **Over-engineering**: Don't use patterns if you don't need them
❌ **God Pattern**: Using one pattern for everything
❌ **Ignoring SOLID**: Patterns should follow SOLID principles
❌ **No tests**: Patterns are worthless without tests
❌ **Tight coupling**: Patterns should reduce coupling

---

Next: [03-SOLID-PRINCIPLES.md](./03-SOLID-PRINCIPLES.md)
