# 05 - Authentication & Authorization

## Multi-Tenant Authentication for SaaS

---

## Authentication Types

### 1. Web (Session-based)
For user-facing web applications with cookies.

### 2. API (Token-based)
For mobile apps, desktop clients, and third-party integrations.

### 3. Social OAuth
Google, GitHub, Microsoft login.

---

## Web Authentication (Session-Based)

```php
// config/auth.php
return [
    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
    ],
    'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\Models\User::class,
        ],
    ],
];

// routes/auth.php
Route::middleware('guest')->group(function () {
    Route::get('login', [AuthController::class, 'showLogin'])->name('login');
    Route::post('login', [AuthController::class, 'login']);
    Route::get('register', [AuthController::class, 'showRegister'])->name('register');
    Route::post('register', [AuthController::class, 'register']);
});

Route::middleware('auth:web')->group(function () {
    Route::post('logout', [AuthController::class, 'logout'])->name('logout');
});

// Controller
class AuthController extends Controller
{
    public function login(LoginRequest $request)
    {
        if (!auth()->attempt($request->validated())) {
            return back()->withErrors(['email' => 'Invalid credentials']);
        }

        return redirect()->intended('/dashboard');
    }

    public function register(RegisterRequest $request)
    {
        $user = User::create($request->validated());
        auth()->login($user);
        return redirect('/dashboard');
    }

    public function logout()
    {
        auth()->logout();
        return redirect('/login');
    }
}
```

---

## API Authentication (Token-Based with Sanctum)

```php
// config/auth.php
'guards' => [
    'api' => [
        'driver' => 'sanctum',
        'provider' => 'users',
    ],
],

// routes/api.php
Route::post('login', [AuthController::class, 'apiLogin']);
Route::post('register', [AuthController::class, 'apiRegister']);

Route::middleware('auth:sanctum')->group(function () {
    Route::get('user', fn() => auth()->user());
    Route::post('logout', [AuthController::class, 'apiLogout']);
    Route::apiResource('users', UserController::class);
});

// Controller
class AuthController extends Controller
{
    public function apiRegister(RegisterRequest $request)
    {
        $user = User::create($request->validated());
        $token = $user->createToken('api-token')->plainTextToken;
        
        return response()->json([
            'user' => UserResource::make($user),
            'token' => $token,
        ], 201);
    }

    public function apiLogin(LoginRequest $request)
    {
        if (!auth()->attempt($request->validated())) {
            return response()->json(
                ['message' => 'Unauthorized'],
                401
            );
        }

        $user = auth()->user();
        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'user' => UserResource::make($user),
            'token' => $token,
        ]);
    }

    public function apiLogout()
    {
        auth()->user()->currentAccessToken()->delete();
        return response()->json(['message' => 'Logged out']);
    }
}
```

---

## Social Authentication (OAuth)

```php
// Install: composer require laravel/socialite

// config/services.php
return [
    'github' => [
        'client_id' => env('GITHUB_CLIENT_ID'),
        'client_secret' => env('GITHUB_CLIENT_SECRET'),
        'redirect' => env('APP_URL') . '/auth/github/callback',
    ],
];

// routes
Route::get('auth/github', [SocialAuthController::class, 'redirectToGithub']);
Route::get('auth/github/callback', [SocialAuthController::class, 'handleGithubCallback']);

// Controller
class SocialAuthController extends Controller
{
    public function redirectToGithub()
    {
        return Socialite::driver('github')->redirect();
    }

    public function handleGithubCallback()
    {
        try {
            $user = Socialite::driver('github')->user();
        } catch (Exception $e) {
            return redirect('/login')->with('error', 'Authentication failed');
        }

        $authUser = User::firstOrCreate(
            ['github_id' => $user->getId()],
            [
                'name' => $user->getName(),
                'email' => $user->getEmail(),
                'email_verified_at' => now(),
            ]
        );

        auth()->login($authUser);
        return redirect('/dashboard');
    }
}
```

---

## Multi-Tenant Authentication

Ensure users can only access their tenant's data.

```php
// Middleware
class TenantMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        if (!auth()->check()) {
            return redirect('/login');
        }

        $tenant = auth()->user()->team;
        if (!$tenant) {
            auth()->logout();
            return redirect('/login')->with('error', 'No team assigned');
        }

        // Set current tenant in container
        app()->instance('current_tenant', $tenant);
        
        // Verify tenant from URL matches user's team
        if ($request->route('team')) {
            if ($request->route('team')->id !== $tenant->id) {
                return abort(403);
            }
        }

        return $next($request);
    }
}

// Global Scope
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $tenant = app('current_tenant');
        if ($tenant) {
            $builder->where('team_id', $tenant->id);
        }
    }
}

// Model
class User extends Model
{
    protected static function boot(): void
    {
        parent::boot();
        static::addGlobalScope(new TenantScope());
    }
}
```

---

## Email Verification

```php
// Migration
Schema::table('users', function (Blueprint $table) {
    $table->timestamp('email_verified_at')->nullable();
});

// Model
class User extends Model implements MustVerifyEmail
{
    // ...
}

// routes
Route::middleware('auth')->group(function () {
    Route::get('email/verify', [VerifyEmailController::class, 'show'])
        ->name('verification.notice');
    Route::get('email/verify/{id}/{hash}', [VerifyEmailController::class, 'verify'])
        ->middleware('signed')
        ->name('verification.verify');
    Route::post('email/resend', [VerifyEmailController::class, 'send'])
        ->middleware('throttle:6,1')
        ->name('verification.send');
});

// Middleware
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', fn() => view('dashboard'));
});
```

---

## Two-Factor Authentication

```php
// Install: composer require laravel/fortify

// Migration
Schema::table('users', function (Blueprint $table) {
    $table->string('two_factor_secret')->nullable();
    $table->text('two_factor_recovery_codes')->nullable();
});

// Enable 2FA
class Enable2FAController
{
    public function store(Request $request)
    {
        $user = $request->user();
        $secret = app(TwoFactorAuthenticationProvider::class)->generateSecretKey();

        // Store in session temporarily
        session(['two_factor_secret' => $secret]);

        return response()->json([
            'svg' => QrCode::size(200)->generate(
                app(TwoFactorAuthenticationProvider::class)->getQrCodeUrl(
                    $user->email,
                    $secret
                )
            ),
        ]);
    }

    public function confirm(Request $request)
    {
        $request->validate(['code' => 'required|numeric']);

        $secret = session('two_factor_secret');

        if (!app(TwoFactorAuthenticationProvider::class)->verify($request->code, $secret)) {
            return back()->withErrors(['code' => 'Invalid code']);
        }

        $request->user()->update([
            'two_factor_secret' => $secret,
            'two_factor_recovery_codes' => json_encode(
                app(TwoFactorAuthenticationProvider::class)->generateRecoveryCodes()
            ),
        ]);

        return redirect('/dashboard')->with('message', '2FA enabled');
    }
}
```

---

## Authorization (Policies)

```php
// Generate: php artisan make:policy UserPolicy --model=User

class UserPolicy
{
    use HandlesAuthorization;

    // Super admin bypasses all checks
    public function before(User $user, string $ability): ?bool
    {
        if ($user->hasRole('admin')) {
            return true;
        }
        return null;
    }

    // User can only view themselves or team members
    public function view(User $user, User $model): bool
    {
        return $user->id === $model->id || 
               $user->team_id === $model->team_id;
    }

    // User can only update themselves
    public function update(User $user, User $model): bool
    {
        return $user->id === $model->id;
    }

    // Only team owner can delete
    public function delete(User $user, User $model): bool
    {
        return $user->id === $model->team->owner_id;
    }
}

// Usage
class UserController
{
    public function show(User $user)
    {
        $this->authorize('view', $user);
        return UserResource::make($user);
    }

    public function update(UpdateUserRequest $request, User $user)
    {
        $this->authorize('update', $user);
        $user->update($request->validated());
        return UserResource::make($user);
    }

    public function destroy(User $user)
    {
        $this->authorize('delete', $user);
        $user->delete();
        return response()->noContent();
    }
}
```

---

## Role-Based Access Control (RBAC)

```php
// Install: composer require spatie/laravel-permission

// Models
class User extends Model
{
    use HasRoles;
}

class Role extends Model
{
    use HasPermissions;
}

// Setup
class RoleSeeder
{
    public function run()
    {
        $adminRole = Role::create(['name' => 'admin']);
        $ownerRole = Role::create(['name' => 'team_owner']);
        $memberRole = Role::create(['name' => 'team_member']);

        $adminRole->givePermissionTo([
            'view_users',
            'edit_users',
            'delete_users',
        ]);

        $ownerRole->givePermissionTo([
            'view_team',
            'invite_members',
            'remove_members',
            'update_team',
        ]);

        $memberRole->givePermissionTo([
            'view_team',
        ]);
    }
}

// Usage
if (auth()->user()->hasPermissionTo('edit_users')) {
    // ...
}

if (auth()->user()->hasRole('admin')) {
    // ...
}

// Middleware
Route::middleware('role:admin')->group(fn() => ...);
Route::middleware('permission:edit_users')->group(fn() => ...);
```

---

## API Token Management

```php
class ApiToken extends Model
{
    protected $fillable = ['user_id', 'token', 'name', 'last_used_at'];
}

class ApiTokenController
{
    public function store(Request $request)
    {
        $token = $request->user()->createToken(
            $request->input('name'),
            ['api']
        );

        return response()->json([
            'token' => $token->plainTextToken,
        ], 201);
    }

    public function destroy(ApiToken $token)
    {
        $this->authorize('delete', $token);
        $token->delete();
        return response()->noContent();
    }
}
```

---

## Password Reset

```php
// Generate: php artisan make:request ForgotPasswordRequest

Route::post('forgot-password', [PasswordResetController::class, 'store']);
Route::get('reset-password/{token}', [PasswordResetController::class, 'show']);
Route::post('reset-password', [PasswordResetController::class, 'update']);

class PasswordResetController
{
    public function store(ForgotPasswordRequest $request)
    {
        $user = User::where('email', $request->email)->first();
        if (!$user) {
            return back()->with('status', 'Password reset link sent');
        }

        Password::sendResetLink(['email' => $request->email]);
        return back()->with('status', 'Password reset link sent');
    }

    public function update(ResetPasswordRequest $request)
    {
        $status = Password::reset(
            $request->validated(),
            function ($user, $password) {
                $user->update(['password' => Hash::make($password)]);
            }
        );

        return $status === Password::PASSWORD_RESET
            ? redirect('/login')->with('status', 'Password reset')
            : back()->withErrors(['email' => trans($status)]);
    }
}
```

---

## Authentication Checklist

- [ ] Web authentication uses sessions
- [ ] API uses tokens (Sanctum)
- [ ] Email is verified
- [ ] Password reset works
- [ ] 2FA is optional for users
- [ ] Roles and permissions are defined
- [ ] Authorization policies exist
- [ ] Multi-tenancy is enforced
- [ ] Rate limiting on auth endpoints
- [ ] Failed login attempts are logged
- [ ] Sessions timeout properly
- [ ] Remember me functionality works

---

Next: [06-MULTI-TENANCY.md](./06-MULTI-TENANCY.md)
