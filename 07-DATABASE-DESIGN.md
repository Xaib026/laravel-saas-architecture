# 07 - Database Design & Optimization

## Building Performant, Scalable Database Layer

---

## Migration Best Practices

```php
// Good migration naming
// 2024_01_01_000000_create_users_table.php
// 2024_01_01_000001_create_teams_table.php
// 2024_01_02_000000_add_stripe_id_to_subscriptions.php

class CreateUsersTable extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            // Primary key
            $table->id(); // auto-incrementing bigint
            // or
            $table->uuid('id')->primary(); // for UUIDs

            // Basic fields
            $table->string('name', 255);
            $table->string('email', 255)->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password', 255);

            // Foreign keys
            $table->foreignId('team_id')
                ->constrained('teams')
                ->cascadeOnDelete();

            // Optional fields
            $table->string('phone', 20)->nullable();
            $table->string('avatar_url')->nullable();

            // Status
            $table->enum('status', ['active', 'inactive', 'banned'])->default('active');

            // Timestamps
            $table->timestamps();
            $table->softDeletes(); // For audit trail

            // Indexes (CRITICAL for performance)
            $table->index('email');
            $table->index(['team_id', 'status']);
            $table->index('created_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
    }
}
```

---

## Indexing Strategy

### Single Column Indexes
```php
// For columns frequently used in WHERE clauses
$table->index('email');
$table->index('status');
$table->index('created_at');
```

### Composite Indexes
```php
// For frequent multi-column queries
// Query: WHERE team_id = ? AND status = ? ORDER BY created_at DESC
$table->index(['team_id', 'status', 'created_at']);

// Query: WHERE user_id = ? AND payment_status = ?
$table->index(['user_id', 'payment_status']);
```

### Unique Indexes
```php
// Enforce uniqueness + fast lookups
$table->unique('email');
$table->unique(['team_id', 'slug']); // Unique per team
```

### Full-Text Indexes
```php
// For search functionality
$table->fullText(['title', 'description']);

// Query
Post::whereFullText(['title', 'description'], 'laravel')->get();
```

### Indexing Checklist
```php
// ✅ Index foreign keys
$table->foreignId('user_id')->constrained();

// ✅ Index columns in WHERE clauses
$table->index('status');

// ✅ Index columns in ORDER BY
$table->index('created_at');

// ✅ Composite indexes for common queries
$table->index(['team_id', 'status', 'created_at']);

// ✅ Unique constraints
$table->unique('email');

// ❌ Don't index everything (slows inserts)
// ❌ Don't create duplicate indexes
// ❌ Don't use indexes on low-cardinality columns
```

---

## Common Migrations

```php
// Add column
class AddStripeIdToSubscriptionsTable extends Migration
{
    public function up(): void
    {
        Schema::table('subscriptions', function (Blueprint $table) {
            $table->string('stripe_id')->nullable()->after('id');
            $table->index('stripe_id');
        });
    }

    public function down(): void
    {
        Schema::table('subscriptions', function (Blueprint $table) {
            $table->dropColumn('stripe_id');
        });
    }
}

// Rename column
class RenameStatusToPaymentStatusInPaymentsTable extends Migration
{
    public function up(): void
    {
        Schema::table('payments', function (Blueprint $table) {
            $table->renameColumn('status', 'payment_status');
        });
    }
}

// Change column type
class ChangeAmountTypeInPaymentsTable extends Migration
{
    public function up(): void
    {
        Schema::table('payments', function (Blueprint $table) {
            $table->decimal('amount', 10, 2)->change();
        });
    }
}

// Add index
class AddIndexToUsersTable extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->index(['team_id', 'created_at']);
        });
    }
}
```

---

## Relationships & Keys

```php
// One-to-Many
class Team extends Model
{
    public function projects()
    {
        return $this->hasMany(Project::class);
    }
}

class Project extends Model
{
    public function team()
    {
        return $this->belongsTo(Team::class);
    }
}

// Many-to-Many
class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }
}

class Role extends Model
{
    public function users()
    {
        return $this->belongsToMany(User::class);
    }
}

// One-to-One
class User extends Model
{
    public function profile()
    {
        return $this->hasOne(UserProfile::class);
    }
}

// Polymorphic
class Comment extends Model
{
    public function commentable()
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

---

## Query Optimization

### Eager Loading (Prevent N+1)

```php
// ❌ Bad - 101 queries
foreach (User::all() as $user) {
    echo $user->team->name; // Query for each user
}

// ✅ Good - 2 queries
foreach (User::with('team')->get() as $user) {
    echo $user->team->name;
}

// ✅ Nested eager loading
User::with('team.projects.tasks')->get();

// ✅ Conditional eager loading
User::with(['team' => fn($q) => $q->active()])->get();
```

### Select Only Needed Columns

```php
// ❌ Bad - selects all columns
User::get();

// ✅ Good - selects only needed
User::select('id', 'name', 'email')->get();

// ✅ In relationships
User::with(['team' => fn($q) => $q->select('id', 'name')])->get();
```

### Chunking Large Datasets

```php
// ❌ Bad - loads all in memory
$users = User::all();
foreach ($users as $user) {
    // Process
}

// ✅ Good - processes in batches
User::chunk(500, function ($users) {
    foreach ($users as $user) {
        // Process
    }
});

// ✅ With ordering
User::orderBy('id')->chunkById(500, function ($users) {
    foreach ($users as $user) {
        // Process
    }
});
```

### Lazy Collections

```php
// ✅ Memory-efficient iteration
User::lazy()->each(function ($user) {
    // Process one at a time
});

// ✅ With chunking
User::lazy(500)->each(function ($user) {
    // Process
});
```

---

## Scopes (Reusable Queries)

```php
class User extends Model
{
    // Local scope
    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }

    // Scope with parameters
    public function scopeForTeam($query, Team $team)
    {
        return $query->where('team_id', $team->id);
    }

    // Scope with multiple conditions
    public function scopeRecent($query, $days = 7)
    {
        return $query->where('created_at', '>', now()->subDays($days));
    }
}

// Usage
User::active()->get();
User::forTeam($team)->get();
User::active()->recent(30)->get();

// In relationships
$team->projects()->active()->get();
```

---

## Casts & Accessors

```php
class User extends Model
{
    // Automatic type casting
    protected $casts = [
        'email_verified_at' => 'datetime',
        'is_active' => 'boolean',
        'settings' => 'json',
        'created_at' => 'datetime:Y-m-d',
    ];

    // Accessor (computed property)
    public function getFullNameAttribute(): string
    {
        return "{$this->first_name} {$this->last_name}";
    }

    // Mutator (set transformation)
    public function setPasswordAttribute($value): void
    {
        $this->attributes['password'] = bcrypt($value);
    }

    // Computed property (PHP 8.1+)
    #[Attribute]
    public function displayName(): string
    {
        return ucfirst($this->name);
    }
}
```

---

## Database Performance Monitoring

```php
// Development: Log slow queries
if (app()->isLocal()) {
    DB::listen(function ($query) {
        if ($query->time > 100) { // 100ms threshold
            Log::warning('Slow Query', [
                'sql' => $query->sql,
                'bindings' => $query->bindings,
                'time' => $query->time . 'ms',
                'stack' => debug_backtrace(),
            ]);
        }
    });
}

// Production: Monitoring
// Use tools like New Relic, DataDog, or Sentry
```

---

## Data Retention & Cleanup

```php
// Delete old soft-deleted records
class PruneDeletedUsers extends Command
{
    public function handle()
    {
        User::onlyTrashed()
            ->where('deleted_at', '<', now()->subMonths(12))
            ->forceDelete();
    }
}

// Schedule in kernel
protected function schedule(Schedule $schedule)
{
    $schedule->command('command:prune-deleted-users')
        ->dailyAt('02:00');
}
```

---

## Database Checklist

- [ ] All foreign keys are indexed
- [ ] Composite indexes for multi-column queries
- [ ] Unique constraints where needed
- [ ] Soft deletes for audit trail
- [ ] Team_id on all tenant tables
- [ ] Created_at and updated_at timestamps
- [ ] Proper column types (string length, decimal precision)
- [ ] Eager loading prevents N+1
- [ ] Scopes for common queries
- [ ] Tests verify query performance
- [ ] Indexes are tested (EXPLAIN ANALYZE)
- [ ] Data cleanup scheduled

---

Next: [08-API-DESIGN.md](./08-API-DESIGN.md)
