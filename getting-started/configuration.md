# Configuration

CoreCart reads application settings from environment variables. The CLI installer writes the main settings to a `.env` file in the project root.

## Environment file

A typical local `.env` file looks like this:

```dotenv
DB_HOST=127.0.0.1
DB_NAME=corecart
DB_USER=root
DB_PASS=your-database-password
DB_CHARSET=utf8mb4

APP_NAME=CoreCart
APP_URL=http://localhost:8000
APP_DEBUG=true
```

Do not commit `.env` to source control. It may contain database credentials and other secrets.

## Database settings

| Variable | Default | Description |
|---|---:|---|
| `DB_HOST` | `localhost` | MySQL or MariaDB host. Use `db` with the bundled Docker Compose configuration. |
| `DB_NAME` | `corecart` | Database name. Only letters, numbers, and underscores are accepted by the CLI installer. |
| `DB_USER` | `root` | Database user. |
| `DB_PASS` | Empty | Database password. |
| `DB_CHARSET` | `utf8mb4` | Database character set written by the installer. CoreCart currently connects using `utf8mb4`. |

Use a dedicated database user outside local development. Grant it access only to the CoreCart database.

Example:

```sql
CREATE USER 'corecart'@'localhost' IDENTIFIED BY 'strong-password';
GRANT ALL PRIVILEGES ON corecart.* TO 'corecart'@'localhost';
FLUSH PRIVILEGES;
```

## Application settings

| Variable | Recommended local value | Description |
|---|---:|---|
| `APP_NAME` | `CoreCart` | Application name. |
| `APP_URL` | `http://localhost:8000` | Base application URL. Use `http://localhost:8080` with the bundled Docker setup. |
| `APP_DEBUG` | `true` | Local development flag. Keep it disabled in hosted environments. |

For local HTTP development, export `APP_DEBUG=true` before starting PHP so session cookies are not marked as HTTPS-only:

```bash
APP_DEBUG=true php -S localhost:8000 system/engine/router_builtin.php
```

On Windows Command Prompt:

```bat
set APP_DEBUG=true
php -S localhost:8000 system/engine/router_builtin.php
```

!> Never expose debug output or development credentials on a public server.

## Docker Compose settings

The bundled Docker configuration supports this host-side variable:

| Variable | Default | Description |
|---|---:|---|
| `DB_ROOT_PASSWORD` | `secret` | MySQL root password used by Docker Compose and passed to the PHP container. |

Set it before running Docker Compose:

```bash
export DB_ROOT_PASSWORD='replace-with-a-strong-password'
```

The bundled services use the following internal values:

```text
Database host: db
Database name: corecart
Database user: root
Application port: 8080
```

## Storage directories

CoreCart uses these paths:

| Path | Purpose |
|---|---|
| `storage/cache/` | Generated cache and modification files. |
| `storage/logs/` | Application and error logs. |
| `storage/uploads/` | Uploaded files. |
| `storage/installed.lock` | Indicates that the installer has completed. |

The web-server process must be able to write to the storage directories.

## Session configuration

CoreCart configures PHP sessions with:

- HTTP-only cookies;
- `SameSite=Lax`;
- strict session mode;
- cookie-only sessions;
- session ID regeneration after authentication;
- a two-hour maximum lifetime for administrative sessions;
- an administrative idle timeout.

Use HTTPS in hosted environments so secure session cookies can be transmitted correctly.

## Configuration precedence

Process-level environment variables should be treated as the highest-priority configuration. This is especially useful for Docker, CI, and hosted environments where secrets should not be written to the repository.

Example:

```bash
DB_HOST=127.0.0.1 \
DB_NAME=corecart \
DB_USER=corecart \
DB_PASS='strong-password' \
APP_DEBUG=false \
php -S localhost:8000 system/engine/router_builtin.php
```
