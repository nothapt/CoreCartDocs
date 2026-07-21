# Contributing to CoreCart

Thank you for helping improve CoreCart.

Contributions may include bug fixes, tests, documentation, refactoring, security improvements, and new features.

## Before You Start

Before opening a pull request:

1. Search existing issues and pull requests.
2. Keep the change focused on one problem.
3. Avoid unrelated formatting or refactoring.
4. Add or update tests when behavior changes.
5. Update the documentation when public behavior changes.

For larger changes, open an issue first and describe the proposed behavior.

## Development Setup

Fork the repository and clone your fork:

```bash
git clone https://github.com/YOUR_USERNAME/CoreCart.git
cd CoreCart
```

Install dependencies:

```bash
composer install
```

Create a branch:

```bash
git checkout -b fix/short-description
```

Use a clear branch prefix:

- `fix/` for bug fixes
- `feat/` for new features
- `docs/` for documentation
- `refactor/` for internal improvements
- `test/` for tests

## Required Checks

Run the test suite:

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

Check PHP syntax when necessary:

```bash
find account admin cart catalog checkout install system   -type f -name '*.php' -print0 | xargs -0 -n1 php -l
```

A pull request should not knowingly introduce failing tests, PHPStan errors, syntax errors, or broken installation steps.

## Coding Rules

CoreCart uses strict PHP code:

```php
<?php
declare(strict_types=1);
```

Follow these architectural rules:

- Controllers accept a `Request` and return a `Response`.
- Controllers must not read PHP superglobals directly.
- Controllers must not execute SQL.
- Services contain business logic.
- Repositories contain persistence logic.
- Middleware contains reusable HTTP concerns.
- DTOs represent validated input data.
- Entities represent domain data.
- State-changing routes must use the required security middleware.
- Database operations that must succeed together must use a transaction.

Use typed properties, constructor injection, explicit return types, and prepared SQL statements.

## Tests

Place tests in the matching suite:

```text
tests/
├── Unit/
├── Integration/
└── Feature/
```

Use unit tests for isolated behavior.

Use integration tests for repositories, transactions, database constraints, and service behavior with real infrastructure.

Use feature tests for complete HTTP scenarios such as authentication, cart operations, checkout, CSRF handling, and authorization.

A bug fix should normally include a test that fails before the fix and passes after it.

## Commit Messages

Use short, descriptive commit messages:

```text
fix: preserve customer cart after login
feat: add order status transition validation
docs: document local installation
test: cover checkout rollback
refactor: inject session into auth middleware
```

Do not combine unrelated changes in one commit when they can be reviewed separately.

## Pull Requests

A pull request should explain:

- what changed;
- why the change is needed;
- how it was tested;
- whether configuration, database schema, routes, or public responses changed.

Keep pull requests small enough to review.

Do not include generated files, local logs, `.env`, credentials, database dumps, IDE settings, or the `vendor` directory.

## Documentation Contributions

Documentation is maintained in the `CoreCartDocs` repository.

To suggest a documentation change:

1. Open the page on the documentation website.
2. Select **Edit this page on GitHub**.
3. Edit the Markdown file.
4. Submit the change through a GitHub pull request.

Documentation should describe actual supported behavior. Avoid documenting unfinished or experimental behavior as stable.

## Security Issues

Do not publish exploitable security vulnerabilities in a public issue.

Provide a minimal description to the project maintainer and avoid including credentials, real customer data, or production secrets.

## License

By submitting a contribution, you agree that it may be distributed under the same license as CoreCart.
