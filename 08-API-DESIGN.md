# 08 - API Design & Versioning

## Building Professional, Scalable APIs

---

## RESTful Principles

### HTTP Methods
```php
// GET - Retrieve resource(s)
GET /api/v1/users              // List all users
GET /api/v1/users/123          // Get specific user

// POST - Create resource
POST /api/v1/users             // Create new user

// PUT/PATCH - Update resource
PUT /api/v1/users/123          // Full update
PATCH /api/v1/users/123        // Partial update

// DELETE - Delete resource
DELETE /api/v1/users/123       // Delete user
```

### Status Codes
```php
// 2xx Success
200 OK              // Request successful
201 Created         // Resource created
202 Accepted        // Request accepted for processing
204 No Content      // Successful, no response body

// 4xx Client Error
400 Bad Request     // Invalid request
401 Unauthorized    // Authentication required
403 Forbidden       // Authorized but no access
404 Not Found       // Resource doesn't exist
422 Unprocessable   // Validation failed
429 Too Many Requests // Rate limit exceeded

// 5xx Server Error
500 Internal Error  // Server error
503 Service Unavailable // Maintenance
```

---

## API Routes

```php
// routes/api.php
Route::middleware('auth:sanctum')->prefix('v1')->group(function () {
    // User endpoints
    Route::apiResource('users', UserController::class);
    
    // Team endpoints
    Route::prefix('teams/{team}')->middleware('tenant')->group(function () {
        Route::get('/', [TeamController::class, 'show']);
        Route::patch('/', [TeamController::class, 'update']);
        Route::apiResource('members', TeamMemberController::class);
        Route::apiResource('invitations', TeamInvitationController::class);
    });
    
    // Project endpoints
    Route::prefix('teams/{team}/projects')->middleware('tenant')->group(function () {
        Route::get('/', [ProjectController::class, 'index']);
        Route::post('/', [ProjectController::class, 'store']);
        Route::get('{project}', [ProjectController::class, 'show']);
        Route::patch('{project}', [ProjectController::class, 'update']);
        Route::delete('{project}', [ProjectController::class, 'destroy']);
    });
});
```

---

## API Resources (Serialization)

```php
// app/Presentation/Http/Resources/UserResource.php
class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'email_verified_at' => $this->email_verified_at?->toIso8601String(),
            
            // Conditional data
            'is_owner' => $this->whenPivotLoaded(
                'team_user',
                fn() => $this->pivot->role === 'owner'
            ),
            
            // Relationships
            'team' => new TeamResource($this->whenLoaded('team')),
            'roles' => RoleResource::collection($this->whenLoaded('roles')),
            
            // Computed properties
            'profile_url' => $this->getProfileUrl(),
            
            // Timestamps
            'created_at' => $this->created_at->toIso8601String(),
            'updated_at' => $this->updated_at->toIso8601String(),
        ];
    }
}

// Collection wrapper
class UserCollection extends ResourceCollection
{
    public $collects = UserResource::class;

    public function toArray($request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total' => $this->resource->total(),
                'per_page' => $this->resource->perPage(),
                'current_page' => $this->resource->currentPage(),
            ],
        ];
    }
}

// Usage
class UserController
{
    public function index()
    {
        return UserCollection::make(User::paginate());
    }

    public function show(User $user)
    {
        return UserResource::make($user->load('team', 'roles'));
    }
}
```

---

## Error Responses

```php
// Global error handler
class Handler extends ExceptionHandler
{
    public function render($request, Throwable $exception)
    {
        if ($request->wantsJson()) {
            return response()->json($this->errorResponse($exception));
        }

        return parent::render($request, $exception);
    }

    private function errorResponse(Throwable $exception): array
    {
        return [
            'error' => [
                'message' => $this->getMessage($exception),
                'status' => $this->getStatus($exception),
                'errors' => $this->getValidationErrors($exception),
            ],
        ];
    }
}

// Validation error response
[
    'error' => [
        'message' => 'The given data was invalid',
        'status' => 422,
        'errors' => [
            'email' => ['Email is required', 'Email must be valid'],
            'password' => ['Password must be at least 8 characters'],
        ],
    ],
]

// Not found error
[
    'error' => [
        'message' => 'Resource not found',
        'status' => 404,
    ],
]

// Unauthorized error
[
    'error' => [
        'message' => 'Unauthorized',
        'status' => 401,
    ],
]
```

---

## Pagination

```php
// Cursor pagination (better for large datasets)
class UserController
{
    public function index()
    {
        $users = User::orderBy('id')->cursorPaginate(50);
        return UserResource::collection($users);
    }
}

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

// Offset pagination (for smaller datasets)
class UserController
{
    public function index()
    {
        $users = User::paginate(15);
        return UserResource::collection($users);
    }
}

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
```

---

## Filtering & Searching

```php
// Query parameters
GET /api/v1/users?status=active&team_id=123&sort=-created_at&search=john

// Using spatie/laravel-query-builder
class UserController
{
    public function index()
    {
        return UserResource::collection(
            QueryBuilder::for(User::class)
                ->allowedFilters([
                    'status',
                    AllowedFilter::partial('name'),
                    AllowedFilter::callback('team_id', fn($q, $value) => 
                        $q->where('team_id', $value)
                    ),
                ])
                ->allowedSorts([
                    'created_at',
                    'name',
                    'email',
                    '-created_at', // Descending
                ])
                ->allowedIncludes(['team', 'roles'])
                ->paginate()
        );
    }
}

// Usage
GET /api/v1/users?filter[status]=active&filter[name]=john&sort=-created_at&include=team,roles
```

---

## API Versioning

### URL-Based (Recommended)
```php
// routes/api.php
Route::prefix('v1')->group(function () {
    Route::apiResource('users', UserControllerV1::class);
});

Route::prefix('v2')->group(function () {
    Route::apiResource('users', UserControllerV2::class);
});
```

### Header-Based
```php
// Request: Accept: application/vnd.api+json;version=1

Route::middleware('api.version:1')->group(fn() => ...);
```

### Deprecation Headers
```php
class DeprecationMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        if ($request->is('api/v1/*')) {
            return $response
                ->header('Deprecation', 'true')
                ->header('Sunset', now()->addMonths(6)->toRfc7231String())
                ->header('Link', '</api/v2>; rel="successor-version"');
        }

        return $response;
    }
}
```

---

## Request Validation

```php
class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return auth()->check();
    }

    public function rules(): array
    {
        return [
            'email' => 'required|email|unique:users',
            'name' => 'required|string|max:255',
            'password' => 'required|string|min:8|confirmed',
            'role' => 'required|in:admin,user',
        ];
    }

    public function messages(): array
    {
        return [
            'email.unique' => 'An account with this email already exists',
            'password.min' => 'Password must be at least 8 characters',
        ];
    }
}
```

---

## API Documentation

### Using Scribe
```bash
composer require knuckleswtf/scribe
php artisan scribe:generate
```

```php
/**
 * Get all users
 * 
 * @authenticated
 * @response 200 {
 *   "data": [...],
 *   "meta": {...}
 * }
 */
public function index()
{
    // ...
}
```

### OpenAPI/Swagger
```php
/**
 * @OA\Get(
 *     path="/api/v1/users",
 *     summary="List users",
 *     @OA\Response(response=200, description="Users list"),
 * )
 */
public function index()
{
    // ...
}
```

---

## API Checklist

- [ ] Consistent HTTP methods
- [ ] Proper status codes
- [ ] Standard error responses
- [ ] Resources for serialization
- [ ] Pagination implemented
- [ ] Filtering/searching available
- [ ] API versioning strategy
- [ ] Authentication required
- [ ] Rate limiting enabled
- [ ] Request validation
- [ ] Deprecation notices
- [ ] Documentation complete
- [ ] Tests cover endpoints
- [ ] CORS configured
- [ ] API logging enabled

---

Next: [09-SERVICE-LAYER.md](./09-SERVICE-LAYER.md)
