# Troubleshooting

This page covers common installation and local startup problems.

## `composer: command not found`

Composer is not installed or is not available in `PATH`.

Check:

```bash
composer --version
```

Install Composer, restart the terminal, and run:

```bash
composer install
```

## `vendor/autoload.php` not found

PHP dependencies have not been installed in the project directory.

Run:

```bash
composer install
```

With Docker:

```bash
docker compose -f docker/docker-compose.yml run --rm php composer install
```

The Docker project mount can hide dependencies created while building the image, so installing them into the mounted project directory may be required.

## Unsupported PHP version

Check the CLI version:

```bash
php --version
```

CoreCart requires PHP 8.4 or later. Also verify the PHP version used by PHP-FPM or Apache; it may differ from the CLI version.

## `could not find driver`

The PDO MySQL extension is missing.

Check:

```bash
php -m | grep -E 'PDO|pdo_mysql'
```

On Debian or Ubuntu:

```bash
sudo apt install php8.4-mysql
```

Restart PHP-FPM or Apache after installing an extension.

## `Call to undefined function bcadd()` or `bcmul()`

The `bcmath` extension is missing. It is required for order total calculations.

On Debian or Ubuntu:

```bash
sudo apt install php8.4-bcmath
```

For Docker, add `bcmath` to `docker-php-ext-install` in the PHP image and rebuild it:

```bash
docker compose -f docker/docker-compose.yml build --no-cache php
docker compose -f docker/docker-compose.yml up -d
```

## Database access denied

Typical message:

```text
SQLSTATE[HY000] [1045] Access denied for user
```

Verify:

- `DB_HOST`;
- `DB_USER`;
- `DB_PASS`;
- whether the database user may connect from the current host;
- whether MySQL is running.

Test the credentials directly:

```bash
mysql -h 127.0.0.1 -u root -p
```

For Docker, use `DB_HOST=db` inside the PHP container, not `127.0.0.1`.

## Unknown database

Typical message:

```text
SQLSTATE[HY000] [1049] Unknown database 'corecart'
```

Run the CoreCart installer, which creates the database and imports the schema:

```bash
php cli.php install \
  --db_user=root \
  --db_pass=your-database-password \
  --db_name=corecart \
  --admin_pass=use-a-strong-password
```

## `CoreCart is already installed`

The file below exists:

```text
storage/installed.lock
```

Do not remove it from an active installation. For a clean local reinstall, back up required data, remove the lock file, reset the database, and run the installer again.

## Administrator login always fails

Do not rely on the placeholder administrator record in `install/database.sql`. The SQL file does not contain a usable default password hash.

Create the administrator through the CLI installer:

```bash
php cli.php install \
  --db_user=root \
  --db_pass=your-database-password \
  --db_name=corecart \
  --admin_user=admin \
  --admin_email=admin@example.com \
  --admin_pass=use-a-strong-password
```

## Login does not persist on local HTTP

Symptoms include:

- login succeeds but the next request is unauthenticated;
- CSRF tokens change unexpectedly;
- the session cookie is not sent by the browser.

Start the local server with `APP_DEBUG=true` in the process environment:

```bash
APP_DEBUG=true php -S localhost:8000 system/engine/router_builtin.php
```

Windows Command Prompt:

```bat
set APP_DEBUG=true
run.bat
```

Use HTTPS when `APP_DEBUG` is disabled.

## `Invalid CSRF token`

Check that:

- the same browser session is used for the form and submission;
- cookies are enabled;
- the request contains `_csrf_token` or the `X-CSRF-Token` header;
- the session cookie is not being blocked because of HTTP/HTTPS settings.

A failed or replaced session invalidates its CSRF token.

## Routes return 404 with the PHP development server

Start PHP with the included router script:

```bash
php -S localhost:8000 system/engine/router_builtin.php
```

Starting only with `php -S localhost:8000` does not forward dynamic URLs to `index.php`.

For Apache, ensure URL rewriting is enabled and `.htaccess` is allowed. For Nginx, configure missing files to be forwarded to the CoreCart entry point.

## Port already in use

Typical message:

```text
Failed to listen on localhost:8000
```

Use another port:

```bash
APP_DEBUG=true php -S localhost:8001 system/engine/router_builtin.php
```

Update `APP_URL` accordingly.

## Storage permission errors

Symptoms include missing logs, failed cache writes, or errors creating `installed.lock`.

Create the directories:

```bash
mkdir -p storage/cache storage/logs storage/uploads
```

For a local Linux installation:

```bash
chmod -R u+rwX storage
```

For Nginx or Apache, assign the directories to the relevant web-server user. Avoid `chmod -R 777`.

## Docker returns `502 Bad Gateway`

Check the services:

```bash
docker compose -f docker/docker-compose.yml ps
```

Inspect logs:

```bash
docker compose -f docker/docker-compose.yml logs php
docker compose -f docker/docker-compose.yml logs web
```

Rebuild and restart when the PHP image has changed:

```bash
docker compose -f docker/docker-compose.yml build --no-cache php
docker compose -f docker/docker-compose.yml up -d
```

## Docker database is not ready

Inspect the database health and logs:

```bash
docker compose -f docker/docker-compose.yml ps
docker compose -f docker/docker-compose.yml logs db
```

Confirm that the password used by the installer matches `DB_ROOT_PASSWORD` used by Docker Compose.

## Health check diagnostics

Liveness:

```bash
curl -i http://localhost:8000/health/live
```

Readiness:

```bash
curl -i http://localhost:8000/health/ready
```

The readiness response reports database, configuration, and installation checks. Also inspect:

```text
storage/logs/errors.log
storage/logs/ocmod_errors.log
```
