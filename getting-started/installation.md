# Installation and Startup

This guide covers local installation and Docker-based startup.

## Standard installation

### 1. Download CoreCart

```bash
git clone https://github.com/nothapt/CoreCart.git
cd CoreCart
```

### 2. Install PHP dependencies

```bash
composer install
```

### 3. Prepare storage directories

```bash
mkdir -p storage/cache storage/logs storage/uploads
```

The PHP process must have write access to these directories.

### 4. Create the database and administrator

Use the CLI installer:

```bash
php cli.php install \
  --db_host=127.0.0.1 \
  --db_user=root \
  --db_pass=your-database-password \
  --db_name=corecart \
  --admin_user=admin \
  --admin_email=admin@example.com \
  --admin_pass=use-a-strong-password
```

The installer performs the following operations:

1. Connects to MySQL.
2. Creates the database when it does not exist.
3. Imports `install/database.sql`.
4. Creates or updates the administrator password.
5. Writes the root `.env` file.
6. Creates `storage/installed.lock`.

!> Do not use the example administrator password on a public server.

### 5. Start the development server

```bash
APP_DEBUG=true php -S localhost:8000 system/engine/router_builtin.php
```

Open:

```text
http://localhost:8000
```

Health checks:

```text
http://localhost:8000/health/live
http://localhost:8000/health/ready
```

## Startup scripts

After installation, Linux and macOS users can start the server with:

```bash
chmod +x run.sh
APP_DEBUG=true ./run.sh
```

Windows users can run:

```bat
set APP_DEBUG=true
run.bat
```

Both scripts install Composer dependencies when `vendor/` is missing and start the application on port `8000`.

## Docker installation

The Docker configuration starts Nginx, PHP-FPM, and MySQL. The application is exposed on port `8080`.

### 1. Clone the repository

```bash
git clone https://github.com/nothapt/CoreCart.git
cd CoreCart
```

### 2. Set the MySQL root password

Linux and macOS:

```bash
export DB_ROOT_PASSWORD='replace-with-a-strong-password'
```

PowerShell:

```powershell
$env:DB_ROOT_PASSWORD = 'replace-with-a-strong-password'
```

### 3. Build the containers

```bash
docker compose -f docker/docker-compose.yml build
```

### 4. Install Composer dependencies into the mounted project

```bash
docker compose -f docker/docker-compose.yml run --rm php composer install
```

### 5. Run the CoreCart installer

```bash
docker compose -f docker/docker-compose.yml run --rm php \
  php cli.php install \
  --db_host=db \
  --db_user=root \
  --db_pass="$DB_ROOT_PASSWORD" \
  --db_name=corecart \
  --admin_user=admin \
  --admin_email=admin@example.com \
  --admin_pass=use-a-strong-password
```

PowerShell users should replace `"$DB_ROOT_PASSWORD"` with `$env:DB_ROOT_PASSWORD` or enter the value directly.

### 6. Start CoreCart

```bash
docker compose -f docker/docker-compose.yml up -d
```

Open:

```text
http://localhost:8080
```

View container status:

```bash
docker compose -f docker/docker-compose.yml ps
```

View logs:

```bash
docker compose -f docker/docker-compose.yml logs -f
```

Stop the containers:

```bash
docker compose -f docker/docker-compose.yml down
```

## Reinstallation

The CLI installer refuses to run when `storage/installed.lock` exists.

For a clean local reinstallation:

1. Back up any required data.
2. Remove `storage/installed.lock`.
3. Drop or rename the existing database.
4. Run the installer again.

!> Reinstallation can destroy existing store data. Do not perform it against a database that contains required information.
