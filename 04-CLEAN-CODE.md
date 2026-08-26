# 04 - Clean Code & Alexey's Best Practices

## Laravel Best Practices from Alexey Mezenin + SaaS Enhancements

This guide combines Alexey's proven Laravel best practices with enterprise SaaS patterns.

---

## 1. Single Responsibility Principle

### ❌ Bad
```php
public function update(Request $request): string
{
    $validated = $request->validate([
        'title' => 'required|max:255',
        'events' => 'required|array:date,type'
    ]);

    foreach ($request->events as $event) {
        $date = $this->carbon->parse($event['date'])->toString();
        $this->logger->log('Update event ' . $date . ' :: ' . $event['type']);
    }

    $this->event->updateGeneralEvent($request->validated());
    return back();
}
```

### ✅ Good
```php
public function update(UpdateEventRequest $request): RedirectResponse
{
    $this->logService->logEvents($request->events);
    $this->event->updateGeneralEvent($request->validated());
    return back();
}
```

---

## 2. Methods Should Do One Thing

### ❌ Bad
```php
public function getFullNameAttribute(): string
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

### ✅ Good
```php
public function getFullNameAttribute(): string
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

private function isVerifiedClient(): bool
{
    return auth()->check() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

private function getFullNameLong(): string
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

private function getFullNameShort(): string
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

---

## 3. Fat Models, Skinny Controllers

### ❌ Bad
```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

### ✅ Good
```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

// In Client Model
public function getWithNewOrders(): Collection
{
    return $this->verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();
}
```

---

## 4. Validation in Request Classes

### ❌ Bad
```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);
    // ...
}
```

### ✅ Good
```php
public function store(PostRequest $request)
{
    // ...
}

class PostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

---

## 5. Business Logic in Service Classes

### ❌ Bad
```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images/temp'));
    }
    // More logic...
}
```

### ✅ Good
```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));
    // ...
}

class ArticleService
{
    public function handleUploadedImage(?UploadedFile $image): void
    {
        if (!is_null($image)) {
            $image->move(public_path('images/temp'));
        }
    }
}
```

---

## 6. Don't Repeat Yourself (DRY)

### ❌ Bad
```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
        $q->where('verified', 1)->whereNotNull('deleted_at');
    })->get();
}
```

### ✅ Good
```php
public function scopeActive($q)
{
    return $q->where('verified', true)->whereNotNull('deleted_at');
}

public function getActive(): Collection
{
    return $this->active()->get();
}

public function getArticles(): Collection
{
    return $this->whereHas('user', function ($q) {
        $q->active();
    })->get();
}
```

---

## 7. Prefer Eloquent Over Raw SQL

### ❌ Bad (Raw SQL)
```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

### ✅ Good (Eloquent)
```php
Article::has('user.profile')
    ->verified()
    ->active()
    ->latest()
    ->get();
```

---

## 8. Use Mass Assignment Properly

### ❌ Bad
```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
$article->category_id = $category->id;
$article->save();
```

### ✅ Good
```php
$category->article()->create($request->validated());
```

---

## 9. Avoid N+1 Problem

### ❌ Bad (101 queries for 100 users)
```blade
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

### ✅ Good (2 queries)
```php
$users = User::with('profile')->get();

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

---

## 10. Chunk Data for Heavy Operations

### ❌ Bad
```php
$users = $this->get();

foreach ($users as $user) {
    // Process
}
```

### ✅ Good
```php
$this->chunk(500, function ($users) {
    foreach ($users as $user) {
        // Process
    }
});
```

---

## 11. Use Descriptive Names

### ❌ Bad
```php
// Determine if there are any joins
if (count((array) $builder->getQuery()->joins) > 0)
```

### ✅ Good
```php
if ($this->hasJoins())
```

---

## 12. Separate JavaScript and HTML from PHP

### ❌ Bad
```javascript
let article = `{{ json_encode($article) }}`;
```

### ✅ Good (HTML)
```php
<input id="article" type="hidden" value='@json($article)'>
<button class="js-favorite" data-article='@json($article)'>{{ $article->name }}</button>
```

### Good (JavaScript)
```javascript
let article = document.getElementById('article').value;
```

---

## 13. Use Config Files Instead of Hard-Coded Values

### ❌ Bad
```php
public function isNormal(): bool
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

### ✅ Good
```php
public function isNormal(): bool
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));

// In config/app.php or enum
class ArticleType
{
    public const TYPE_NORMAL = 'normal';
    public const TYPE_FEATURED = 'featured';
    public const TYPE_DRAFT = 'draft';
}

// In resources/lang/en/app.php
return [
    'article_added' => 'Your article has been added!',
];
```

---

## 14. Use Standard Laravel Tools

| Task | Standard Tool | Avoid |
|------|---------------|-------|
| Authorization | Policies | Entrust, Sentinel |
| Authentication | Sanctum/Passport | Custom solutions |
| Testing | PHPUnit, Pest | Phpspec |
| Validation | Form Requests | Validation in controllers |
| Database | Eloquent | Raw SQL, Doctrine |
| Templates | Blade | Twig |
| Collections | Laravel Collections | Arrays |
| Task Scheduling | Laravel Scheduler | Cron scripts |

---

## 15. Follow Naming Conventions

### Controllers
```php
// Singular
class UserController { }
class SubscriptionController { }

// NOT
class UsersController { }
class SubscriptionsController { }
```

### Routes
```php
// Plural
Route::get('/users/1', ...)
Route::get('/subscriptions', ...)

// NOT
Route::get('/user/1', ...)
Route::get('/subscription', ...)
```

### Models
```php
class User { }
class Subscription { }

// NOT
class Users { }
class Subscriptions { }
```

### Relationships
```php
public function comments() { } // hasMany - plural
public function author() { }   // belongsTo - singular
public function profile() { }  // hasOne - singular
```

### Database
```php
// Tables - plural
users, subscriptions, posts, comments

// Foreign keys - singular model + _id
user_id, subscription_id, post_id

// Pivot tables - alphabetical, singular
article_user (not user_article or articles_users)
```

### Files and Classes
```php
// Requests
StoreUserRequest, UpdateUserRequest

// Resources
UserResource, UserCollection

// Policies
UserPolicy, SubscriptionPolicy

// Events
UserRegistered, SubscriptionCancelled

// Jobs
SendWelcomeEmailJob, ProcessPaymentJob

// Migrations
2024_01_01_000000_create_users_table

// Seeders
UserSeeder, SubscriptionSeeder

// Factories
UserFactory, SubscriptionFactory
```

---

## 16. Convention Over Configuration

### ❌ Bad
```php
class Customer extends Model
{
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';
    
    protected $table = 'Customer';
    protected $primaryKey = 'customer_id';
    
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class, 'role_customer', 'customer_id', 'role_id');
    }
}
```

### ✅ Good
```php
class Customer extends Model
{
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }
}

// Laravel assumes:
// - Table: customers
// - Primary key: id
// - Foreign key: customer_id
// - Pivot table: customer_role
```

---

## 17. Use Shorter Syntax

### Common Patterns
```php
// Instead of
Session::get('cart')
$request->session()->get('cart')

// Use
session('cart')

// Instead of
$request->input('name')
Request::get('name')

// Use
$request->name
request('name')

// Instead of
return Redirect::back()

// Use
return back()

// Instead of
is_null($object->relation) ? null : $object->relation->id

// Use
optional($object->relation)->id
// Or PHP 8+
$object->relation?->id

// Instead of
->orderBy('created_at', 'desc')

// Use
->latest()

// Instead of
->orderBy('created_at', 'asc')

// Use
->oldest()

// Instead of
App::make('Class')

// Use
app('Class')
```

---

## 18. Use IoC Container Instead of `new`

### ❌ Bad (Tight Coupling)
```php
class UserService
{
    public function __construct()
    {
        $this->user = new User();
    }
}
```

### ✅ Good (Loose Coupling)
```php
class UserService
{
    public function __construct(protected User $user) {}
}
```

---

## 19. Don't Access .env Directly

### ❌ Bad
```php
$apiKey = env('API_KEY');
```

### ✅ Good
```php
// config/api.php
return [
    'key' => env('API_KEY'),
];

// Use it
$apiKey = config('api.key');
```

---

## 20. Store Dates Properly

### ❌ Bad
```blade
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
```

### ✅ Good
```php
// Model
protected $casts = [
    'ordered_at' => 'datetime',
];

// Blade
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->format('m-d') }}
```

---

## 21. Use Type Hints and Return Types

### ❌ Bad (No type hints)
```php
public function getUser($id)
{
    return User::find($id);
}
```

### ✅ Good (Full type hints)
```php
public function getUser(string $id): ?User
{
    return User::find($id);
}
```

---

## 22. No DocBlocks for Type Hints

### ❌ Bad
```php
/**
 * The function checks if given string is a valid ASCII string
 *
 * @param string $string String we get from frontend
 * @return bool
 * @author  John Smith
 * @license GPL
 */
public function checkString($string)
{
}
```

### ✅ Good
```php
public function isValidAsciiString(string $string): bool
{
}
```

---

## SaaS-Specific Best Practices

### 1. Always Scope to Tenant
```php
// Every query should include tenant
User::whereTenantId($tenantId)->get();

// Use global scope
protected static function boot(): void
{
    parent::boot();
    static::addGlobalScope(new TenantScope());
}
```

### 2. Use Soft Deletes
```php
// Never truly delete in SaaS
use SoftDeletes;

// Allows audit trail and data recovery
```

### 3. Implement Audit Logging
```php
// Track all important changes
AuditLog::create([
    'model_type' => get_class($model),
    'action' => 'updated',
    'changes' => $model->getDirty(),
]);
```

### 4. Use Authorization Policies
```php
// Every resource needs authorization
$this->authorize('update', $user);
```

### 5. Implement Rate Limiting
```php
// Prevent abuse
Route::middleware('throttle:60,1')->group(fn() => ...);
```

---

## Checklist for Clean Code

- [ ] Each class has ONE responsibility
- [ ] Methods are small and focused
- [ ] Logic is in models and services, not controllers
- [ ] Validation is in request classes
- [ ] Database queries use Eloquent
- [ ] No N+1 queries (use eager loading)
- [ ] No hard-coded values (use config)
- [ ] Names are descriptive
- [ ] Type hints are present
- [ ] No nested conditionals (max 2-3 levels)
- [ ] DRY principle is followed
- [ ] Tests cover business logic
- [ ] Error handling is proper
- [ ] All dependencies are injected
- [ ] Authorization is checked
- [ ] Data is logged/audited

---

Next: [05-AUTHENTICATION.md](./05-AUTHENTICATION.md)
