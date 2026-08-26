# 09 - Service Layer & Use Cases

## Application Layer Orchestration

---

## Service Classes

```php
// Application/Services/UserService.php
namespace App\Application\Services;

use App\Application\DTO\CreateUserDTO;
use App\Application\DTO\UpdateUserDTO;
use App\Domain\Users\User;
use App\Domain\Users\UserRepository;
use App\Infrastructure\Mail\WelcomeEmail;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;

class UserService
{
    public function __construct(
        private UserRepository $userRepository,
        private EventDispatcher $eventDispatcher,
    ) {}

    public function createUser(CreateUserDTO $dto): User
    {
        // Validate
        if ($this->userRepository->findByEmail($dto->email)) {
            throw new UserAlreadyExistsException();
        }

        // Create within transaction
        return DB::transaction(function () use ($dto) {
            $user = User::create([
                'email' => $dto->email,
                'name' => $dto->name,
                'password' => bcrypt($dto->password),
                'team_id' => $dto->teamId,
            ]);

            $this->userRepository->save($user);

            // Dispatch events
            $this->eventDispatcher->dispatch(new UserCreated($user));

            return $user;
        });
    }

    public function updateUser(User $user, UpdateUserDTO $dto): User
    {
        $user->update($dto->toArray());
        $this->userRepository->save($user);
        $this->eventDispatcher->dispatch(new UserUpdated($user));
        return $user;
    }

    public function deleteUser(User $user): void
    {
        $this->userRepository->delete($user);
        $this->eventDispatcher->dispatch(new UserDeleted($user));
    }
}
```

---

## Use Cases (Application Commands)

```php
// Application/UseCases/RegisterUserUseCase.php
namespace App\Application\UseCases;

use App\Application\DTO\RegisterUserDTO;
use App\Application\Services\UserService;
use App\Domain\Users\User;
use App\Domain\Users\Events\UserRegistered;
use Illuminate\Support\Facades\Event;

class RegisterUserUseCase
{
    public function __construct(
        private UserService $userService,
    ) {}

    public function execute(RegisterUserDTO $dto): User
    {
        // Business logic orchestration
        $user = $this->userService->createUser($dto);

        // Can dispatch multiple events
        Event::dispatch(new UserRegistered($user));

        return $user;
    }
}

// More complex use case example
class CreateSubscriptionUseCase
{
    public function __construct(
        private SubscriptionService $subscriptionService,
        private PaymentService $paymentService,
        private InvoiceService $invoiceService,
    ) {}

    public function execute(CreateSubscriptionDTO $dto): Subscription
    {
        // Step 1: Validate
        $plan = Plan::findOrFail($dto->planId);
        if (!$plan->isAvailable()) {
            throw new PlanNotAvailableException();
        }

        // Step 2: Process payment
        $payment = $this->paymentService->processPayment($dto);
        if (!$payment->isSuccessful()) {
            throw new PaymentFailedException();
        }

        // Step 3: Create subscription
        $subscription = $this->subscriptionService->create($dto);

        // Step 4: Generate invoice
        $invoice = $this->invoiceService->generate($subscription);

        // Step 5: Dispatch events
        event(new SubscriptionCreated($subscription));
        event(new InvoiceGenerated($invoice));

        return $subscription;
    }
}
```

---

## Data Transfer Objects (DTOs)

```php
// Application/DTO/RegisterUserDTO.php
namespace App\Application\DTO;

use Illuminate\Http\Request;

final class RegisterUserDTO
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly string $password,
        public readonly ?string $teamName = null,
    ) {
        $this->validate();
    }

    private function validate(): void
    {
        if (empty($this->name) || strlen($this->name) > 255) {
            throw new InvalidArgumentException('Invalid name');
        }

        if (!filter_var($this->email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }

        if (strlen($this->password) < 8) {
            throw new InvalidArgumentException('Password too short');
        }
    }

    public static function fromRequest(Request $request): self
    {
        return new self(
            name: $request->input('name'),
            email: $request->input('email'),
            password: $request->input('password'),
            teamName: $request->input('team_name'),
        );
    }

    public static function fromArray(array $data): self
    {
        return new self(
            name: $data['name'],
            email: $data['email'],
            password: $data['password'],
            teamName: $data['team_name'] ?? null,
        );
    }
}
```

---

## Service Provider Registration

```php
// Infrastructure/Providers/UseCaseServiceProvider.php
namespace App\Infrastructure\Providers;

use App\Application\UseCases\RegisterUserUseCase;
use App\Application\UseCases\CreateSubscriptionUseCase;
use App\Application\Services\UserService;
use App\Application\Services\SubscriptionService;
use App\Domain\Users\UserRepository;
use Illuminate\Support\ServiceProvider;

class UseCaseServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Register services
        $this->app->bind(UserService::class, function ($app) {
            return new UserService(
                $app->make(UserRepository::class),
                $app->make('events'),
            );
        });

        $this->app->bind(SubscriptionService::class, function ($app) {
            return new SubscriptionService(
                $app->make(SubscriptionRepository::class),
                $app->make('events'),
            );
        });

        // Register use cases
        $this->app->bind(RegisterUserUseCase::class, function ($app) {
            return new RegisterUserUseCase(
                $app->make(UserService::class),
            );
        });

        $this->app->bind(CreateSubscriptionUseCase::class, function ($app) {
            return new CreateSubscriptionUseCase(
                $app->make(SubscriptionService::class),
                $app->make(PaymentService::class),
                $app->make(InvoiceService::class),
            );
        });
    }
}
```

---

## Controller Usage

```php
// Presentation/Http/Controllers/Api/V1/UserController.php
namespace App\Presentation\Http\Controllers\Api\V1;

use App\Application\DTO\RegisterUserDTO;
use App\Application\UseCases\RegisterUserUseCase;
use App\Presentation\Http\Requests\StoreUserRequest;
use App\Presentation\Http\Resources\UserResource;
use Illuminate\Http\JsonResponse;

class UserController extends Controller
{
    public function __construct(
        private RegisterUserUseCase $registerUserUseCase,
    ) {}

    public function store(StoreUserRequest $request): JsonResponse
    {
        // Convert request to DTO
        $dto = RegisterUserDTO::fromRequest($request);

        // Execute use case
        $user = $this->registerUserUseCase->execute($dto);

        // Return response
        return response()->json(
            UserResource::make($user),
            201
        );
    }
}
```

---

## Testing Services

```php
// tests/Unit/Application/UseCases/RegisterUserUseCaseTest.php
namespace Tests\Unit\Application\UseCases;

use App\Application\DTO\RegisterUserDTO;
use App\Application\UseCases\RegisterUserUseCase;
use App\Domain\Users\Events\UserRegistered;
use App\Domain\Users\UserRepository;
use Illuminate\Support\Facades\Event;
use Tests\TestCase;

class RegisterUserUseCaseTest extends TestCase
{
    private UserRepository $userRepository;
    private RegisterUserUseCase $useCase;

    protected function setUp(): void
    {
        parent::setUp();

        $this->userRepository = $this->createMock(UserRepository::class);
        $this->useCase = new RegisterUserUseCase(
            new UserService($this->userRepository, app('events')),
        );
    }

    public function test_can_register_new_user()
    {
        Event::fake();

        $dto = new RegisterUserDTO(
            name: 'John Doe',
            email: 'john@example.com',
            password: 'password123',
        );

        $user = $this->useCase->execute($dto);

        $this->assertEquals('john@example.com', $user->email);
        Event::assertDispatched(UserRegistered::class);
    }

    public function test_throws_if_email_exists()
    {
        // Setup
        $existingUser = User::factory()->create(['email' => 'john@example.com']);

        $dto = new RegisterUserDTO(
            name: 'John Doe',
            email: 'john@example.com',
            password: 'password123',
        );

        // Test
        $this->expectException(UserAlreadyExistsException::class);
        $this->useCase->execute($dto);
    }
}
```

---

## Checklist

- [ ] One service per domain concept
- [ ] One use case per command/workflow
- [ ] DTOs for data transfer
- [ ] Services use repositories (not models)
- [ ] Use cases orchestrate services
- [ ] Transactions for critical operations
- [ ] Events dispatched for side effects
- [ ] Proper error handling
- [ ] Tests cover happy and error paths
- [ ] No business logic in controllers

---

Next: [10-EVENT-DRIVEN-ARCHITECTURE.md](./10-EVENT-DRIVEN-ARCHITECTURE.md)
