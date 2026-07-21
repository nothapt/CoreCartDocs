# Development

This page describes the recommended workflow for developing CoreCart.

## Requirements

Development requires:

- PHP 8.4 or newer;
- Composer 2;
- MySQL 8;
- required PHP extensions;
- Git.

Install dependencies:

```bash
composer install
```

## Local Server

Start the PHP development server:

```bash
php -S localhost:8000 system/engine/router_builtin.php
```

On Windows, the repository may provide:

```bat
run.bat
```

The built-in server is intended for local development, not production.

## Project Structure

```text
CoreCart/
├── account/           Customer account controllers
├── admin/             Administration entry point and controllers
├── cart/              Cart controllers
├── catalog/           Catalog controllers
├── checkout/          Checkout controllers
├── docker/            Docker development configuration
├── install/           Database schema and installation resources
├── storage/           Logs, cache, and installation state
├── system/
│   ├── dto/           Input DTOs
│   ├── engine/        HTTP and application infrastructure
│   ├── entity/        Domain entities
│   ├── infrastructure/Shared infrastructure abstractions
│   ├── repository/    Database access
│   ├── route/         Route providers
│   └── service/       Business services
├── tests/
│   ├── Unit/
│   ├── Integration/
│   └── Feature/
├── cli.php
├── composer.json
├── index.php
├── phpstan.neon
└── phpunit.xml
```

## Composer Commands

Run all tests:

```bash
composer test
```

Run static analysis:

```bash
composer analyse
```

Validate Composer metadata:

```bash
composer validate --strict
```

Install locked dependencies:

```bash
composer install
```

Update dependencies intentionally:

```bash
composer update
```

Commit the resulting `composer.lock` after reviewing and testing the changes.

## Coding Style

Every PHP file should begin with:

```php
<?php
declare(strict_types=1);
```

Use:

- PSR-4 namespaces;
- typed properties;
- constructor injection;
- explicit parameter and return types;
- strict comparisons;
- prepared statements;
- small focused methods;
- immutable request transformations.

Avoid:

- application logic in entry points;
- SQL in controllers;
- HTTP responses in services;
- direct superglobal access outside request/session infrastructure;
- global service locators;
- silent fallback from invalid input;
- floating-point arithmetic for money.

## Adding a Route

Add routes inside the relevant route provider.

Example:

```php
$router->addRoute(
    'catalog/product/recommended',
    ProductController::class,
    'recommended',
    [SecurityHeaders::class],
    ['GET']
);
```

For multiple routes:

```php
$router->addRoutes([
    'catalog/product/recommended' => [
        'controller' => ProductController::class,
        'method' => 'recommended',
        'middleware' => [SecurityHeaders::class],
        'methods' => ['GET'],
    ],
]);
```

Choose middleware based on the route behavior:

- public read route;
- optional customer context;
- required customer authentication;
- administrator authentication;
- CSRF protection;
- request content validation;
- security headers.

State-changing routes must not be registered without the required protection.

## Adding a Controller

A controller accepts a `Request` and returns a `Response`.

Example:

```php
final class ExampleController
{
    public function __construct(
        private Container $container
    ) {}

    public function index(Request $request): Response
    {
        $service = $this->container->get(ExampleService::class);
        $result = $service->getData();

        return JsonResponse::success($result);
    }
}
```

Controllers should stay thin.

Use DTOs for structured input:

```php
$dto = ExampleDTO::fromArray($request->getBody());
```

Convert expected exceptions into appropriate HTTP responses.

## Adding a Service

Services contain business rules.

Example:

```php
final class ExampleService
{
    public function __construct(
        private ExampleRepository $repository
    ) {}

    public function create(ExampleDTO $dto): int
    {
        if (trim($dto->name) === '') {
            throw new InvalidArgumentException('Name is required');
        }

        return $this->repository->create($dto);
    }
}
```

A service may coordinate several repositories.

Use a transaction when multiple database operations form one business action.

## Adding a Repository

Repositories contain persistence logic.

Example:

```php
final class ExampleRepository
{
    public function __construct(
        private Database $database
    ) {}

    public function findById(int $id): ?Example
    {
        $rows = $this->database->query(
            'SELECT example_id, name FROM cc_example WHERE example_id = :id',
            ['id' => $id]
        );

        return $rows === []
            ? null
            : Example::fromRow($rows[0]);
    }
}
```

Never concatenate user-provided values into SQL.

For dynamic ordering, map accepted values to known columns:

```php
$column = match ($sort) {
    'name' => 'name',
    'date' => 'date_added',
    default => 'example_id',
};
```

## Adding a DTO

DTOs normalize external input:

```php
final class ExampleDTO
{
    public function __construct(
        public readonly string $name,
        public readonly int $status = 1,
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            name: trim((string) ($data['name'] ?? '')),
            status: (int) ($data['status'] ?? 1),
        );
    }
}
```

Place optional constructor parameters after required parameters.

Do not silently convert invalid enum or identifier values into valid defaults.

## Adding an Entity

Entities map stored domain data:

```php
final class Example
{
    public function __construct(
        public readonly int $id,
        public readonly string $name,
    ) {}

    public static function fromRow(array $row): self
    {
        return new self(
            id: (int) $row['example_id'],
            name: (string) $row['name'],
        );
    }
}
```

Entities may contain simple domain behavior, but should not access HTTP or database infrastructure.

## Dependency Registration

Register new dependencies in the application bootstrap:

```php
$container->set(
    ExampleRepository::class,
    fn (Container $container) => new ExampleRepository(
        $container->get(Database::class)
    )
);

$container->set(
    ExampleService::class,
    fn (Container $container) => new ExampleService(
        $container->get(ExampleRepository::class)
    )
);
```

Constructor injection is preferred over globals.

## Testing

### Unit Tests

Unit tests verify isolated behavior without real infrastructure.

Good candidates:

- DTO normalization;
- value objects;
- validators;
- status transitions;
- response formatting;
- request transformations.

### Integration Tests

Integration tests use real infrastructure such as MySQL.

Good candidates:

- repository queries;
- schema constraints;
- transactions;
- cart merging;
- order creation;
- stock locking;
- revenue queries;
- address ownership.

### Feature Tests

Feature tests verify complete HTTP scenarios.

Required high-value scenarios include:

- customer registration and login;
- session regeneration;
- guest cart to customer cart merge;
- cart ownership;
- CSRF token lifecycle;
- guest checkout;
- customer checkout;
- checkout rollback;
- customer order authorization;
- administrator authentication;
- order status transitions;
- readiness and liveness endpoints.

## Database Tests

Integration and feature tests should use a dedicated test database.

Before the suite:

1. create the database;
2. import `install/database.sql` or run migrations;
3. insert only required fixtures.

Each test should be isolated through transactions, cleanup, or database reset.

Never run automated tests against a production database.

## Static Analysis

PHPStan is configured through:

```text
phpstan.neon
```

Run:

```bash
vendor/bin/phpstan analyse --no-progress
```

Do not add an ignored error only to make CI green.

First determine whether the report identifies:

- a real type mismatch;
- unreachable code;
- a wrong namespace;
- a missing import;
- an invalid method contract;
- unused dependencies;
- an inaccurate PHPDoc type.

Ignored patterns should be specific, justified, and periodically reviewed.

## Continuous Integration

The CI pipeline should verify:

1. Composer configuration;
2. PHP syntax;
3. static analysis;
4. unit tests;
5. integration tests;
6. feature tests;
7. application smoke checks;
8. Docker build when Docker is supported.

A smoke test should validate response content, not only status codes.

Tests that require MySQL must import the schema before execution.

## Debugging

Useful checks:

```bash
composer validate --strict
composer analyse
composer test
php -l path/to/file.php
```

Inspect logs under:

```text
storage/logs/
```

Do not enable detailed errors in production.

Use request IDs to correlate client errors with server logs.

## Documentation

Update documentation when changing:

- requirements;
- environment variables;
- installation steps;
- routes;
- request or response formats;
- authentication behavior;
- database schema;
- public extension points.

Documentation changes are maintained in the `CoreCartDocs` repository.

## Pull Request Checklist

Before submitting a pull request:

- the change has one clear purpose;
- tests cover changed behavior;
- Composer validation passes;
- PHPStan passes;
- relevant integration tests pass;
- no credentials or local files are included;
- documentation is updated;
- database changes include an upgrade path;
- public behavior is described in the pull request.
