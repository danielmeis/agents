# WordPress 7.0+ Deployment & Environment Reference

> Load this file for WP-CLI workflows, staging-to-production processes,
> environment management, performance constants, and server configuration.

---

## Environment Constants

```php
// wp-config.php — environment-aware setup

// Detect environment (set via server env var, not in wp-config itself)
define( 'WP_ENVIRONMENT_TYPE', getenv( 'WP_ENV' ) ?: 'production' );
// Valid values: 'local', 'development', 'staging', 'production'

// Debug — only on non-production environments
if ( in_array( WP_ENVIRONMENT_TYPE, [ 'local', 'development' ], true ) ) {
    define( 'WP_DEBUG',         true );
    define( 'WP_DEBUG_LOG',     true );      // logs to wp-content/debug.log
    define( 'WP_DEBUG_DISPLAY', false );     // never display errors
    define( 'SCRIPT_DEBUG',     true );      // load unminified JS/CSS
    define( 'SAVEQUERIES',      true );      // log all DB queries (dev only)
} else {
    define( 'WP_DEBUG',         false );
    define( 'WP_DEBUG_LOG',     false );
    define( 'WP_DEBUG_DISPLAY', false );
    @ini_set( 'display_errors', 0 );
}

// Performance constants
define( 'WP_POST_REVISIONS',  3 );
define( 'EMPTY_TRASH_DAYS',   7 );
define( 'AUTOSAVE_INTERVAL',  60 );         // seconds; default is 60
define( 'WP_MEMORY_LIMIT',    '256M' );     // for front-end
define( 'WP_MAX_MEMORY_LIMIT','512M' );     // for admin/cron

// Security
define( 'DISALLOW_FILE_EDIT', true );
define( 'FORCE_SSL_ADMIN',    true );
define( 'WP_AUTO_UPDATE_CORE', 'minor' );   // auto-receive security patches

// Cron — disable built-in cron if using server-side cron (recommended for large sites)
define( 'DISABLE_WP_CRON', true );          // pair with: * * * * * wp cron event run --due-now

// Uploads / paths (do not change on existing sites without migrating files)
define( 'UPLOADS', 'wp-content/uploads' );  // relative to ABSPATH

// Redis (if using Redis Object Cache plugin or similar)
define( 'WP_REDIS_HOST',         '127.0.0.1' );
define( 'WP_REDIS_PORT',         6379 );
define( 'WP_REDIS_DATABASE',     0 );
define( 'WP_REDIS_TIMEOUT',      1 );
define( 'WP_REDIS_READ_TIMEOUT', 1 );
define( 'WP_CACHE',              true );
```

---

## Server-Side Cron (Required for Large Sites)

WordPress's built-in pseudo-cron (`wp-cron.php`) fires on page loads — unreliable,
and it can cause a stampede on high-traffic sites if multiple requests trigger
heavy cron jobs simultaneously.

```bash
# Disable WP-Cron in wp-config.php:
# define( 'DISABLE_WP_CRON', true );

# Add a real cron job (runs every minute):
* * * * * /usr/bin/wp --path=/var/www/html cron event run --due-now \
    --url=https://example.com >> /var/log/wp-cron.log 2>&1

# Or with PHP directly:
* * * * * php /var/www/html/wp-cron.php >> /var/log/wp-cron.log 2>&1

# Monitor the cron queue
wp cron event list
wp cron schedule list
wp cron event run --due-now --dry-run    # see what would fire without running
```

---

## WP-CLI Reference for Large Sites

```bash
# ── Core ──────────────────────────────────────────────────────────────────────
wp core version
wp core update
wp core verify-checksums                  # detect modified core files
wp core update-db                         # run after major version update

# ── Database ──────────────────────────────────────────────────────────────────
wp db size --tables                       # size per table
wp db optimize                            # runs OPTIMIZE TABLE on all tables
wp db repair                              # attempt repair on corrupted tables
wp db export --add-drop-table \
    --single-transaction \                # consistent snapshot (InnoDB)
    - | gzip > backup-$(date +%F).sql.gz

# ── Plugin/Theme management ───────────────────────────────────────────────────
wp plugin list --status=inactive --format=table    # find and remove unused plugins
wp plugin update --all
wp theme list --status=inactive --format=table
wp plugin verify-checksums --all                   # detect modified plugin files

# ── Users ─────────────────────────────────────────────────────────────────────
wp user list --role=administrator
wp user create deploy_bot deploy@example.com \
    --role=editor --user_pass=$(openssl rand -base64 32)

# ── Options / Autoload ────────────────────────────────────────────────────────
# Total autoloaded payload
wp db query "SELECT ROUND(SUM(LENGTH(option_value))/1024,1) AS kb
             FROM wp_options WHERE autoload='yes';"

# Transients
wp transient delete --expired --all

# ── Search-replace (handles serialized data) ──────────────────────────────────
wp search-replace 'http://staging.example.com' 'https://example.com' \
    --skip-columns=guid \
    --dry-run
wp search-replace 'http://staging.example.com' 'https://example.com' \
    --skip-columns=guid

# ── Media ─────────────────────────────────────────────────────────────────────
wp media regenerate --yes --batch-size=50   # regenerate thumbnails in batches

# ── Cache ─────────────────────────────────────────────────────────────────────
wp cache flush
wp rewrite flush

# ── Performance audit ─────────────────────────────────────────────────────────
wp post list --post_type=post --post_status=publish --format=count
wp post list --post_type=revision --format=count    # how many revisions exist?
```

---

## Staging-to-Production Workflow

```bash
# ── Step 1: Export from staging ────────────────────────────────────────────────
# On staging server:
wp db export --add-drop-table --single-transaction - | gzip > /tmp/staging-$(date +%F).sql.gz
rsync -avz --exclude='node_modules' --exclude='.git' \
    /var/www/staging/wp-content/ deploy@prod:/var/www/html/wp-content.new/

# ── Step 2: Deploy code (never FTP — use git or rsync) ────────────────────────
# On production:
cd /var/www/html
git fetch origin
git checkout main
git pull origin main

# ── Step 3: Run database updates ───────────────────────────────────────────────
wp core update-db
wp plugin update --all

# ── Step 4: Search-replace domain ──────────────────────────────────────────────
wp search-replace 'https://staging.example.com' 'https://example.com' \
    --skip-columns=guid

# ── Step 5: Flush everything ────────────────────────────────────────────────────
wp cache flush
wp rewrite flush
wp cron event run --due-now

# ── Step 6: Verify ──────────────────────────────────────────────────────────────
wp core verify-checksums
wp plugin verify-checksums --all
wp core version
```

---

## PHP-FPM Configuration for Large WordPress Sites

```ini
; /etc/php/8.3/fpm/pool.d/wordpress.conf

[wordpress]
user  = www-data
group = www-data

listen = /run/php/php8.3-fpm-wordpress.sock
listen.owner = www-data
listen.group = www-data

; Process management: dynamic is safest for variable traffic
pm                   = dynamic
pm.max_children      = 50     ; (total RAM - OS overhead) / per-process RAM (~40-80MB)
pm.start_servers     = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 20
pm.max_requests      = 500    ; recycle workers to prevent memory leaks

; Timeouts
request_terminate_timeout = 60s

; PHP settings for WordPress
php_admin_value[memory_limit]        = 256M
php_admin_value[upload_max_filesize] = 64M
php_admin_value[post_max_size]       = 64M
php_admin_value[max_execution_time]  = 60
php_admin_value[max_input_vars]      = 3000    ; needed for large option pages
php_admin_flag[display_errors]       = off
php_admin_value[error_log]           = /var/log/php-fpm-wordpress-error.log

; OPcache (essential for WordPress performance)
php_admin_value[opcache.enable]           = 1
php_admin_value[opcache.memory_consumption] = 256
php_admin_value[opcache.interned_strings_buffer] = 16
php_admin_value[opcache.max_accelerated_files]   = 20000
php_admin_value[opcache.revalidate_freq]         = 60    ; check for changes every 60s
php_admin_value[opcache.validate_timestamps]      = 0    ; 0 = never revalidate (prod only)
php_admin_value[opcache.save_comments]           = 1    ; required for WordPress docblocks
```

---

## Object Cache: Redis Setup

Without a persistent object cache, every `wp_cache_get()` miss hits the database
and `wp_options` autoloads fire on every request. Redis is the standard solution.

```bash
# Install Redis server
apt install redis-server
systemctl enable redis-server

# Install Redis PHP extension
apt install php8.3-redis

# Install the Redis Object Cache plugin (or use the Relay plugin for better perf)
wp plugin install redis-cache --activate
wp redis enable

# Verify Redis is working
wp redis status
```

```php
// After enabling Redis, the object cache backend changes from in-memory-per-request
// to persistent Redis. This means:
// - wp_cache_set() persists across requests
// - Transients are stored in Redis, NOT in wp_options (autoload issue solved)
// - wp_cache_flush() flushes Redis — call it after bulk imports
// - cache groups let you selectively invalidate related keys

// Flush only a specific cache group (requires object-cache.php that supports groups)
wp_cache_flush_group( 'my_plugin' );

// Use cache groups consistently across your plugin
wp_cache_set( $key, $value, 'my_plugin', 5 * MINUTE_IN_SECONDS );
wp_cache_get( $key, 'my_plugin' );
wp_cache_delete( $key, 'my_plugin' );
```

---

## Deployment Checklist

Before every production deploy:

```
Security
[ ] WordPress core is on latest stable (7.0.2 minimum as of July 2026)
[ ] All plugins updated; inactive plugins removed
[ ] wp core verify-checksums passes
[ ] wp plugin verify-checksums --all passes
[ ] DISALLOW_FILE_EDIT is defined true in wp-config.php
[ ] WP_DEBUG is false in production
[ ] No error_log() calls left in plugin code (use conditional on WP_DEBUG)
[ ] No hardcoded credentials or API keys in committed code

Performance
[ ] Redis object cache is enabled and connected (wp redis status)
[ ] OPcache is enabled (check phpinfo() or opcache_get_status())
[ ] DISABLE_WP_CRON is true; server cron is configured
[ ] WP_POST_REVISIONS is limited (3 or false)
[ ] Autoloaded options payload is under 800KB (see mariadb-best-practices skill)

Database
[ ] wp core update-db has been run after any core update
[ ] No pending plugin database migrations
[ ] Backup taken and tested before deploy

Functionality
[ ] Tested on staging with production data snapshot
[ ] Search-replace completed if domain changed
[ ] wp rewrite flush run after deploy
[ ] wp cache flush run after deploy
[ ] wp cron event run --due-now run to catch up any missed cron jobs
```
*Last Updated: 2026-07-28*
