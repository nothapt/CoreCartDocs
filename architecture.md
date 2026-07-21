# Architecture

CoreCart is a modular PHP e-commerce application built around explicit HTTP, application, domain, and persistence layers.

The main request flow is:

```text
HTTP Request
→ Router
→ Middleware
→ Controller
→ Service
→ Repository
→ Database
→ Response
```

Each layer has a limited responsibility. Code should not bypass layers without a clear reason.

## Entry Points

CoreCart has separate application entry points for the storefront and administration area.

An entry point is responsible for:

1. defining application paths;
2. loading Composer;
3. loading environment configuration;
4. creating the dependency injection container;
5. registering services and middleware;
6. registering route providers;
7. creating the `Request`;
8. dispatching it through the `Router`;
9. sending the returned `Response`.

Business logic must not be placed in an entry point.

## Request

`CoreCart\System\Engine\Request` represents an incoming HTTP request.

Only `Request::fromGlobals()` may read PHP request superglobals. Application code should use Request methods instead:

```php
$request->getMethod();
$request->getPath();
$request->getQueryParam('page');
$request->getInput('email');
$request->getHeader('Accept');
$request->getCookie('name');
$request->getUploadedFile('image');
$request->getUser();
$request->getUserId();
```

Request objects are treated as immutable. Methods such as `withUser()` and `withBody()` return a new instance.

## Response

Controllers and middleware return a `Response`.

Available response types include:

- `Response` for general HTTP responses;
- `JsonResponse` for JSON APIs;
- `RedirectResponse` for redirects.

Example:

```php
return JsonResponse::success(
    ['product_id' => $productId],
    'Product created',
    201
);
```

Response objects own the status code, headers, cookies, and body.

## Router

The Router maps a normalized path to:

- a controller class;
- a public controller method;
- allowed HTTP methods;
- middleware classes.

Example route:

```php
$router->addRoute(
    'catalog/product',
    ProductController::class,
    'index',
    [SecurityHeaders::class],
    ['GET']
);
```

Route registration is grouped into route provider classes such as:

- `CatalogRouteProvider`
- `CartRouteProvider`
- `CheckoutRouteProvider`
- `AccountRouteProvider`
- `AdminRouteProvider`
- `ApiRouteProvider`
- `HealthRouteProvider`

Route providers must only register routes. They should not execute requests or contain business logic.

## Middleware

Middleware handles reusable HTTP concerns before or after a controller.

The middleware contract is:

```php
public function handle(
    Request $request,
    callable $next
): Response;
```

A middleware may:

- reject the request and return a response;
- create a modified Request and pass it forward;
- call the next layer and modify the returned Response.

Example:

```php
public function handle(Request $request, callable $next): Response
{
    $response = $next($request);
    $response->setHeader('X-Content-Type-Options', 'nosniff');

    return $response;
}
```

Typical middleware concerns include:

- authentication;
- CSRF validation;
- request size and content type checks;
- security headers;
- rate limiting;
- authenticated user context.

Middleware must pass the current Request to the next layer.

## Controllers

Controllers translate HTTP input into application calls.

A controller should:

- read values from `Request`;
- create DTOs;
- call a service;
- select an appropriate HTTP response;
- convert expected failures into suitable status codes.

A controller must not:

- read `$_GET`, `$_POST`, `$_SERVER`, `$_COOKIE`, or `$_SESSION`;
- execute SQL;
- implement transactions;
- contain complex business rules.

Controllers receive the application container through their constructor.

## Services

Services implement business use cases.

Examples include:

- adding a product to a cart;
- registering a customer;
- changing an order status;
- creating an order;
- merging a guest cart;
- calculating dashboard information.

Services coordinate repositories and enforce business rules. They should not depend on HTTP-specific objects such as `Request` or `Response`.

Operations that must be atomic should be executed inside a database transaction.

## Repositories

Repositories contain database access.

A repository is responsible for:

- prepared SQL queries;
- mapping rows to entities;
- inserts, updates, and deletes;
- persistence-specific filtering and pagination.

Repositories should not return HTTP responses or read request data.

All user-provided values must be passed as prepared statement parameters. Dynamic SQL identifiers such as sort columns must be selected from an explicit allowlist.

## DTOs

Data Transfer Objects represent input for a specific operation.

Examples:

- `LoginDTO`
- `RegisterDTO`
- `CartAddDTO`
- `OrderCreateDTO`
- `ProductCreateDTO`
- `AddressDTO`

DTOs normalize external field names and provide typed properties. Validation may be performed before or inside the related service, depending on the use case.

DTOs should not access the database or application container.

## Entities

Entities represent stored domain data.

Examples include:

- `Product`
- `Category`
- `Customer`
- `Address`
- `CartItem`
- `Order`
- `OrderItem`
- `AdminUser`

Entities should contain typed data and simple domain-related behavior. They should not execute SQL or produce HTTP responses.

## Infrastructure

Infrastructure classes provide technical capabilities shared by the application.

Current examples include:

- `Database`
- `SessionInterface`
- `Session`
- `AuthenticatedUser`
- `OrderStatus`
- `RateLimiter`

Application code should depend on interfaces where an abstraction is useful, especially for sessions and other stateful infrastructure.

## Dependency Injection

`Container` stores service factories and shared service instances.

Example registration:

```php
$container->set(
    ProductRepository::class,
    fn (Container $container) => new ProductRepository(
        $container->get(Database::class)
    )
);
```

Constructor injection is preferred. Classes should not locate dependencies through globals or create database connections internally.

## Transactions

Use a transaction when partial completion would leave invalid data.

Typical transactional operations include:

- creating an order and its items;
- decreasing product stock;
- clearing the ordered cart;
- creating a product and its descriptions;
- changing a default address;
- updating related records that must remain consistent.

The transaction boundary belongs in the service or repository that coordinates the complete persistence operation.

## Error Handling

Expected errors should be represented using meaningful exceptions and converted by controllers into suitable status codes.

Typical responses:

| Situation | Status |
|---|---:|
| Invalid input | `400` or `422` |
| Authentication required | `401` |
| Access denied | `403` |
| Resource not found | `404` |
| Method not allowed | `405` |
| Conflict | `409` |
| Request too large | `413` |
| Unsupported content type | `415` |
| Rate limited | `429` |
| Unexpected failure | `500` |

Unexpected errors must be logged with a request ID. Production responses must not expose stack traces, SQL errors, credentials, or internal paths.
