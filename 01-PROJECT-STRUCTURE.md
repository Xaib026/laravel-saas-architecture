# 01 - Project Structure & Organization

## DDD-Based Laravel Project Structure

A scalable, maintainable project structure based on Domain-Driven Design principles, optimized for SaaS applications.

---

## Directory Structure

```
project/
├── app/
│   ├── Domain/                          # Core business logic (framework-agnostic)
│   │   ├── Users/
│   │   │   ├── User.php                # Domain entity
│   │   │   ├── UserRepository.php      # Repository interface
│   │   │   ├── UserService.php         # Domain service
│   │   │   ├── Exceptions/
│   │   │   │   ├── UserAlreadyExistsException.php
│   │   │   │   ├── InvalidEmailException.php
│   │   │   │   └── UserNotVerifiedException.php
│   │   │   └── ValueObjects/
│   │   │       ├── Email.php
│   │   │       ├── Password.php
│   │   │       └── UserId.php
│   │   ├── Subscriptions/
│   │   │   ├── Subscription.php
│   │   │   ├── SubscriptionRepository.php
│   │   │   ├── Plan.php
│   │   │   └── Events/
│   │   │       ├── SubscriptionCreated.php
│   │   │       ├── SubscriptionCancelled.php
│   │   │       └── SubscriptionExpired.php
│   │   ├── Payments/
│   │   │   ├── Payment.php
│   │   │   ├── PaymentRepository.php
│   │   │   └── PaymentProcessor.php
│   │   ├── Teams/
│   │   │   ├── Team.php
│   │   │   ├── TeamRepository.php
│   │   │   └── Role.php
│   │   └── Shared/
│   │       ├── Exceptions/
│   │       │   ├── DomainException.php
│   │       │   ├── ValidationException.php
│   │       │   └── AuthorizationException.php
│   │       ├── ValueObjects/
│   │       │   ├── Money.php
│   │       │   ├── Uuid.php
│   │       │   └── Email.php
│   │       └── Traits/
│   │           ├── HasIdentifier.php
│   │           └── IsEntity.php
│   │
│   ├── Application/                     # Use cases & orchestration
│   │   ├── UseCases/
│   │   │   ├── RegisterUserUseCase.php
│   │   │   ├── CreateSubscriptionUseCase.php
│   │   │   ├── ProcessPaymentUseCase.php
│   │   │   ├── CancelSubscriptionUseCase.php
│   │   │   └── UpdateUserProfileUseCase.php
│   │   ├── DTO/                         # Data Transfer Objects
│   │   │   ├── RegisterUserDTO.php
│   │   │   ├── CreateSubscriptionDTO.php
│   │   │   ├── UpdateUserDTO.php
│   │   │   └── FilterDTO.php
│   │   ├── Services/                    # Application services
│   │   │   ├── UserService.php
│   │   │   ├── SubscriptionService.php
│   │   │   ├── PaymentService.php
│   │   │   └── NotificationService.php
│   │   └── Events/
│   │       ├── DomainEventDispatcher.php
│   │       └── DomainEventListener.php
│   │
│   ├── Infrastructure/                  # Framework-specific implementations
│   │   ├── Repositories/
│   │   │   ├── EloquentUserRepository.php
│   │   │   ├── EloquentSubscriptionRepository.php
│   │   │   ├── EloquentPaymentRepository.php
│   │   │   └── EloquentTeamRepository.php
│   │   ├── Providers/
│   │   │   ├── AppServiceProvider.php
│   │   │   ├── RepositoryServiceProvider.php
│   │   │   ├── UseCaseServiceProvider.php
│   │   │   ├── EventServiceProvider.php
│   │   │   └── RouteServiceProvider.php
│   │   ├── Events/
│   │   │   ├── UserRegistered.php
│   │   │   ├── SubscriptionCreated.php
│   │   │   └── PaymentProcessed.php
│   │   ├── Listeners/
│   │   │   ├── SendWelcomeEmailListener.php
│   │   │   ├── SendSubscriptionConfirmationListener.php
│   │   │   ├── LogUserRegistration.php
│   │   │   └── UpdateAnalytics.php
│   │   ├── Jobs/
│   │   │   ├── SendWelcomeEmailJob.php
│   │   │   ├── ProcessPaymentJob.php
│   │   │   ├── GenerateInvoiceJob.php
│   │   │   ├── SendReminderEmailJob.php
│   │   │   └── ExportDataJob.php
│   │   ├── Mail/
│   │   │   ├── WelcomeEmail.php
│   │   │   ├── SubscriptionConfirmationEmail.php
│   │   │   ├── InvoiceEmail.php
│   │   │   └── VerifyEmailMailable.php
│   │   ├── Notifications/
│   │   │   ├── SubscriptionExpiringNotification.php
│   │   │   ├── PaymentFailedNotification.php
│   │   │   └── LargeOrderNotification.php
│   │   ├── External/
│   │   │   ├── PaymentGateways/
│   │   │   │   ├── StripePaymentGateway.php
│   │   │   │   ├── PayPalPaymentGateway.php
│   │   │   │   └── PaymentGatewayInterface.php
│   │   │   ├── Analytics/
│   │   │   │   ├── SegmentAnalytics.php
│   │   │   │   └── MixpanelAnalytics.php
│   │   │   └── EmailService/
│   │   │       ├── SendgridEmailService.php
│   │   │       └── MailgunEmailService.php
│   │   └── Caching/
│   │       ├── CachedUserRepository.php
│   │       ├── CacheKeyGenerator.php
│   │       └── CacheManager.php
│   │
│   ├── Presentation/                    # Controllers, requests, resources
│   │   ├── Http/
│   │   │   ├── Controllers/
│   │   │   │   ├── Api/
│   │   │   │   │   ├── V1/
│   │   │   │   │   │   ├── UserController.php
│   │   │   │   │   │   ├── SubscriptionController.php
│   │   │   │   │   │   ├── PaymentController.php
│   │   │   │   │   │   └── TeamController.php
│   │   │   │   │   └── V2/
│   │   │   │   │       └── (V2 controllers if needed)
│   │   │   │   └── Web/
│   │   │   │       ├── DashboardController.php
│   │   │   │       ├── AccountController.php
│   │   │   │       └── SettingsController.php
│   │   │   ├── Requests/
│   │   │   │   ├── StoreUserRequest.php
│   │   │   │   ├── UpdateUserRequest.php
│   │   │   │   ├── CreateSubscriptionRequest.php
│   │   │   │   ├── UpdateSubscriptionRequest.php
│   │   │   │   └── FilterUsersRequest.php
│   │   │   ├── Resources/
│   │   │   │   ├── UserResource.php
│   │   │   │   ├── UserCollection.php
│   │   │   │   ├── SubscriptionResource.php
│   │   │   │   ├── PaymentResource.php
│   │   │   │   ├── TeamResource.php
│   │   │   │   └── ErrorResource.php
│   │   │   └── Middleware/
│   │   │       ├── TenantMiddleware.php
│   │   │       ├── VerifyApiToken.php
│   │   │       ├── ThrottleRequests.php
│   │   │       ├── LogApiRequests.php
│   │   │       └── HandleApiErrors.php
│   │   └── Policies/
│   │       ├── UserPolicy.php
│   │       ├── SubscriptionPolicy.php
│   │       ├── TeamPolicy.php
│   │       └── PaymentPolicy.php
│   │
│   ├── Models/                          # Eloquent models (database layer)
│   │   ├── User.php
│   │   ├── Team.php
│   │   ├── Subscription.php
│   │   ├── Plan.php
│   │   ├── Payment.php
│   │   ├── Invoice.php
│   │   ├── AuditLog.php
│   │   └── FeatureFlag.php
│   │
│   └── Shared/                          # Shared utilities
│       ├── Exceptions/
│       │   ├── DomainException.php
│       │   ├── NotFoundException.php
│       │   ├── UnauthorizedException.php
│       │   ├── ValidationException.php
│       │   └── ConflictException.php
│       ├── ValueObjects/
│       │   ├── Email.php
│       │   ├── Money.php
│       │   ├── Uuid.php
│       │   ├── BillingCycle.php
│       │   └── SubscriptionStatus.php
│       ├── Traits/
│       │   ├── HasUuid.php
│       │   ├── BelongsToTenant.php
│       │   ├── HasTimestamps.php
│       │   └── IsAuditable.php
│       ├── Enums/
│       │   ├── UserRole.php
│       │   ├── SubscriptionStatus.php
│       │   ├── PaymentStatus.php
│       │   └── BillingCycle.php
│       └── Helpers/
│           ├── UuidHelper.php
│           ├── MoneyHelper.php
│           └── DateHelper.php
│
├── tests/
│   ├── Unit/
│   │   ├── Domain/
│   │   │   ├── Users/
│   │   │   │   ├── UserTest.php
│   │   │   │   └── EmailTest.php
│   │   │   └── Subscriptions/
│   │   │       ├── SubscriptionTest.php
│   │   │       └── PlanTest.php
│   │   ├── Application/
│   │   │   ├── UseCases/
│   │   │   │   ├── RegisterUserUseCaseTest.php
│   │   │   │   └── CreateSubscriptionUseCaseTest.php
│   │   │   └── Services/
│   │   │       └── UserServiceTest.php
│   │   └── Infrastructure/
│   │       ├── Repositories/
│   │       │   ├── EloquentUserRepositoryTest.php
│   │       │   └── CachedUserRepositoryTest.php
│   │       └── External/
│   │           └── StripePaymentGatewayTest.php
│   ├── Feature/
│   │   ├── Api/
│   │   │   ├── V1/
│   │   │   │   ├── UserApiTest.php
│   │   │   │   ├── SubscriptionApiTest.php
│   │   │   │   └── PaymentApiTest.php
│   │   │   └── V2/
│   │   │       └── (V2 tests if needed)
│   │   └── Web/
│   │       ├── AuthenticationTest.php
│   │       ├── DashboardTest.php
│   │       └── SettingsTest.php
│   ├── Integration/
│   │   ├── SubscriptionWorkflowTest.php
│   │   ├── PaymentProcessingTest.php
│   │   └── AuthenticationFlowTest.php
│   └── TestCase.php
│
├── database/
│   ├── migrations/
│   │   ├── 0001_01_01_000000_create_users_table.php
│   │   ├── 0001_01_01_000001_create_teams_table.php
│   │   ├── 0001_01_01_000002_create_subscriptions_table.php
│   │   ├── 0001_01_01_000003_create_payments_table.php
│   │   ├── 0001_01_01_000004_create_invoices_table.php
│   │   └── 0001_01_01_000005_create_audit_logs_table.php
│   ├── seeders/
│   │   ├── DatabaseSeeder.php
│   │   ├── UserSeeder.php
│   │   ├── TeamSeeder.php
│   │   ├── PlanSeeder.php
│   │   └── FeatureFlagSeeder.php
│   └── factories/
│       ├── UserFactory.php
│       ├── TeamFactory.php
│       ├── SubscriptionFactory.php
│       ├── PaymentFactory.php
│       └── InvoiceFactory.php
│
├── routes/
│   ├── api.php                          # API v1 routes
│   ├── api-v2.php                       # API v2 routes (if needed)
│   ├── web.php                          # Web application routes
│   ├── admin.php                        # Admin panel routes
│   ├── auth.php                         # Authentication routes
│   └── health.php                       # Health check routes
│
├── config/
│   ├── app.php
│   ├── database.php
│   ├── queue.php
│   ├── cache.php
│   ├── mail.php
│   ├── services.php                     # Third-party service configs
│   ├── subscription.php                 # SaaS subscription config
│   ├── audit.php                        # Audit logging config
│   ├── feature-flags.php                # Feature flags config
│   └── api.php                          # API configuration
│
├── resources/
│   ├── views/
│   │   ├── layouts/
│   │   │   ├── app.blade.php
│   │   │   ├── auth.blade.php
│   │   │   └── admin.blade.php
│   │   ├── pages/
│   │   │   ├── dashboard.blade.php
│   │   │   ├── subscription.blade.php
│   │   │   └── settings.blade.php
│   │   ├── auth/
│   │   │   ├── login.blade.php
│   │   │   ├── register.blade.php
│   │   │   └── forgot-password.blade.php
│   │   ├── components/
│   │   │   ├── navbar.blade.php
│   │   │   ├── sidebar.blade.php
│   │   │   └── alerts.blade.php
│   │   └── emails/
│   │       ├── welcome.blade.php
│   │       ├── subscription-confirmation.blade.php
│   │       └── invoice.blade.php
│   └── css/
│   └── js/
│
├── storage/
│   ├── app/
│   │   ├── uploads/
│   │   └── exports/
│   ├── logs/
│   ├── cache/
│   └── framework/
│
├── bootstrap/
│   └── app.php
│
├── .env.example
├── .env.local
├── .gitignore
├── .github/
│   └── workflows/
│       ├── tests.yml
│       ├── lint.yml
│       └── deploy.yml
├── docker-compose.yml                   # Local development
├── Dockerfile                            # Production container
├── phpunit.xml                          # Testing configuration
├── phpstan.neon                         # Static analysis
├── .php-cs-fixer.php                   # Code formatting
├── composer.json
├── composer.lock
└── README.md
```

---

## Directory Responsibilities

### Domain/
**Pure business logic with NO framework dependencies**
- Define entities, value objects, services
- Repository interfaces (not implementations)
- Domain exceptions and events
- Can be tested without Laravel
- Language and framework agnostic

### Application/
**Orchestration layer that uses Domain**
- Use cases (commands that perform business operations)
- DTOs for input/output
- Application services that coordinate domain services
- Maps domain events to infrastructure

### Infrastructure/
**Framework & third-party implementations**
- Repository implementations (Eloquent)
- Job classes for background work
- Event listeners
- External service integrations
- Caching implementations

### Presentation/
**User interface layer**
- HTTP controllers
- Request validation (form requests)
- Resource serialization
- Authorization policies
- HTTP middleware

### Models/
**Eloquent models ONLY**
- Database access
- Relationships
- Casts and accessors
- Scopes
- NO business logic

---

## Key Principles

### 1. Separation of Concerns
```
Presentation (Controllers) 
    ↓
Application (Use Cases) 
    ↓
Domain (Business Logic) 
    ↓
Infrastructure (Database, APIs) 
```

### 2. Dependency Flow
- Presentation depends on Application
- Application depends on Domain
- Infrastructure depends on Domain (via interfaces)
- Domain depends on NOTHING

### 3. Every Layer Can Be Tested Independently
- Domain: Pure unit tests
- Application: Unit tests + mocked infrastructure
- Infrastructure: Integration tests
- Presentation: Feature tests

### 4. Easy to Replace Components
- Switch database: Change EloquentUserRepository
- Change payment gateway: New StripePaymentGateway
- Add caching: Wrap repository in CachedUserRepository
- No changes needed to Domain or Application

---

## File Naming Conventions

| Type | Naming | Example |
|------|--------|----------|
| Controllers | Singular + Controller | `UserController` |
| Models | Singular | `User` |
| Migrations | Timestamp + Operation | `2024_01_01_000000_create_users_table` |
| Seeders | Plural + Seeder | `UserSeeder` |
| Factories | Singular + Factory | `UserFactory` |
| Requests | Singular + Request/Action | `StoreUserRequest` |
| Resources | Singular + Resource | `UserResource` |
| Policies | Singular + Policy | `UserPolicy` |
| Events | Past tense | `UserRegistered` |
| Listeners | Action + Listener | `SendWelcomeEmailListener` |
| Jobs | Action + Job | `ProcessPaymentJob` |
| Middleware | Description + Middleware | `ThrottleRequests` |
| Domain Services | Domain + Service | `UserService` |
| Use Cases | Action + UseCase | `RegisterUserUseCase` |
| Repositories | Plural + Repository | `UserRepository` |
| Value Objects | Singular | `Email` |

---

## Configuration Files Organization

```php
// config/services.php - Third-party services
return [
    'stripe' => [
        'key' => env('STRIPE_SECRET_KEY'),
        'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
    ],
    'sendgrid' => [
        'api_key' => env('SENDGRID_API_KEY'),
    ],
];

// config/subscription.php - SaaS-specific
return [
    'plans' => [
        'basic' => [
            'name' => 'Basic',
            'price' => 2999,
            'features' => ['feature_1', 'feature_2'],
        ],
    ],
];

// config/audit.php - Audit logging
return [
    'enabled' => true,
    'log_model_changes' => true,
    'tracked_attributes' => ['email', 'status', 'role'],
];
```

---

## Next Steps

1. **Read**: [02-DESIGN-PATTERNS.md](./02-DESIGN-PATTERNS.md)
2. **Implement**: Start with your domain entities
3. **Scaffold**: Use Laravel generators, then move files to correct folders
4. **Test**: Add tests as you build each layer
5. **Reference**: Come back here when adding new features

---

## Common Mistakes to Avoid

❌ **Mixing layers**: Business logic in controllers
❌ **Dependencies pointing up**: Domain depending on Infrastructure
❌ **Skipping interfaces**: Using concrete classes instead of contracts
❌ **Models with logic**: Models should only map to database
❌ **Duplicate queries**: Not using repositories
❌ **God services**: Services that do too many things
❌ **No type hints**: Makes refactoring dangerous

---

Next: [02-DESIGN-PATTERNS.md](./02-DESIGN-PATTERNS.md)
