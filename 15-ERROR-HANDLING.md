# 15 - Error Handling & Exceptions

## Comprehensive Exception Management

---

## Custom Exceptions

```php
// Domain/Shared/Exceptions/DomainException.php
abstract class DomainException extends Exception {}

// Domain/Users/Exceptions/UserAlreadyExistsException.php
class UserAlreadyExistsException extends DomainException
{
    public function __construct()
    {
        parent::__construct('User with this email already exists', 409);
    }
}

class UserNotFoundException extends DomainException
{
    public function __construct(string $userId)
    {
        parent::__construct("User {$userId} not found", 404);
    }
}

class InvalidTeamException extends DomainException
{
    public function __construct()
    {
        parent::__construct('Team is invalid or inactive', 400);
    }
}

// Domain/Subscriptions/Exceptions
class SubscriptionAlreadyActiveException extends DomainException
{
    public function __construct()
    {
        parent::__construct('Subscription is already active', 400);
    }
}

class PaymentFailedException extends DomainException
{
    public function __construct(string $reason = 'Payment processing failed')
    {
        parent::__construct($reason, 402);
    }
}
```

---

## Exception Handler

```php
// Infrastructure/Providers/ExceptionHandler.php
class Handler extends ExceptionHandler
{
    protected $dontReport = [
        \Illuminate\Auth\AuthenticationException::class,
        \Illuminate\Auth\Access\AuthorizationException::class,
    ];

    public function render($request, Throwable $exception)
    {
        // JSON API responses
        if ($request->wantsJson() || $request->is('api/*')) {
            return $this->jsonResponse($exception);
        }

        // HTML responses
        return parent::render($request, $exception);
    }

    private function jsonResponse(Throwable $exception): Response
    {
        $status = $this->getHttpStatus($exception);
        $message = $this->getMessage($exception);

        return response()->json([
            'error' => [
                'message' => $message,
                'status' => $status,
                'errors' => $this->getValidationErrors($exception),
            ],
        ], $status);
    }

    private function getHttpStatus(Throwable $exception): int
    {
        if ($exception instanceof ValidationException) {
            return 422;
        }

        if ($exception instanceof AuthorizationException) {
            return 403;
        }

        if ($exception instanceof ModelNotFoundException) {
            return 404;
        }

        if ($exception instanceof ThrottleRequestsException) {
            return 429;
        }

        if (method_exists($exception, 'getStatusCode')) {
            return $exception->getStatusCode();
        }

        return 500;
    }

    private function getMessage(Throwable $exception): string
    {
        if (app()->isProduction()) {
            // Don't expose internal details in production
            if ($exception->getCode() >= 400 && $exception->getCode() < 500) {
                return $exception->getMessage();
            }
            return 'An error occurred. Please try again later.';
        }

        return $exception->getMessage();
    }

    private function getValidationErrors(Throwable $exception): array
    {
        if ($exception instanceof ValidationException) {
            return $exception->validator->errors()->toArray();
        }

        return [];
    }
}
```

---

## Try-Catch Best Practices

```php
// Good: Catch specific exceptions
try {
    $user = User::findOrFail($id);
} catch (ModelNotFoundException $e) {
    return response()->json(['error' => 'User not found'], 404);
}

// Better: Throw custom exception
try {
    $subscription = Subscription::where('id', $id)->firstOrFail();
} catch (ModelNotFoundException $e) {
    throw new SubscriptionNotFoundException($id);
}

// Best: Use validation/authorization
class SubscriptionPolicy
{
    public function view(User $user, Subscription $subscription): bool
    {
        return $subscription->user_id === $user->id;
    }
}

// In controller
public function show(Subscription $subscription)
{
    $this->authorize('view', $subscription); // Throws AuthorizationException
    return SubscriptionResource::make($subscription);
}
```

---

## Logging Exceptions

```php
// Log with context
Log::error('Payment processing failed', [
    'user_id' => auth()->id(),
    'subscription_id' => $subscription->id,
    'amount' => $subscription->amount,
    'error' => $exception->getMessage(),
    'trace' => $exception->getTraceAsString(),
]);

// In exception handler
public function report(Throwable $exception): void
{
    if ($exception instanceof PaymentFailedException) {
        Log::critical('Payment failed - requires investigation', [
            'exception' => $exception,
        ]);
        // Alert admins
        Notification::route('mail', config('admin.email'))
            ->notify(new PaymentFailedNotification($exception));
    }

    parent::report($exception);
}
```

---

## User-Friendly Error Messages

```php
// resources/lang/en/errors.php
return [
    'user_not_found' => 'User not found',
    'user_already_exists' => 'A user with this email already exists',
    'payment_failed' => 'Payment processing failed. Please try again.',
    'invalid_team' => 'You do not have access to this team',
    'subscription_inactive' => 'Your subscription is inactive',
    'quota_exceeded' => 'You have reached your usage limit',
];

// Exception
class UserNotFoundException extends DomainException
{
    public function __construct()
    {
        parent::__construct(__('errors.user_not_found'), 404);
    }
}
```

---

## Validation Error Responses

```php
// When validation fails
[
    'error' => [
        'message' => 'The given data was invalid',
        'status' => 422,
        'errors' => [
            'email' => [
                'The email field is required.',
                'The email must be a valid email address.',
            ],
            'password' => [
                'The password must be at least 8 characters.',
            ],
        ],
    ],
]
```

---

## Error Monitoring (Sentry)

```php
// config/sentry.php
'dsn' => env('SENTRY_LARAVEL_DSN'),
'traces_sample_rate' => 0.1,
'profiles_sample_rate' => 0.1,

// In exception handler
public function report(Throwable $exception): void
{
    if (app()->bound('sentry')) {
        app('sentry')->captureException($exception);
    }
    parent::report($exception);
}

// Manual reporting
if (app()->bound('sentry')) {
    app('sentry')->captureMessage('Payment processing started', 'info');
}
```

---

## Testing Exceptions

```php
class UserServiceTest extends TestCase
{
    public function test_throws_if_user_exists()
    {
        User::factory()->create(['email' => 'john@example.com']);

        $this->expectException(UserAlreadyExistsException::class);
        $this->expectExceptionCode(409);

        $service = new UserService();
        $service->create(new CreateUserDTO(
            email: 'john@example.com',
            name: 'John',
            password: 'password',
        ));
    }

    public function test_error_response_format()
    {
        $response = $this->getJson('/api/v1/users/999');

        $response->assertStatus(404)
            ->assertJsonStructure([
                'error' => ['message', 'status'],
            ]);
    }
}
```

---

Next: [16-AUDIT-LOGGING.md](./16-AUDIT-LOGGING.md)
