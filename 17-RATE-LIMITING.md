# 17 - Rate Limiting & DDoS Protection

## Protecting APIs from Abuse

---

## Built-in Rate Limiting

```php
// routes/api.php
Route::middleware('throttle:60,1')->group(function () {
    Route::get('/users', [UserController::class, 'index']);
    Route::post('/login', [AuthController::class, 'login']);
});

// Per-user rate limiting
Route::middleware('throttle:60,1')->group(function () {
    Route::apiResource('users', UserController::class);
});

// Different limits for different endpoints
Route::middleware('throttle:5,1')->group(function () {
    // Strict limit for sensitive endpoints
    Route::post('/password/reset', [PasswordController::class, 'reset']);
    Route::post('/login', [AuthController::class, 'login']);
});

Route::middleware('throttle:30,1')->group(function () {
    // Medium limit for write operations
    Route::post('/posts', [PostController::class, 'store']);
    Route::patch('/posts/{post}', [PostController::class, 'update']);
});

Route::middleware('throttle:100,1')->group(function () {
    // High limit for read operations
    Route::get('/posts', [PostController::class, 'index']);
    Route::get('/posts/{post}', [PostController::class, 'show']);
});
```

---

## Custom Rate Limiting

```php
// config/rate-limiting.php
return [
    'limits' => [
        'auth' => '5,1', // 5 requests per 1 minute
        'api' => '60,1', // 60 per minute
        'search' => '30,1', // 30 per minute
        'export' => '10,1', // 10 per minute
    ],
];

// Middleware
class RateLimitByEndpoint
{
    public function handle(Request $request, Closure $next): Response
    {
        $limit = $this->getLimitForEndpoint($request);
        return $request->middleware("throttle:{$limit}");
    }

    private function getLimitForEndpoint(Request $request): string
    {
        if ($request->is('api/auth/*')) {
            return config('rate-limiting.limits.auth');
        }

        if ($request->is('api/*/search')) {
            return config('rate-limiting.limits.search');
        }

        return config('rate-limiting.limits.api');
    }
}
```

---

## Rate Limit by User Tier

```php
class RateLimitByTier
{
    public function handle(Request $request, Closure $next): Response
    {
        $user = auth()->user();
        if (!$user) {
            return $next($request)->header('X-RateLimit-Limit', '60');
        }

        $limits = [
            'free' => '10,1',
            'pro' => '100,1',
            'enterprise' => '1000,1',
        ];

        $limit = $limits[$user->subscription->plan->tier] ?? $limits['free'];

        return $request->middleware("throttle:{$limit}")
            ->header('X-RateLimit-Limit', explode(',', $limit)[0]);
    }
}

// In kernel
protected $routeMiddleware = [
    'rate-limit-tier' => RateLimitByTier::class,
];

// Usage
Route::middleware('auth:sanctum', 'rate-limit-tier')
    ->apiResource('documents', DocumentController::class);
```

---

## Rate Limit Response Headers

```php
// When rate limited, return
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000
Retry-After: 60

[
    'error' => [
        'message' => 'Too many requests',
        'status' => 429,
        'retry_after' => 60,
    ],
]
```

---

## IP-Based Rate Limiting

```php
class RateLimitByIp
{
    public function handle(Request $request, Closure $next): Response
    {
        // Whitelist internal IPs
        if ($this->isInternalIp($request->ip())) {
            return $next($request);
        }

        // Stricter limit for unknown IPs
        return $request->middleware('throttle:10,1');
    }

    private function isInternalIp(string $ip): bool
    {
        $whitelist = explode(',', env('RATE_LIMIT_WHITELIST', ''));
        return in_array($ip, array_map('trim', $whitelist));
    }
}
```

---

## Redis-Based Rate Limiting

```php
class AdvancedRateLimiter
{
    public function __construct(private Redis $redis) {}

    public function isAllowed(string $identifier, int $limit = 60, int $window = 60): bool
    {
        $key = "rate_limit:{$identifier}";
        $current = $this->redis->incr($key);

        if ($current === 1) {
            $this->redis->expire($key, $window);
        }

        return $current <= $limit;
    }

    public function remaining(string $identifier, int $limit = 60): int
    {
        $key = "rate_limit:{$identifier}";
        $current = (int) $this->redis->get($key);
        return max(0, $limit - $current);
    }

    public function reset(string $identifier): void
    {
        $this->redis->del("rate_limit:{$identifier}");
    }
}

// Usage
class ExportController
{
    public function __construct(private AdvancedRateLimiter $limiter) {}

    public function export()
    {
        $identifier = auth()->id() . ':exports';

        if (!$this->limiter->isAllowed($identifier, 10, 3600)) {
            return response()->json(
                ['error' => 'Export limit exceeded'],
                429
            );
        }

        // Process export
    }
}
```

---

## DDoS Protection

```php
// Middleware for DDoS protection
class DdosProtection
{
    public function handle(Request $request, Closure $next): Response
    {
        $ip = $request->ip();
        $key = "ddos:{$ip}";
        $limit = 100; // 100 requests
        $window = 60; // per minute

        $current = cache()->increment($key, 1, $window);

        // After threshold, increase window
        if ($current > $limit * 2) {
            cache()->put($key, $current, 3600); // Block for 1 hour
            return response('Too many requests', 429);
        }

        if ($current > $limit) {
            return response('Too many requests', 429);
        }

        return $next($request);
    }
}
```

---

## Testing Rate Limiting

```php
class RateLimitingTest extends TestCase
{
    public function test_rate_limit_exceeded()
    {
        for ($i = 0; $i < 61; $i++) {
            $response = $this->getJson('/api/users');
        }

        $this->assertEquals(429, $response->status());
    }

    public function test_rate_limit_resets_after_window()
    {
        $response1 = $this->getJson('/api/users');
        $this->assertEquals(200, $response1->status());

        // Wait for rate limit window
        sleep(61);

        $response2 = $this->getJson('/api/users');
        $this->assertEquals(200, $response2->status());
    }
}
```

---

Next: [18-VALIDATION.md](./18-VALIDATION.md)
