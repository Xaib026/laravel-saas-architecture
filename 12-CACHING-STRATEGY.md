# 12 - Caching Strategy

## Multi-Layer Caching for Performance

---

## Cache Drivers

```php
// config/cache.php
return [
    'default' => env('CACHE_DRIVER', 'redis'),

    'stores' => [
        // Array cache (per-request, testing)
        'array' => [
            'driver' => 'array',
            'serialize' => false,
        ],

        // Redis (recommended for production)
        'redis' => [
            'driver' => 'redis',
            'connection' => 'cache',
            'options' => [
                'cluster' => env('REDIS_CLUSTER'),
                'prefix' => env('CACHE_PREFIX', 'laravel_cache_'),
            ],
        ],

        // Memcached
        'memcached' => [
            'driver' => 'memcached',
            'servers' => [
                [
                    'host' => env('MEMCACHED_HOST', '127.0.0.1'),
                    'port' => env('MEMCACHED_PORT', 11211),
                    'weight' => 100,
                ],
            ],
        ],

        // Database cache
        'database' => [
            'driver' => 'database',
            'connection' => null,
            'table' => 'cache',
        ],
    ],
];
```

---

## Query Result Caching

```php
// Cache query results
class UserRepository implements UserRepositoryInterface
{
    public function __construct(private Cache $cache) {}

    public function findById(string $id): ?User
    {
        // Cache for 24 hours
        return $this->cache->remember(
            "user:{$id}",
            now()->addHours(24),
            fn() => User::find($id)
        );
    }

    public function findByEmail(string $email): ?User
    {
        return $this->cache->remember(
            "user:email:{$email}",
            now()->addHours(24),
            fn() => User::where('email', $email)->first()
        );
    }

    public function getActive(): Collection
    {
        return $this->cache->remember(
            'users:active',
            now()->addHours(1),
            fn() => User::active()->get()
        );
    }

    public function save(User $user): void
    {
        // Invalidate caches
        $this->cache->forget("user:{$user->id}");
        $this->cache->forget("user:email:{$user->email}");
        $this->cache->forget('users:active');
    }
}
```

---

## Cache Decorator Pattern

```php
// Wrap repository with caching
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

    public function save(User $user): void
    {
        $this->repository->save($user);
        $this->cache->forget("user:{$user->id}");
    }
}

// Register in provider
app()->bind(UserRepository::class, function () {
    $eloquent = new EloquentUserRepository();
    return new CachedUserRepository($eloquent, cache());
});
```

---

## Cache Tags

```php
// Tag related cache entries
class ProjectService
{
    public function getProjects(Team $team): Collection
    {
        return cache()
            ->tags(["team:{$team->id}", 'projects'])
            ->remember(
                "team:{$team->id}:projects",
                now()->addHours(1),
                fn() => $team->projects()->get()
            );
    }

    public function updateProject(Project $project): void
    {
        $project->save();
        // Invalidate all team's project caches
        cache()->tags(["team:{$project->team_id}"])->flush();
    }

    public function deleteProject(Project $project): void
    {
        $project->delete();
        cache()->tags(["team:{$project->team_id}", 'projects'])->flush();
    }
}
```

---

## View Caching

```php
// Cache entire view output
Route::get('/dashboard', function () {
    $data = cache()->remember(
        "dashboard:" . auth()->id(),
        now()->addHours(1),
        fn() => [
            'projects' => Project::where('user_id', auth()->id())->get(),
            'stats' => UserStats::for(auth()->user()),
        ]
    );

    return view('dashboard', $data);
});

// Cache in view
@cache('products-list', now()->addHours(24))
    @foreach($products as $product)
        <div>{{ $product->name }}</div>
    @endforeach
@endcache
```

---

## Cache Invalidation Strategies

### Time-Based (TTL)
```php
// Expires automatically
cache()->remember('key', now()->addHours(1), fn() => $data);
```

### Manual Invalidation
```php
// On update
public function update(User $user, UpdateUserRequest $request): void
{
    $user->update($request->validated());
    cache()->forget("user:{$user->id}");
}

// Batch invalidation
cache()->tags(['users'])->flush();
```

### Event-Based
```php
// Invalidate on domain event
class UserUpdated
{
    public function __construct(public User $user) {}
}

class InvalidateUserCacheListener
{
    public function handle(UserUpdated $event): void
    {
        cache()->forget("user:{$event->user->id}");
    }
}
```

---

## Cache Warming

```php
class WarmCacheCommand extends Command
{
    public function handle(): void
    {
        // Pre-populate frequently accessed data
        $plans = Plan::all();
        cache()->remember('plans:all', now()->addDays(7), fn() => $plans);

        // Popular products
        $products = Product::popular()->get();
        cache()->remember('products:popular', now()->addHours(1), fn() => $products);

        $this->info('Cache warmed successfully');
    }
}

// Schedule
protected function schedule(Schedule $schedule): void
{
    $schedule->command('cache:warm')
        ->everyMinute()
        ->runInBackground();
}
```

---

## Database Query Caching

```php
class CachingQueryBuilder
{
    public function __construct(
        private Builder $query,
        private Cache $cache,
    ) {}

    public function remember(string $key, int $minutes = 60): Collection
    {
        return $this->cache->remember(
            $key,
            now()->addMinutes($minutes),
            fn() => $this->query->get()
        );
    }
}

// Usage
$users = new CachingQueryBuilder(
    User::where('status', 'active'),
    cache()
)->remember('active_users', 60);
```

---

## HTTP Response Caching

```php
// Cache HTTP responses
class CacheResponse
{
    public function handle(Request $request, Closure $next)
    {
        // Don't cache mutations
        if ($request->isMethod('GET') || $request->isMethod('HEAD')) {
            $cacheKey = "http:" . $request->getPathInfo();
            
            if (cache()->has($cacheKey)) {
                return cache()->get($cacheKey);
            }

            $response = $next($request);
            cache()->put($cacheKey, $response, now()->addHours(1));
            return $response;
        }

        return $next($request);
    }
}

// Or use Laravel built-in
Route::middleware('throttle:60,1')->group(fn() => 
    Route::get('/api/users', [UserController::class, 'index'])
        ->middleware('cache.response:3600')
);
```

---

## Testing with Cache

```php
class UserRepositoryTest extends TestCase
{
    public function test_caches_user_by_id()
    {
        Cache::spy();

        $user = User::factory()->create();
        $repo = new CachedUserRepository($this->repo, cache());

        $result = $repo->findById($user->id);

        Cache::shouldHaveReceived('remember')
            ->with("user:{$user->id}");
    }

    public function test_invalidates_cache_on_update()
    {
        Cache::spy();

        $user = User::factory()->create();
        $repo = new CachedUserRepository($this->repo, cache());

        $repo->save($user);

        Cache::shouldHaveReceived('forget')
            ->with("user:{$user->id}");
    }
}
```

---

## Caching Checklist

- [ ] Redis configured for production
- [ ] Query results cached where needed
- [ ] Cache keys are meaningful
- [ ] Cache TTLs are appropriate
- [ ] Invalidation strategy defined
- [ ] Cache hits monitored
- [ ] Decorator pattern for optional caching
- [ ] Cache warming for popular data
- [ ] Tests mock cache
- [ ] No caching of sensitive data

---

Next: [13-EAGER-LOADING.md](./13-EAGER-LOADING.md)
