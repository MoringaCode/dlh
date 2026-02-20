# Moringa DLH - Moodle Docker Setup

A fully containerised Moodle setup using Docker Compose, featuring PHP-FPM, Nginx, MariaDB, Redis, phpMyAdmin, and Mailpit.

---

## Stack

| Service | Image | Purpose |
|---|---|---|
| `moodle` | Custom (PHP-FPM) | Moodle application server |
| `cron` | Custom (PHP-FPM) | Moodle background task runner |
| `nginx` | nginx | Web server / reverse proxy |
| `db` | mariadb | Database |
| `redis` | redis | Cache / session store |
| `phpmyadmin` | phpmyadmin | Database GUI |
| `mailpit` | axllent/mailpit | Local email testing |

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed
- Git

---

## Getting Started

### 1. Clone this repository

```bash
git clone git@github.com:MoringaCode/dlh.git
cd dlh
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

```dotenv
#App Image
APP_IMAGE=moodle:dev

# Database
DB_HOST=db
DB_PORT=3306

# MySQL Credentials
MYSQL_ROOT_PASSWORD=
MYSQL_DATABASE=
MYSQL_USER=
MYSQL_PASSWORD=

# PHPMyAdmin
PMA_UPLOAD_LIMIT=100M

# Versions
MARIADB_VERSION=10.11
REDIS_VERSION=7
NGINX_VERSION=1.26
PHPMYADMIN_VERSION=5
MAILPIT_VERSION=latest

# Ports
NGINX_PORT=8083
PHPMYADMIN_PORT=8084
MAILPIT_UI_PORT=8025
MAILPIT_SMTP_PORT=1025

MOODLE_URL=http://localhost:8083

# Cron
CRON_INTERVAL=60
```

### 4. Configure Moodle

Create `dlh/config.php` (copy from `dlh/config-dist.php` and update):

```php
$CFG->dbtype    = 'mariadb';
$CFG->dbhost    = getenv('DB_HOST') ?: 'db';
$CFG->dbname    = getenv('MYSQL_DATABASE') ?: 'moodle';
$CFG->dbuser    = getenv('MYSQL_USER') ?: 'moodle';
$CFG->dbpass    = getenv('MYSQL_PASSWORD') ?: 'pass';
$CFG->wwwroot   = getenv('MOODLE_URL') ?: 'http://localhost:8080';
$CFG->dataroot  = '/var/www/moodledata';
$CFG->directorypermissions = 02777;

// Mail — route through Mailpit
$CFG->smtphosts = 'mailpit:1025';
$CFG->smtpsecure = '';
$CFG->smtpuser  = '';
$CFG->smtppass  = '';
```

### 5. Build and start the environment

```bash
docker compose up -d --build
```

### 6. Complete the Moodle installation

Open your browser and go to `http://localhost:8083` to run the Moodle web installer.

---

## Service URLs

| Service | URL |
|---|---|
| Moodle | http://localhost:8083 |
| phpMyAdmin | http://localhost:8084 |
| Mailpit UI | http://localhost:8025 |

---

## Common Commands

### Start / stop

```bash
docker compose up -d        # Start all services
docker compose down         # Stop and remove containers
docker compose down -v      # Stop and remove containers + volumes (destructive)
```

### Rebuild after Dockerfile changes

```bash
docker compose up -d --build
```

### View logs

```bash
docker compose logs -f              # All services
docker compose logs -f moodle       # Specific service
```

### Purge Moodle caches

```bash
docker compose exec moodle php admin/cli/purge_caches.php
```

### Access a container shell

```bash
docker compose exec moodle bash
docker compose exec db bash
```

### Run a manual cron cycle

```bash
docker compose exec moodle php admin/cli/cron.php
```

---

## Project Structure

```
.
├── dlh/                    # Moodle source code (git-ignored, clone separately)
├── docker/
│   └── php/
│       └── Dockerfile      # Custom PHP-FPM image
│       └── moodle.ini      # php.ini file
├── nginx/
│   └── default.conf        # Nginx server configuration
├── .env                    # Local environment variables (git-ignored)
├── .env.example            # Environment variable template
├── .gitignore
├── docker-compose.yml
└── README.md
```

---

## Email Testing

All outgoing email from Moodle is caught by Mailpit — no real emails are sent. View captured emails at [http://localhost:8025](http://localhost:8025).

To test email from within Moodle, go to **Site Administration → Server → Email → Outgoing mail configuration** and use the **Send test email** button.

---

## Troubleshooting

### `$CFG->dataroot is not writable`

**Full error:** `Fatal error: $CFG->dataroot is not writable, admin has to fix directory permissions! Exiting.`

This happens because the `moodledata` volume is owned by `root` when Docker first creates it, but the PHP-FPM process runs as `www-data` and cannot write to it.

Fix it by setting the correct ownership from inside the container:

```bash
docker compose exec moodle chown -R www-data:www-data /var/www/moodledata
```

Then verify the permissions look correct:

```bash
docker compose exec moodle ls -la /var/www/
```

You should see `moodledata` owned by `www-data`. Reload the Moodle installer in your browser — the error should be gone.

If the error persists, you can also explicitly set the permissions:

```bash
docker compose exec moodle chmod -R 777 /var/www/moodledata
```

Note that `777` is permissive and fine for local development, but should never be used in production.

---

## Notes

- The `dlh/` directory is bind-mounted into both the `moodle` and `nginx` containers so code changes are reflected immediately without a rebuild.
- The `moodledata/` volume stores user uploads, sessions, and cache — do not delete it while Moodle is in use.
- Redis is available as a cache and session store. Configure it in Moodle under **Site Administration → Server → Caching**.