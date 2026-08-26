# 13 - Eager Loading & N+1 Prevention

## Preventing N+1 Query Problems

---

## N+1 Problem Explained

```php
// ❌ BAD: 101 queries (1 + 100)
$users = User::all(); // 1 query

foreach ($users as $user) {
    echo $user->team->name; // 100 queries - 1 per user
}

// ✅ GOOD: 2 queries
$users = User::with('team')->get(); // 2 queries total

foreach ($users as $user) {
    echo $user->team->name; // No additional queries
}
```

---

## Eager Loading Strategies

### Basic Eager Loading
```php
// Load relationships upfront
$users = User::with('team')->get();
$users = User::with('team', 'roles', 'profile')->get();
```

### Nested Eager Loading
```php
// Load relationships of relationships
$users = User::with('team.projects.tasks')->get();

// 3 queries:
// 1. SELECT * FROM users
// 2. SELECT * FROM teams WHERE id IN (...)
// 3. SELECT * FROM projects WHERE team_id IN (...)
// 4. SELECT * FROM tasks WHERE project_id IN (...)
```

### Conditional Eager Loading
```php
// Only load if needed
$users = User::with([
    'team' => fn($q) => $q->active(),
    'roles' => fn($q) => $q->whereIn('name', ['admin', 'owner']),
])->get();
```

### Select Only Needed Columns
```php
// ❌ Bad: Load all columns
$users = User::with('team')->get();

// ✅ Good: Select only what you need
$users = User::with([
    'team' => fn($q) => $q->select('id', 'name'),
])->select('id', 'name', 'email', 'team_id')->get();
```

---

## Query Builder Methods

### whereHas (Filter by Relationship)
```php
// Get users who have teams
$users = User::whereHas('team')->get();

// Get users with active teams
$users = User::whereHas('team', fn($q) => 
    $q->where('status', 'active')
)->get();

// Get users with at least 1 project
$users = User::whereHas('projects', fn($q) => 
    $q->where('status', '!=', 'archived')
)->get();
```

### doesntHave
```php
// Get users without teams
$orphanUsers = User::doesntHave('team')->get();
```

### hasMorph / doesntHaveMorph
```php
// Get posts that have comments
$posts = Post::hasMorph('comments', Comment::class)->get();
```

### withCount
```php
// Get user with project count
$users = User::withCount('projects')->get();

echo $users[0]->projects_count; // 5

// Multiple counts
$users = User::withCount(['projects', 'tasks', 'comments'])->get();

// Conditional count
$users = User::withCount([
    'projects' => fn($q) => $q->active(),
])->get();
```

### withSum, withAvg, withMax, withMin
```php
// Get project with total task hours
$projects = Project::withSum('tasks', 'hours')->get();
echo $projects[0]->tasks_sum_hours; // 120

// Average rating
$products = Product::withAvg('reviews', 'rating')->get();
echo $products[0]->reviews_avg_rating; // 4.5
```

---

## Lazy Loading Prevention

```php
// Prevent lazy loading in production
if (app()->isProduction()) {
    Model::handleLazyLoadingViolationUsing(function ($model, $key) {
        info("Attempted to lazy load [{$key}] on model [{$model}]");
        throw new LazyLoadingException(
            "Attempted lazy load of [{$key}] on [{$model}]"
        );
    });
}
```

---

## Relationship Performance

### One-to-Many
```php
// ❌ Bad: N+1
$teams = Team::all();
foreach ($teams as $team) {
    $users = $team->users; // Query per team
}

// ✅ Good
$teams = Team::with('users')->get();
foreach ($teams as $team) {
    $users = $team->users; // Already loaded
}
```

### Many-to-Many
```php
// ❌ Bad: N+1
$users = User::all();
foreach ($users as $user) {
    $roles = $user->roles; // Query per user
}

// ✅ Good
$users = User::with('roles')->get();
foreach ($users as $user) {
    $roles = $user->roles; // Already loaded
}
```

### Polymorphic
```php
// ❌ Bad: Multiple queries
$comments = Comment::all();
foreach ($comments as $comment) {
    $commentable = $comment->commentable; // Query per comment
}

// ✅ Good
$comments = Comment::with('commentable')->get();
foreach ($comments as $comment) {
    $commentable = $comment->commentable; // Already loaded
}
```

---

## Chunking & Lazy Collections

### For Large Datasets
```php
// ❌ Bad: Load all in memory
$users = User::all(); // 1M users = memory spike
foreach ($users as $user) {
    // Process
}

// ✅ Good: Process in batches
User::chunkById(500, function ($users) {
    foreach ($users as $user) {
        // Process
    }
});

// ✅ With relationships
User::with('team')->chunkById(500, function ($users) {
    foreach ($users as $user) {
        echo $user->team->name;
    }
});

// ✅ Lazy collection
User::lazy(500)->each(function ($user) {
    // Memory-efficient
});
```

---

## Repository Pattern with Eager Loading

```php
interface UserRepository
{
    public function all(): Collection;
    public function withTeam(): Collection;
    public function withRelations(): Collection;
}

class EloquentUserRepository implements UserRepository
{
    public function all(): Collection
    {
        return User::get();
    }

    public function withTeam(): Collection
    {
        return User::with('team')->get();
    }

    public function withRelations(): Collection
    {
        return User::with([
            'team' => fn($q) => $q->select('id', 'name'),
            'roles' => fn($q) => $q->select('id', 'name'),
            'profile' => fn($q) => $q->select('id', 'bio'),
        ])->select('id', 'name', 'email', 'team_id')->get();
    }
}

// Usage
class UserService
{
    public function __construct(private UserRepository $repository) {}

    public function getAll(): Collection
    {
        return $this->repository->withRelations();
    }
}
```

---

## Query Debugging

```php
// Log all queries
if (app()->isLocal()) {
    DB::listen(function ($query) {
        Log::debug('Query', [
            'sql' => $query->sql,
            'bindings' => $query->bindings,
            'time' => $query->time,
        ]);
    });
}

// Use Debugbar
composer require barryvdh/laravel-debugbar

// Check queries in tests
class UserControllerTest extends TestCase
{
    public function test_no_n_plus_one()
    {
        User::factory(10)->create();

        DB::enableQueryLog();
        User::with('team')->get();
        $queries = count(DB::getQueryLog());

        $this->assertEquals(2, $queries); // Should be 2, not 11
    }
}
```

---

## Eager Loading Checklist

- [ ] No lazy loading in production
- [ ] Relationships loaded with with()
- [ ] Nested relationships loaded
- [ ] Only needed columns selected
- [ ] whereHas used for filtering
- [ ] Large datasets chunked or lazy-loaded
- [ ] Query count verified in tests
- [ ] Query logging in development
- [ ] Repository methods document eager loading
- [ ] No queries in loops

---

Next: [14-PAGINATION-AND-CHUNKING.md](./14-PAGINATION-AND-CHUNKING.md)
