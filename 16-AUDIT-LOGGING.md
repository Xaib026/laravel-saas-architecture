# 16 - Audit Logging & Change Tracking

## Complete Audit Trail for Compliance

---

## Audit Log Schema

```php
// Migration
Schema::create('audit_logs', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->nullable()->constrained('users');
    $table->string('model_type'); // App\Models\User
    $table->unsignedBigInteger('model_id');
    $table->string('action'); // created, updated, deleted
    $table->json('old_values')->nullable(); // Previous values
    $table->json('new_values')->nullable(); // New values
    $table->json('changes')->nullable(); // What changed
    $table->ipAddress('ip_address')->nullable();
    $table->string('user_agent')->nullable();
    $table->timestamps();
    
    $table->index(['model_type', 'model_id']);
    $table->index('user_id');
    $table->index('created_at');
});
```

---

## Audit Log Model

```php
// Domain/Audit/AuditLog.php
class AuditLog extends Model
{
    protected $fillable = [
        'user_id',
        'model_type',
        'model_id',
        'action',
        'old_values',
        'new_values',
        'changes',
        'ip_address',
        'user_agent',
    ];

    protected $casts = [
        'old_values' => 'json',
        'new_values' => 'json',
        'changes' => 'json',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public static function log(
        Model $model,
        string $action,
        array $oldValues = [],
        array $newValues = []
    ): self {
        return self::create([
            'user_id' => auth()->id(),
            'model_type' => get_class($model),
            'model_id' => $model->id,
            'action' => $action,
            'old_values' => $oldValues,
            'new_values' => $newValues,
            'changes' => self::computeChanges($oldValues, $newValues),
            'ip_address' => request()->ip(),
            'user_agent' => request()->header('User-Agent'),
        ]);
    }

    private static function computeChanges(array $old, array $new): array
    {
        $changes = [];
        foreach ($new as $key => $value) {
            if (!isset($old[$key]) || $old[$key] !== $value) {
                $changes[$key] = [
                    'from' => $old[$key] ?? null,
                    'to' => $value,
                ];
            }
        }
        return $changes;
    }
}
```

---

## Automatic Auditing with Events

```php
// Trait for auditable models
trait Auditable
{
    protected static function boot(): void
    {
        parent::boot();

        static::created(function ($model) {
            AuditLog::log($model, 'created', [], $model->toArray());
        });

        static::updating(function ($model) {
            $model->auditOldValues = $model->getOriginal();
        });

        static::updated(function ($model) {
            if (isset($model->auditOldValues)) {
                AuditLog::log(
                    $model,
                    'updated',
                    $model->auditOldValues,
                    $model->toArray()
                );
            }
        });

        static::deleted(function ($model) {
            AuditLog::log($model, 'deleted', $model->toArray(), []);
        });
    }
}

// Usage in model
class User extends Model
{
    use Auditable;
}
```

---

## Manual Audit Logging

```php
// In service
class UserService
{
    public function updateUser(User $user, array $data): User
    {
        $oldValues = $user->only(array_keys($data));
        $user->update($data);
        
        AuditLog::log($user, 'updated', $oldValues, $user->toArray());
        
        return $user;
    }
}
```

---

## Query Audit Logs

```php
// Get all changes to a user
$logs = AuditLog::where('model_type', User::class)
    ->where('model_id', $userId)
    ->orderBy('created_at', 'desc')
    ->get();

// Get user's actions
$logs = AuditLog::where('user_id', auth()->id())
    ->orderBy('created_at', 'desc')
    ->paginate();

// Get specific action
$deletedUsers = AuditLog::where('model_type', User::class)
    ->where('action', 'deleted')
    ->get();

// Find changes to specific field
$emailChanges = AuditLog::where('model_type', User::class)
    ->whereJsonContains('changes->email')
    ->get();
```

---

## Sensitive Data Handling

```php
// Don't audit sensitive fields
class User extends Model
{
    use Auditable;

    protected $hidden = ['password', 'api_token'];
    
    // Exclude from audit
    protected static $auditExcluded = [
        'password',
        'api_token',
        'remember_token',
    ];
}

// Custom auditing with exclusions
trait AuditableWithExclusions
{
    protected static $auditExcluded = [];

    protected static function boot(): void
    {
        parent::boot();

        static::updated(function ($model) {
            $oldValues = array_filter(
                $model->auditOldValues,
                fn($key) => !in_array($key, static::$auditExcluded),
                ARRAY_FILTER_USE_KEY
            );

            $newValues = array_filter(
                $model->toArray(),
                fn($key) => !in_array($key, static::$auditExcluded),
                ARRAY_FILTER_USE_KEY
            );

            AuditLog::log($model, 'updated', $oldValues, $newValues);
        });
    }
}
```

---

## Audit Log API

```php
// routes/api.php
Route::middleware(['auth:sanctum', 'tenant'])->group(function () {
    Route::get('audit-logs', [AuditLogController::class, 'index']);
    Route::get('users/{user}/audit-logs', [AuditLogController::class, 'forUser']);
});

// Controller
class AuditLogController
{
    public function index()
    {
        $logs = AuditLog::where('user_id', auth()->id())
            ->latest()
            ->paginate();

        return AuditLogResource::collection($logs);
    }

    public function forUser(User $user)
    {
        $this->authorize('view', $user);

        $logs = AuditLog::where('model_type', User::class)
            ->where('model_id', $user->id)
            ->latest()
            ->paginate();

        return AuditLogResource::collection($logs);
    }
}

// Resource
class AuditLogResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'user' => UserResource::make($this->whenLoaded('user')),
            'action' => $this->action,
            'model_type' => class_basename($this->model_type),
            'model_id' => $this->model_id,
            'changes' => $this->changes,
            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

---

## Data Retention

```php
// Prune old audit logs
class PruneAuditLogsCommand extends Command
{
    public function handle(): void
    {
        AuditLog::where('created_at', '<', now()->subMonths(12))
            ->delete();

        $this->info('Audit logs pruned');
    }
}

// Schedule
protected function schedule(Schedule $schedule): void
{
    $schedule->command('audit:prune')
        ->monthlyOn(1, '02:00');
}
```

---

Next: [17-RATE-LIMITING.md](./17-RATE-LIMITING.md)
