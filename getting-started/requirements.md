# System Requirements

This page lists the software and PHP extensions required to install and run CoreCart.

## Server requirements

- **PHP 8.4 or later**
- **MySQL 8.0 or later**, or **MariaDB 10.6 or later**
- **Composer 2.x**
- A supported web server:
  - PHP built-in server for local development
  - Nginx or Apache for hosted environments

## Required PHP extensions

CoreCart requires the following PHP extensions:

- `pdo`
- `pdo_mysql`
- `mbstring`
- `curl`
- `intl`
- `bcmath`

The `bcmath` extension is required for price and order total calculations.

Check the installed PHP version and extensions:

```bash
php --version
php -m
```

On Debian or Ubuntu, the required packages can usually be installed with:

```bash
sudo apt update
sudo apt install php8.4 php8.4-cli php8.4-mysql php8.4-mbstring \
  php8.4-curl php8.4-intl php8.4-bcmath
```

Package names may differ between operating systems and PHP repositories.

## Composer

Confirm that Composer is available:

```bash
composer --version
```

CoreCart dependencies are installed from the project root:

```bash
composer install
```

## Writable directories

The PHP process must be able to write to these directories:

```text
storage/cache/
storage/logs/
storage/uploads/
```

Create them when they do not exist:

```bash
mkdir -p storage/cache storage/logs storage/uploads
```

On Linux, assign ownership to the web-server user when CoreCart is served by Nginx or Apache. Avoid using world-writable permissions such as `0777`.

## Optional tools

The following tools are optional but useful:

- Git
- Docker Engine
- Docker Compose
- MySQL command-line client

## Quick requirement check

Run the following commands before installation:

```bash
php --version
php -r "echo extension_loaded('pdo_mysql') ? 'pdo_mysql: OK' : 'pdo_mysql: MISSING';"
php -r "echo extension_loaded('bcmath') ? 'bcmath: OK' : 'bcmath: MISSING';"
composer --version
```

!> Use PHP 8.4 or later for both the command-line interface and the web server. Different PHP versions between CLI and FPM/Apache can produce inconsistent results.
