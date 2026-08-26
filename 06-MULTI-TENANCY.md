# 06 - Multi-Tenancy Implementation

## Complete Multi-Tenant SaaS Architecture

Multi-tenancy allows multiple customers (teams) to use the same application while keeping their data completely isolated.

---

## Multi-Tenancy Models

### 1. Database-Per-Tenant
**Pros**: Complete isolation, easy scaling
**Cons**: More complex, more databases

```php
// Not recommended for most SaaS - use Shared Database approach
```

### 2. Shared Database with Tenant Column (RECOMMENDED)
**Pros**: Simple, cost-effective, easy scaling
**Cons**: Data isolation must be enforced in code

```php
// This guide uses this approach
```

### 3. Hybrid
Combination of both approaches.

---

## Database Schema

```php
// Migration: Create teams table
Schema::create('teams', function (Blueprint $table) {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->string('name');
    $table->string('slug')->unique();
    $table->foreignId('owner_id')->constrained('users');
    $table->string('plan')->default('free'); // subscription plan
    $table->integer('max_members')->default(5);
    $table->integer('max_projects')->default(10);
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
});

// Migration: Add team_id to users table
Schema::table('users', function (Blueprint $table) {
    $table->foreignId('team_id')->constrained('teams')->cascadeOnDelete();
    $table->enum('role', ['owner', 'admin', 'member'])->default('member');
});

// Migration: All other tables have team_id
Schema::create('projects', function (Blueprint $table) {
    $table->id();
    $table->foreignId('team_id')->constrained('teams')->cascadeOnDelete();
    $table->string('name');
    $table->text('description')->nullable();
    $table->timestamps();
    $table->softDeletes();
    
    // Index for queries
    $table->index('team_id');
    $table->unique(['team_id', 'name']);
});
```

---

## Global Scope for Tenant Isolation

```php
// app/Shared/Traits/BelongsToTenant.php
namespace App\Shared\Traits;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Scope;
use Illuminate\Database\Eloquent\Model;

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

// Trait for models
trait BelongsToTenant
{
    protected static function boot(): void
    {
        parent::boot();
        
        // Auto-fill team_id on create
        static::creating(function ($model) {
            if (!$model->team_id && app('current_tenant')) {
                $model->team_id = app('current_tenant')->id;
            }
        });
        
        // Always scope queries to current tenant
        static::addGlobalScope(new TenantScope());
    }

    public function team()
    {
        return $this->belongsTo(Team::class);
    }
}

// Usage in Model
class Project extends Model
{
    use BelongsToTenant;
}
```

---

## Tenant Middleware

```php
// app/Presentation/Http/Middleware/TenantMiddleware.php
namespace App\Presentation\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class TenantMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        // Not authenticated
        if (!Auth::check()) {
            return redirect('/login');
        }

        $user = Auth::user();
        $tenant = $user->team;

        // User has no team
        if (!$tenant) {
            Auth::logout();
            return redirect('/login')->with('error', 'No team assigned');
        }

        // Set current tenant
        app()->instance('current_tenant', $tenant);

        // Verify route tenant matches user's team
        if ($request->route('team')) {
            $routeTenant = $request->route('team');
            
            if ($routeTenant->id !== $tenant->id) {
                return abort(403, 'Unauthorized to access this team');
            }
        }

        // Set tenant in config for queries
        config(['app.current_tenant_id' => $tenant->id]);

        return $next($request);
    }
}

// Register in Kernel
protected $routeMiddleware = [
    'tenant' => TenantMiddleware::class,
];
```

---

## Routes with Tenant Scoping

```php
// routes/api.php
Route::prefix('api/v1')->middleware(['auth:sanctum', 'tenant'])->group(function () {
    Route::prefix('teams/{team}')->group(function () {
        Route::apiResource('projects', ProjectController::class);
        Route::apiResource('members', TeamMemberController::class);
        Route::apiResource('invitations', TeamInvitationController::class);
    });
});

// routes/web.php
Route::middleware(['auth', 'verified', 'tenant'])->group(function () {
    Route::prefix('teams/{team}')->group(function () {
        Route::get('dashboard', [DashboardController::class, 'show']);
        Route::get('settings', [SettingsController::class, 'show']);
        Route::post('settings', [SettingsController::class, 'update']);
        Route::resource('projects', ProjectController::class);
    });
});
```

---

## Team Management

```php
// Models
class Team extends Model
{
    protected $fillable = ['name', 'slug', 'owner_id', 'plan'];

    public function owner()
    {
        return $this->belongsTo(User::class, 'owner_id');
    }

    public function members()
    {
        return $this->belongsToMany(User::class)
            ->withPivot('role')
            ->withTimestamps();
    }

    public function hasUser(User $user): bool
    {
        return $this->members()->where('user_id', $user->id)->exists();
    }

    public function addMember(User $user, string $role = 'member'): void
    {
        $this->members()->attach($user, ['role' => $role]);
        event(new MemberAddedToTeam($this, $user));
    }

    public function removeMember(User $user): void
    {
        $this->members()->detach($user);
        event(new MemberRemovedFromTeam($this, $user));
    }
}

class User extends Model
{
    public function team()
    {
        return $this->belongsTo(Team::class);
    }

    public function teams()
    {
        return $this->belongsToMany(Team::class)
            ->withPivot('role')
            ->withTimestamps();
    }

    public function isTeamOwner(Team $team): bool
    {
        return $team->owner_id === $this->id;
    }

    public function hasTeamRole(Team $team, string $role): bool
    {
        return $team->members()
            ->where('user_id', $this->id)
            ->wherePivot('role', $role)
            ->exists();
    }
}
```

---

## Team Invitation System

```php
// Migration
Schema::create('team_invitations', function (Blueprint $table) {
    $table->id();
    $table->foreignId('team_id')->constrained('teams')->cascadeOnDelete();
    $table->string('email');
    $table->string('token')->unique();
    $table->enum('role', ['admin', 'member'])->default('member');
    $table->timestamp('accepted_at')->nullable();
    $table->timestamps();
    
    $table->index('team_id');
    $table->index('email');
});

// Model
class TeamInvitation extends Model
{
    protected $fillable = ['team_id', 'email', 'role', 'token'];
    protected $casts = ['accepted_at' => 'datetime'];

    public static function create(Team $team, string $email, string $role = 'member')
    {
        return parent::create([
            'team_id' => $team->id,
            'email' => $email,
            'role' => $role,
            'token' => Str::random(32),
        ]);
    }

    public function accept(User $user): void
    {
        if ($this->email !== $user->email) {
            throw new InvalidInvitationException('Email does not match');
        }

        $this->team->addMember($user, $this->role);
        $this->update(['accepted_at' => now()]);
        event(new InvitationAccepted($this));
    }
}

// Controller
class TeamInvitationController
{
    public function store(Request $request, Team $team)
    {
        $this->authorize('inviteMembers', $team);

        $invitation = TeamInvitation::create(
            $team,
            $request->email,
            $request->role
        );

        Mail::to($invitation->email)->send(
            new SendTeamInvitationMail($invitation)
        );

        return response()->json($invitation, 201);
    }

    public function accept(TeamInvitation $invitation)
    {
        if (!auth()->check()) {
            return redirect('/login');
        }

        $invitation->accept(auth()->user());
        return redirect('/dashboard')->with('message', 'Invitation accepted');
    }
}
```

---

## Policies with Tenant Authorization

```php
class ProjectPolicy
{
    use HandlesAuthorization;

    public function viewAny(User $user): bool
    {
        return auth()->check();
    }

    public function view(User $user, Project $project): bool
    {
        // Only team members can view
        return $project->team_id === $user->team_id &&
               $project->team->hasUser($user);
    }

    public function create(User $user): bool
    {
        $team = $user->team;
        // Check if team can create more projects
        return $team->projects()->count() < $team->max_projects &&
               $user->hasTeamRole($team, ['owner', 'admin']);
    }

    public function update(User $user, Project $project): bool
    {
        return $project->team_id === $user->team_id &&
               $user->hasTeamRole($project->team, ['owner', 'admin']);
    }

    public function delete(User $user, Project $project): bool
    {
        return $this->update($user, $project);
    }
}
```

---

## Enforcing Tenant Isolation

```php
// Service Provider - Force tenant scoping
class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // In production, verify tenant on every query
        if (app()->isProduction()) {
            Model::handleLazyLoadingViolationUsing(function () {
                throw new LazyLoadingException();
            });
        }

        // Verify no queries bypass tenant scope
        DB::listen(function ($query) {
            // Check query doesn't access other tenants' data
            if (should_validate_tenant() && !has_tenant_filter($query)) {
                Log::warning('Query missing tenant filter', [
                    'query' => $query->sql,
                ]);
            }
        });
    }
}
```

---

## Testing Multi-Tenancy

```php
class ProjectControllerTest extends TestCase
{
    private User $user1;
    private User $user2;
    private Team $team1;
    private Team $team2;

    protected function setUp(): void
    {
        parent::setUp();

        $this->team1 = Team::factory()->create();
        $this->team2 = Team::factory()->create();

        $this->user1 = User::factory()->create(['team_id' => $this->team1->id]);
        $this->user2 = User::factory()->create(['team_id' => $this->team2->id]);

        $this->team1->addMember($this->user1, 'owner');
        $this->team2->addMember($this->user2, 'owner');
    }

    public function test_user_can_only_view_own_team_projects()
    {
        $project1 = Project::factory()->create(['team_id' => $this->team1->id]);
        $project2 = Project::factory()->create(['team_id' => $this->team2->id]);

        $this->actingAs($this->user1)
            ->get("/api/v1/teams/{$this->team1->id}/projects/{$project1->id}")
            ->assertOk();

        // User1 should NOT see user2's project
        $this->actingAs($this->user1)
            ->get("/api/v1/teams/{$this->team2->id}/projects/{$project2->id}")
            ->assertForbidden();
    }

    public function test_team_scope_filters_queries()
    {
        Project::factory(3)->create(['team_id' => $this->team1->id]);
        Project::factory(2)->create(['team_id' => $this->team2->id]);

        app()->instance('current_tenant', $this->team1);

        $this->assertEquals(3, Project::count());
        $this->assertEquals(5, Project::withoutGlobalScopes()->count());
    }
}
```

---

## Best Practices

✅ **Always use middleware**: Apply `tenant` middleware to all protected routes
✅ **Always use traits**: Add `BelongsToTenant` to all tenant-scoped models
✅ **Always verify ownership**: Use policies to check team membership
✅ **Always test isolation**: Write tests verifying data isolation
✅ **Query logging**: Monitor queries to ensure tenant filtering
✅ **Default scopes**: Use global scopes to prevent accidental leaks
✅ **Database indexes**: Index team_id for performance
✅ **Unique constraints**: Make fields unique per tenant

---

## Multi-Tenancy Checklist

- [ ] All tables have team_id
- [ ] Global scope applied to all models
- [ ] Tenant middleware on all protected routes
- [ ] Policies verify team membership
- [ ] Team invitations work
- [ ] Team members can be removed
- [ ] Admin can manage team settings
- [ ] Data is isolated per team
- [ ] Tests verify isolation
- [ ] No data leaks in queries

---

Next: [07-DATABASE-DESIGN.md](./07-DATABASE-DESIGN.md)
