# 03 — LAMP Stack and GLPI 11 Installation

## Why GLPI 11 over GLPI 10

At the time of this build, GLPI had moved to major version 11; GLPI 10
is in maintenance-only LTS mode. GLPI 11 was chosen deliberately for a
longer support runway on a brand-new deployment, even though it meant
tracking a newer set of requirements (see below).

## PHP configuration

Tuned in both `/etc/php/8.2/apache2/php.ini` and `/etc/php/8.2/cli/php.ini`:

```ini
memory_limit = 256M
upload_max_filesize = 20M
post_max_size = 20M
max_execution_time = 300
date.timezone = America/Sao_Paulo
```

## Required PHP extensions

GLPI 11 requires: `mysqli, curl, gd, intl, xml (dom/simplexml/xmlwriter/
xmlreader), mbstring, zip, bz2, ldap, apcu` — plus **`bcmath`**, which is
not always called out in third-party install guides but is enforced by
GLPI's own `db:install` requirements check:

```bash
apt install -y php-bcmath
systemctl restart apache2
```

## Secure directory layout

GLPI's recommended production layout moves `config/`, `files/`, and
`log/` **outside** the web-served directory, so they can never be served
by Apache even by accident:

| Constant | Path |
|---|---|
| `GLPI_CONFIG_DIR` | `/etc/glpi` |
| `GLPI_VAR_DIR` | `/var/lib/glpi/files` |
| `GLPI_LOG_DIR` | `/var/log/glpi` |

Declared in `/var/www/glpi/inc/downstream.php`:

```php
<?php
define('GLPI_CONFIG_DIR', '/etc/glpi');
define('GLPI_VAR_DIR',    '/var/lib/glpi/files');
define('GLPI_LOG_DIR',    '/var/log/glpi');
```

`DocumentRoot` points only at `/var/www/glpi/public` — the sole
web-exposed directory; application code, `front/`, `src/`, etc. live
outside the webroot and are only reachable through GLPI's own router.

## Database

A dedicated MariaDB user/database was created for GLPI (`glpi`@`localhost`,
database `glpi`), following least-privilege practice — this account has
no access beyond its own database and is not reachable from outside
`localhost`.

## Installer

GLPI's CLI installer refuses to run as `root` unless given
`--allow-superuser` (applies to essentially every `bin/console` command):

```bash
php bin/console db:install --allow-superuser
```

After install, ownership was reset to make sure nothing ended up owned
by `root` (which Apache/PHP, running as `www-data`, cannot read/write):

```bash
chown -R www-data:www-data /var/www/glpi /etc/glpi /var/lib/glpi /var/log/glpi
```

## Apache virtual host and URL routing

GLPI 11's shipped `public/` directory does **not** include a `.htaccess`
file, unlike some older third-party install guides assume. Without one,
`AllowOverride All` has nothing to act on, and any URL that doesn't map
to a literal file (e.g. `/front/login.php`) returns a raw Apache 404
instead of being routed through GLPI's front controller.

Fix: an explicit `FallbackResource` directive routes any non-existent
path through `index.php`, which GLPI's own router then dispatches:

> Hostname below is a placeholder (`example.com`) — see the note in
> the root `README.md`.

```apache
<VirtualHost *:80>
    ServerName srv-glpi.example.com
    DocumentRoot /var/www/glpi/public

    <Directory /var/www/glpi/public>
        AllowOverride All
        Require all granted

        FallbackResource /index.php
    </Directory>
</VirtualHost>
```

## Automatic actions (cron)

GLPI's background jobs (notifications, SLA checks, session cleanup,
etc.) need a system cron entry to run reliably in CLI mode rather than
only when a user happens to be browsing the site:

`/etc/cron.d/glpi`:
```
* * * * * www-data /usr/bin/php /var/www/glpi/front/cron.php &>/dev/null
```

Verified working: GLPI 11's default install already ships its scheduled
automatic actions set to **CLI** execution mode, and the cron entry
picked them up immediately (confirmed via each action's "last run"
timestamp in *Setup > Automatic actions*).

## Post-install hardening

- Rotated all four default seeded accounts (`glpi`, `tech`, `normal`,
  `post-only`) — these ship with well-known public credentials
  (`glpi/glpi`, etc.) and are the first target of any automated scan.
- Disabled GLPI's built-in demo data.
- Removed the installer entry point once installation was confirmed
  working:
```bash
  rm /var/www/glpi/install/install.php
```
