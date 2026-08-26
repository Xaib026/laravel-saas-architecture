# 11 - Jobs & Queue Management

## Background Processing for Long-Running Tasks

---

## Job Classes

```php
// Infrastructure/Jobs/SendWelcomeEmailJob.php
namespace App\Infrastructure\Jobs;

use App\Domain\Users\User;
use App\Infrastructure\Mail\WelcomeEmail;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Log;
use Throwable;

class SendWelcomeEmailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // Configuration
    public $queue = 'emails';
    public $tries = 3;                    // Retry 3 times
    public $timeout = 60;                 // 60 second timeout
    public $backoff = [10, 60, 300];     // Backoff delays
    public $maxExceptions = 1;            // Max exceptions before fail

    public function __construct(
        public User $user,
    ) {}

    public function handle(): void
    {
        try {
            Mail::to($this->user->email())
                ->send(new WelcomeEmail($this->user));
        } catch (Throwable $e) {
            Log::error('Welcome email failed', [
                'user_id' => $this->user->id,
                'error' => $e->getMessage(),
            ]);
            throw $e; // Let queue retry
        }
    }

    public function failed(Throwable $exception): void
    {
        // Called after all retries exhausted
        Log::critical('Welcome email permanently failed', [
            'user_id' => $this->user->id,
            'error' => $exception->getMessage(),
        ]);

        // Notify admin
        Notification::route('mail', config('admin.email'))
            ->notify(new AdminNotification('Email failed for user'));
    }

    public function middleware(): array
    {
        return [new RateLimited('emails')];
    }
}

// More job examples
class ProcessPaymentJob implements ShouldQueue
{
    public $tries = 5;
    public $maxExceptions = 2;

    public function handle(PaymentService $service): void
    {
        $service->process($this->payment);
    }
}

class GenerateReportJob implements ShouldQueue
{
    public $timeout = 3600; // 1 hour
    public $queue = 'reports';

    public function handle(): void
    {
        // Generate large report
    }
}

class ExportDataJob implements ShouldQueue
{
    public $timeout = 1800; // 30 minutes
    public $queue = 'exports';

    public function handle(): void
    {
        // Export user data
        Storage::put('exports/user.csv', $this->generateCsv());
    }
}
```

---

## Dispatching Jobs

```php
// Immediate dispatch
SendWelcomeEmailJob::dispatch($user);

// Delayed dispatch
SendWelcomeEmailJob::dispatch($user)
    ->delay(now()->addHours(1));

// To specific queue
SendWelcomeEmailJob::dispatch($user)
    ->onQueue('emails');

// After response
SendWelcomeEmailJob::dispatch($user)
    ->afterResponse();

// Batch jobs
Bus::batch([
    new SendEmailJob($user1),
    new SendEmailJob($user2),
    new SendEmailJob($user3),
])
    ->then(fn() => Log::info('All emails sent'))
    ->catch(fn(Throwable $e) => Log::error('Batch failed', ['error' => $e]))
    ->finally(fn() => Log::info('Batch complete'))
    ->dispatch();

// Chained jobs
Bus::chain([
    new ProcessPayment($order),
    new SendConfirmationEmail($order),
    new UpdateInventory($order),
])->dispatch();
```

---

## Queue Configuration

```php
// config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'database'),

    'connections' => [
        // Database queue (simple, suitable for small apps)
        'database' => [
            'driver' => 'database',
            'table' => 'jobs',
            'queue' => 'default',
            'retry_after' => 90,
        ],

        // Redis queue (high performance, recommended)
        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => 'default',
            'retry_after' => 90,
            'block_for' => null,
        ],

        // SQS queue (AWS, for scale)
        'sqs' => [
            'driver' => 'sqs',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/123456789012'),
            'queue' => env('SQS_QUEUE', 'default'),
            'suffix' => env('SQS_SUFFIX'),
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        ],
    ],

    'failed' => [
        'driver' => env('QUEUE_FAILED_DRIVER', 'database'),
        'database' => env('DB_CONNECTION', 'mysql'),
        'table' => 'failed_jobs',
    ],
];
```

---

## Failed Jobs

```php
// Retry failed jobs
php artisan queue:retry all
php artisan queue:retry 5 10 15

// Forget failed jobs
php artisan queue:forget

// Monitor failed jobs table
Schema::create('failed_jobs', function (Blueprint $table) {
    $table->id();
    $table->string('uuid')->unique();
    $table->text('connection');
    $table->text('queue');
    $table->longText('payload');
    $table->longText('exception');
    $table->timestamp('failed_at')->useCurrent();
});

// Failed job handler
class SendEmailJob implements ShouldQueue
{
    public function failed(Throwable $exception): void
    {
        // Send notification
        Notification::route('slack', env('SLACK_WEBHOOK'))
            ->notify(new JobFailedNotification($exception));
    }
}
```

---

## Task Scheduling

```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule): void
{
    // Run command every minute
    $schedule->command('inspire')->everyMinute();

    // Run at specific time
    $schedule->command('subscriptions:check-expiry')
        ->dailyAt('02:00')
        ->timezone('America/Chicago');

    // Run job
    $schedule->job(new CleanupOldLogsJob)
        ->dailyAt('03:00');

    // Run closure
    $schedule->call(function () {
        User::inactive()->delete();
    })->weeklyOn(0, '03:00');

    // Only on production
    $schedule->command('backup:run')
        ->daily()
        ->onOneServer()
        ->onSuccess(fn() => Log::info('Backup successful'))
        ->onFailure(fn() => Log::error('Backup failed'));
}

// Schedule a callback
$schedule->call(new CallbackClass)->everyFiveMinutes();

// Timezone specific
$schedule->command('analytics:process')
    ->dailyAt('00:00')
    ->timezone('UTC');
```

---

## Worker Management

```bash
# Start queue worker
php artisan queue:work

# Work on specific queue
php artisan queue:work --queue=emails,payments

# Limit memory
php artisan queue:work --memory=256

# Timeout per job
php artisan queue:work --timeout=60

# Daemonize with supervisor
# /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work sqs --sleep=3 --tries=3
autostart=true
autorestart=true
numprocs=4
redirect_stderr=true
stdout_logfile=/var/log/worker.log
```

---

## Testing Jobs

```php
class SendWelcomeEmailJobTest extends TestCase
{
    public function test_email_sent_to_user()
    {
        Mail::fake();

        $user = User::factory()->create();
        SendWelcomeEmailJob::dispatch($user);

        Mail::assertSent(WelcomeEmail::class, fn($mail) => 
            $mail->hasTo($user->email)
        );
    }

    public function test_job_fails_with_invalid_email()
    {
        Mail::shouldReceive('send')
            ->andThrow(new Exception('Invalid email'));

        $job = new SendWelcomeEmailJob($user);
        $this->expectException(Exception::class);
        $job->handle();
    }

    public function test_job_is_queued()
    {
        Queue::fake();
        SendWelcomeEmailJob::dispatch($user);
        Queue::assertPushed(SendWelcomeEmailJob::class);
    }
}
```

---

## Queue Checklist

- [ ] Long-running tasks moved to jobs
- [ ] Failed jobs have retry logic
- [ ] Queues monitored for failures
- [ ] Dead-letter queue for permanent failures
- [ ] Task scheduling configured
- [ ] Worker processes daemonized
- [ ] Job timeouts appropriate
- [ ] Tests mock queue
- [ ] Monitoring for queue depth
- [ ] Alerting on job failures

---

Next: [12-CACHING-STRATEGY.md](./12-CACHING-STRATEGY.md)
