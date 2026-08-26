# 10 - Event-Driven Architecture

## Decoupled, Scalable Event Handling

---

## Domain Events

```php
// Domain/Users/Events/UserRegistered.php
namespace App\Domain\Users\Events;

use App\Domain\Users\User;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserRegistered
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly User $user,
    ) {}
}

// More domain events
class UserUpdated
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly User $user,
        public readonly array $changes,
    ) {}
}

class UserDeleted
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly string $userId,
    ) {}
}
```

---

## Event Listeners

```php
// Infrastructure/Listeners/SendWelcomeEmailListener.php
namespace App\Infrastructure\Listeners;

use App\Domain\Users\Events\UserRegistered;
use App\Infrastructure\Mail\WelcomeEmail;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmailListener implements ShouldQueue
{
    public $queue = 'emails';
    public $maxExceptions = 3;

    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email())
            ->queue(new WelcomeEmail($event->user));
    }

    public function failed(UserRegistered $event, Throwable $exception): void
    {
        Log::error('Welcome email failed', [
            'user_id' => $event->user->id(),
            'error' => $exception->getMessage(),
        ]);
    }
}

// Multiple listeners for same event
class LogUserRegistrationListener
{
    public function handle(UserRegistered $event): void
    {
        Log::info('User registered', [
            'user_id' => $event->user->id(),
            'email' => $event->user->email(),
        ]);
    }
}

class TrackUserAnalyticsListener
{
    public function handle(UserRegistered $event): void
    {
        Analytics::track('user.registered', [
            'user_id' => $event->user->id(),
        ]);
    }
}
```

---

## Event Registration

```php
// Infrastructure/Providers/EventServiceProvider.php
namespace App\Infrastructure\Providers;

use App\Domain\Users\Events\UserRegistered;
use App\Domain\Users\Events\UserUpdated;
use App\Domain\Users\Events\UserDeleted;
use App\Infrastructure\Listeners\SendWelcomeEmailListener;
use App\Infrastructure\Listeners\LogUserRegistrationListener;
use App\Infrastructure\Listeners\TrackUserAnalyticsListener;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        UserRegistered::class => [
            SendWelcomeEmailListener::class,
            LogUserRegistrationListener::class,
            TrackUserAnalyticsListener::class,
        ],
        UserUpdated::class => [
            LogUserUpdateListener::class,
        ],
        UserDeleted::class => [
            CleanupUserDataListener::class,
        ],
    ];

    public function boot(): void
    {
        // Listen to eloquent events
        User::created(fn(User $user) => 
            event(new UserRegistered($user))
        );
    }
}
```

---

## Event Broadcasting (Real-time)

```php
// Broadcasting event to connected users
class OrderShipped implements ShouldBroadcast
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public Order $order,
    ) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("order.{$this->order->user_id}"),
        ];
    }

    public function broadcastAs(): string
    {
        return 'order.shipped';
    }
}

// Listen in JavaScript
Echo.private(`order.${userId}`)
    .listen('OrderShipped', (e) => {
        console.log('Order shipped:', e.order);
        updateOrderStatus(e.order);
    });
```

---

## Event Queuing

```php
// Make listeners queued for async processing
class SendWelcomeEmailListener implements ShouldQueue
{
    // Process in background
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email())->send(new WelcomeEmail());
    }

    // Handle failures
    public function failed(UserRegistered $event, Throwable $exception): void
    {
        Log::error('Email send failed', ['error' => $exception]);
        // Retry later or notify admins
    }
}

// Dispatch events with delay
event(new UserRegistered($user));

// Dispatch to specific queue
Dispatch::to('emails')->dispatchAfterResponse(
    new SendReminderEmail($user)
);
```

---

## Webhook Events

```php
// Send events to external services
class WebhookEventListener implements ShouldQueue
{
    public function handle(UserRegistered $event): void
    {
        WebhookService::dispatch(WebhookEvent::class, [
            'event' => 'user.registered',
            'data' => [
                'user_id' => $event->user->id(),
                'email' => $event->user->email(),
                'created_at' => $event->user->created_at,
            ],
        ]);
    }
}

// Service to send webhooks
class WebhookService
{
    public static function dispatch(string $event, array $payload): void
    {
        $webhooks = Webhook::where('event', $event)->active()->get();

        foreach ($webhooks as $webhook) {
            SendWebhookJob::dispatch($webhook, $payload);
        }
    }
}

class SendWebhookJob implements ShouldQueue
{
    public function __construct(
        private Webhook $webhook,
        private array $payload,
    ) {}

    public function handle(): void
    {
        try {
            $response = Http::timeout(10)
                ->withHeaders(['X-Webhook-Signature' => $this->signature()])
                ->post($this->webhook->url, $this->payload);

            $response->throw();
        } catch (Exception $e) {
            $this->fail($e);
        }
    }

    private function signature(): string
    {
        return hash_hmac(
            'sha256',
            json_encode($this->payload),
            $this->webhook->secret
        );
    }
}
```

---

## Event Sourcing (Advanced)

```php
// Store all state changes as events
class EventSourcingService
{
    public function __construct(private EventStore $eventStore) {}

    public function recordEvent(DomainEvent $event): void
    {
        $this->eventStore->append($event);
        event($event); // Dispatch to listeners
    }

    public function getAggregate(string $aggregateId): Aggregate
    {
        $events = $this->eventStore->getEvents($aggregateId);
        return Aggregate::fromEvents($events);
    }
}

class EventStore
{
    public function append(DomainEvent $event): void
    {
        EventLog::create([
            'aggregate_id' => $event->aggregateId,
            'event_type' => get_class($event),
            'payload' => json_encode($event),
            'created_at' => now(),
        ]);
    }

    public function getEvents(string $aggregateId): array
    {
        return EventLog::where('aggregate_id', $aggregateId)
            ->orderBy('created_at')
            ->get()
            ->map(fn($log) => unserialize($log->payload))
            ->all();
    }
}
```

---

## Testing Events

```php
class UserRegistrationTest extends TestCase
{
    public function test_welcome_email_sent_on_registration()
    {
        Mail::fake();

        $user = User::factory()->create();
        event(new UserRegistered($user));

        Mail::assertSent(WelcomeEmail::class, function ($mail) use ($user) {
            return $mail->hasTo($user->email);
        });
    }

    public function test_analytics_tracked_on_registration()
    {
        Event::fake();

        $user = User::factory()->create();
        event(new UserRegistered($user));

        Event::assertDispatched(UserRegistered::class);
    }

    public function test_listener_handles_exception()
    {
        Mail::shouldReceive('send')
            ->andThrow(new Exception('SMTP error'));

        $listener = new SendWelcomeEmailListener();
        $event = new UserRegistered(User::factory()->make());

        $listener->failed($event, new Exception('SMTP error'));

        $this->assertDatabaseHas('failed_jobs', [
            // Check failed job was logged
        ]);
    }
}
```

---

## Event-Driven Checklist

- [ ] Domain events defined for key actions
- [ ] Listeners are decoupled from events
- [ ] Long-running tasks queued asynchronously
- [ ] Failed events have retry logic
- [ ] Webhooks for external integration
- [ ] Events logged for audit trail
- [ ] Broadcasting for real-time updates
- [ ] Tests verify event dispatch
- [ ] No direct coupling in listeners
- [ ] Multiple listeners per event supported

---

Next: [11-JOBS-AND-QUEUES.md](./11-JOBS-AND-QUEUES.md)
