# 14 - Pagination & Chunking Large Datasets

## Handling Large Data Sets Efficiently

---

## Pagination Methods

### Offset Pagination
```php
// Traditional pagination (good for small datasets)
$users = User::paginate(15);

// Response
[
    'data' => [...],
    'meta' => [
        'current_page' => 1,
        'from' => 1,
        'to' => 15,
        'total' => 1000,
        'per_page' => 15,
        'last_page' => 67,
    ],
    'links' => [
        'first' => 'https://api.example.com/v1/users?page=1',
        'last' => 'https://api.example.com/v1/users?page=67',
        'next' => 'https://api.example.com/v1/users?page=2',
        'prev' => null,
    ],
]

// Custom per-page
$users = User::paginate($request->input('per_page', 15));

// Limits
$perPage = min($request->input('per_page', 15), 100); // Max 100
$users = User::paginate($perPage);
```

### Cursor Pagination (RECOMMENDED)
```php
// Better for large datasets
// No need to count total records
$users = User::orderBy('id')->cursorPaginate(50);

// Response
[
    'data' => [...],
    'meta' => [
        'path' => 'https://api.example.com/v1/users',
        'per_page' => 50,
        'next_cursor' => 'eyJpZCI6IDUwLCAi...',
        'prev_cursor' => null,
    ],
]

// Usage
// GET /api/v1/users?cursor=eyJpZCI6IDUwLCAi...
```

### Simple Pagination
```php
// When you don't need total count
$users = User::simplePaginate(15);

// Only has next/prev links
[
    'data' => [...],
    'links' => [
        'first' => '...',
        'last' => '...',
        'next' => '...',
        'prev' => null,
    ],
]
```

---

## Pagination Best Practices

```php
// Always order before paginating
$users = User::orderBy('created_at', 'desc')
    ->cursorPaginate(50);

// Eager load relationships
$users = User::with('team', 'roles')
    ->orderBy('id')
    ->cursorPaginate(50);

// Set max per_page to prevent abuse
$perPage = min($request->input('per_page', 15), 100);
$users = User::paginate($perPage);

// In API controller
class UserController
{
    public function index(Request $request)
    {
        $perPage = min($request->input('per_page', 15), 100);
        
        $users = User::with('team')
            ->orderBy('created_at', 'desc')
            ->paginate($perPage);

        return UserResource::collection($users);
    }
}
```

---

## Chunking for Processing

### chunkById (Recommended)
```php
// Process large datasets without loading all into memory
User::chunkById(500, function ($users) {
    foreach ($users as $user) {
        // Process user
        ProcessUserJob::dispatch($user);
    }
});

// With ordering
User::orderBy('id')
    ->chunkById(500, function ($users) {
        foreach ($users as $user) {
            // Process
        }
    });

// With relationships
User::with('team')
    ->chunkById(500, function ($users) {
        foreach ($users as $user) {
            echo $user->team->name;
        }
    });

// With filtering
User::where('status', 'active')
    ->chunkById(500, function ($users) {
        foreach ($users as $user) {
            // Process active users
        }
    });
```

### chunk (Offset-Based)
```php
// When orderBy id not suitable
Product::chunk(500, function ($products) {
    foreach ($products as $product) {
        // Process
    }
});

// Issues: Slower for large offsets
// Use chunkById instead when possible
```

### Lazy Collections
```php
// Memory-efficient processing
User::lazy()->each(function ($user) {
    // Process one at a time
    ProcessUserJob::dispatch($user);
});

// With chunk size
User::lazy(500)->each(function ($user) {
    // Process
});

// Chain operations
User::lazy(500)
    ->filter(fn($user) => $user->is_active)
    ->map(fn($user) => $user->email)
    ->each(fn($email) => sendEmail($email));
```

---

## Performance Considerations

### When to Use What

| Scenario | Method | Why |
|----------|--------|-----|
| Small dataset (< 1K) | paginate() | Simple, shows total count |
| Large dataset | cursorPaginate() | Efficient, no COUNT query |
| Processing all records | chunkById() | Memory-efficient |
| Filtering/mapping all | lazy() | Composable, lazy evaluation |
| Real-time updates | cursorPaginate() | Stable cursor position |

### Query Optimization

```php
// ❌ Bad: Counts all rows
$users = User::paginate(15); // COUNT(*) query

// ✅ Good: No counting
$users = User::cursorPaginate(15);

// ❌ Bad: Loads all for chunk
$users = User::all();
foreach ($users as $user) { /* process */ }

// ✅ Good: Streams in batches
User::chunkById(500, function ($users) {
    foreach ($users as $user) { /* process */ }
});
```

---

## Database Indexing for Pagination

```php
// Index the orderBy column
Schema::table('users', function (Blueprint $table) {
    $table->index('created_at'); // For ORDER BY
    $table->index('id'); // For cursor pagination
});

// Composite index for common queries
Schema::table('users', function (Blueprint $table) {
    $table->index(['team_id', 'created_at']); // For filtering + ordering
});
```

---

## API Pagination Response

```php
// Custom pagination response
class UserCollection extends ResourceCollection
{
    public function toArray($request): array
    {
        return [
            'data' => $this->collection,
            'pagination' => [
                'current_page' => $this->resource->currentPage(),
                'per_page' => $this->resource->perPage(),
                'total' => $this->resource->total(),
                'total_pages' => $this->resource->lastPage(),
                'from' => $this->resource->firstItem(),
                'to' => $this->resource->lastItem(),
            ],
            'links' => [
                'first' => $this->resource->url(1),
                'last' => $this->resource->url($this->resource->lastPage()),
                'prev' => $this->resource->previousPageUrl(),
                'next' => $this->resource->nextPageUrl(),
            ],
        ];
    }
}
```

---

## Testing Pagination

```php
class UserControllerTest extends TestCase
{
    public function test_pagination_returns_correct_page()
    {
        User::factory(50)->create();

        $response = $this->getJson('/api/v1/users?page=1&per_page=10');

        $response->assertOk()
            ->assertJsonCount(10, 'data')
            ->assertJsonPath('pagination.total', 50)
            ->assertJsonPath('pagination.current_page', 1);
    }

    public function test_pagination_limits_per_page()
    {
        User::factory(50)->create();

        $response = $this->getJson('/api/v1/users?per_page=200');

        // Should not exceed max
        $this->assertLessThanOrEqual(
            100,
            count($response->json('data'))
        );
    }

    public function test_cursor_pagination_works()
    {
        User::factory(100)->create();

        $response = $this->getJson('/api/v1/users?limit=10');

        $this->assertNotNull($response->json('meta.next_cursor'));
    }
}
```

---

## Pagination Checklist

- [ ] Use cursorPaginate() for large datasets
- [ ] Set max per_page limit
- [ ] OrderBy before pagination
- [ ] Eager load relationships
- [ ] Index orderBy columns
- [ ] chunkById() for processing all records
- [ ] lazy() for transforming large collections
- [ ] API response includes pagination metadata
- [ ] Tests cover pagination scenarios
- [ ] Frontend implements pagination correctly

---

Next: [15-ERROR-HANDLING.md](./15-ERROR-HANDLING.md)
