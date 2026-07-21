# Security

Security is a cross-cutting requirement in CoreCart. Authentication, authorization, sessions, CSRF protection, input validation, database safety, and deployment configuration must work together.

This page describes the security contract expected from CoreCart.

## Production Configuration

Production installations should:

- use HTTPS;
- set `APP_DEBUG=false`;
- use unique database credentials;
- use a strong administrator password;
- deny public access to `.env`, `vendor`, `storage`, `install`, source code, logs, and CI files;
- expose only the public web entry point;
- keep PHP and Composer dependencies updated;
- store secrets outside the repository;
- restrict database network access;
- configure backups and monitoring.

Never commit `.env`, credentials, session data, logs, private keys, or production database dumps.

## Password Storage

Passwords must never be stored as plaintext.

CoreCart uses PHP password hashing functions:

```php
password_hash($password, PASSWORD_BCRYPT);
password_verify($password, $hash);
```

Authentication responses must not reveal whether a username or email exists.

Password-changing operations should validate the current password or use a separate verified recovery flow.

## Sessions

Session access is provided through `SessionInterface`.

Controllers should not access `$_SESSION` directly.

Session configuration should include:

- cookie-only sessions;
- strict session mode;
- `HttpOnly`;
- `Secure` in production;
- `SameSite=Lax` or stricter;
- session identifier regeneration after login;
- expiration and idle timeout where appropriate.

Session identifiers must be regenerated after authentication to prevent session fixation.

Customer and administrator state should be cleared independently unless a complete session invalidation is explicitly required.

## Customer Authentication

Protected customer routes must verify:

1. a customer ID exists in the session;
2. the customer still exists;
3. the customer account is active;
4. an `AuthenticatedUser` is attached to the Request.

Controllers should use:

```php
$request->getUser();
$request->getUserId();
```

Customer login endpoints should be rate limited.

Guest cart data should be merged safely into the authenticated customer cart after login or registration.

## Administrator Authentication

Administrator routes require a valid active administrator session.

Recommended controls include:

- rate limiting;
- absolute session lifetime;
- idle timeout;
- session regeneration after login;
- CSRF protection for state-changing actions;
- security headers;
- optional IP or device-change detection;
- audit logging for sensitive actions.

Administrator login, logout, product changes, category changes, customer administration, and order status changes should be covered by feature tests.

## Authorization

Authentication confirms identity. Authorization confirms permission.

Every resource lookup must enforce ownership or role requirements.

Examples:

- a customer may only view their own orders;
- a customer may only update their own addresses;
- a customer or guest may only modify their own cart rows;
- only administrators may use administration endpoints;
- an order success page must be tied to the current session or customer.

For private resources, returning `404 Not Found` instead of `403 Forbidden` may reduce resource enumeration.

## CSRF Protection

All cookie-authenticated state-changing requests require CSRF protection.

Protected methods include:

```text
POST
PUT
PATCH
DELETE
```

A token may be sent in the request body:

```json
{
  "_csrf_token": "TOKEN"
}
```

or in a header:

```http
X-CSRF-Token: TOKEN
```

Tokens must be:

- generated with a cryptographically secure source;
- stored in the current session;
- compared with `hash_equals`;
- available to clients through a documented page response or token endpoint.

CSRF protection is required for:

- customer login and registration;
- customer logout;
- cart mutations;
- checkout confirmation;
- password and address changes;
- administrator login and logout;
- administrator mutations.

## Rate Limiting

Login endpoints should limit repeated failed attempts.

A rate limiter should consider:

- source IP;
- normalized login identifier;
- time window;
- maximum attempts;
- successful-login reset behavior;
- old record cleanup;
- protection against intentional account lockout.

Rate-limit logs must sanitize user-controlled values before writing them.

A `429 Too Many Requests` response should not reveal unnecessary account information.

## Security Headers

Responses should include suitable security headers:

```text
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Content-Security-Policy: ...
```

`Strict-Transport-Security` should only be emitted when the application is served over HTTPS.

`X-XSS-Protection` is obsolete in modern browsers and should not be treated as a replacement for output escaping and Content Security Policy.

The Content Security Policy should be adjusted when frontend assets, inline scripts, or external resources are introduced.

## Input Validation

Input must be validated before business operations.

Validation should cover:

- required fields;
- string length;
- email format;
- numeric ranges;
- allowed enum values;
- identifiers greater than zero;
- product stock;
- address ownership;
- file size and type;
- content type;
- request body size.

Invalid JSON should return a clear `400 Bad Request`.

Do not silently convert invalid enum values into valid defaults.

## Database Security

Database operations must use prepared statements.

Example:

```php
$db->query(
    'SELECT * FROM cc_customer WHERE customer_id = :id',
    ['id' => $customerId]
);
```

Do not concatenate user input into SQL.

Dynamic identifiers such as sort columns must use an explicit allowlist.

Operations that must remain consistent should use transactions, including checkout, stock updates, related entity updates, and default-address changes.

Use row locking where concurrent updates can create overselling or inconsistent state.

## Order Security

Order creation should:

- calculate totals on the server;
- read current product prices from the database;
- validate current stock;
- lock relevant rows;
- create order and item snapshots;
- decrease stock atomically;
- clear the cart inside the same transaction.

Do not trust client-provided totals, prices, customer IDs, ownership data, or order statuses.

Order snapshots should preserve the customer contact and shipping data used at purchase time.

Cancelled orders must not be included in revenue calculations.

## File and Path Safety

Any feature that reads file paths from configuration or extension files must:

- reject absolute paths;
- reject `..` path traversal;
- resolve paths with `realpath`;
- verify that the resolved path remains inside an allowed directory;
- write only to approved cache locations.

Uploaded files should be renamed, validated by content, and stored outside executable source directories.

## Error Handling and Logging

Production responses must not expose:

- stack traces;
- SQL errors;
- credentials;
- filesystem paths;
- environment values;
- raw dependency exceptions.

Unexpected errors should include a request ID in the log and a generic response to the client.

Logs must not contain plaintext passwords, session identifiers, CSRF tokens, access tokens, payment details, or unnecessary personal data.

User-controlled values should be sanitized before being written to logs to prevent log injection.

## Dependencies

Use the committed `composer.lock` for reproducible installations.

Recommended checks include:

```bash
composer validate --strict
composer audit
composer analyse
composer test
```

Dependency updates should be reviewed and tested before production deployment.

## Security Reporting

Do not publish detailed exploitable vulnerabilities in public issues.

A security report should include:

- affected component;
- impact;
- reproduction conditions;
- suggested mitigation;
- whether sensitive data was accessed.

Do not include real credentials or customer data in reports.
