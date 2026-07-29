# WordPress 7.0+ Security Hardening Reference

> Load this file for detailed security checklists, REST API lockdown patterns,
> file hardening, and plugin/theme security review guidance.

---

## ⚠️ WP2Shell — Active Exploit (July 2026)

**CVE-2026-60137** (SQL injection in `WP_Query::author__not_in`) +
**CVE-2026-63030** (REST API batch-route confusion at `/wp-json/batch/v1`)

- Affects: WordPress 6.8.0–6.8.5 (SQLi only), 6.9.0–6.9.4 and 7.0.0–7.0.1 (full RCE chain)
- Fixed in: **7.0.2**, 6.9.5, 6.8.6 — update immediately
- In-the-wild exploitation confirmed as of July 18, 2026
- Added to CISA KEV catalog July 21, 2026
- No plugins or prior authentication required — stock default install is vulnerable

```bash
# Verify you are patched
wp core version          # must show 7.0.2 or higher
wp core update           # if not patched
wp core verify-checksums # confirm core file integrity post-update
```

**Code pattern to audit in your own plugins** (the root cause):

```php
// VULNERABLE: user-controlled string reaches author__not_in
new WP_Query( [ 'author__not_in' => $_GET['exclude'] ] );

// SAFE: always cast to array of integers
$exclude = array_map( 'intval', (array) ( $_GET['exclude'] ?? [] ) );
new WP_Query( [ 'author__not_in' => array_filter( $exclude ) ] );
```

---

## wp-config.php Hardening

```php
<?php
// Move wp-config.php one directory above the web root when possible.
// The following constants belong in every production wp-config.php.

// ── Authentication Keys (generate at https://api.wordpress.org/secret-key/1.1/salt/)
define( 'AUTH_KEY',         'unique-phrase-here' );
define( 'SECURE_AUTH_KEY',  'unique-phrase-here' );
define( 'LOGGED_IN_KEY',    'unique-phrase-here' );
define( 'NONCE_KEY',        'unique-phrase-here' );
define( 'AUTH_SALT',        'unique-phrase-here' );
define( 'SECURE_AUTH_SALT', 'unique-phrase-here' );
define( 'LOGGED_IN_SALT',   'unique-phrase-here' );
define( 'NONCE_SALT',       'unique-phrase-here' );

// ── SSL
define( 'FORCE_SSL_ADMIN', true );

// ── File editing (disables Appearance > Editor and Plugins > Editor)
define( 'DISALLOW_FILE_EDIT', true );

// ── Prevent plugin/theme installation from admin (optional, for locked-down prod)
define( 'DISALLOW_FILE_MODS', true );

// ── Revision control (prevents wp_posts bloat)
define( 'WP_POST_REVISIONS', 3 );

// ── Auto-updates: receive security releases automatically
define( 'WP_AUTO_UPDATE_CORE', 'minor' );  // or true for all updates

// ── Trash
define( 'EMPTY_TRASH_DAYS', 7 );

// ── Debug: OFF in production
define( 'WP_DEBUG',         false );
define( 'WP_DEBUG_LOG',     false );
define( 'WP_DEBUG_DISPLAY', false );
@ini_set( 'display_errors', 0 );

// ── Table prefix (change from default 'wp_' on new installs)
$table_prefix = 'xk7m_';   // use a random prefix

// ── Redis object cache (if using Redis)
define( 'WP_REDIS_HOST',     '127.0.0.1' );
define( 'WP_REDIS_PORT',     6379 );
define( 'WP_REDIS_DATABASE', 0 );
define( 'WP_CACHE',          true );
```

---

## .htaccess Hardening (Apache)

```apache
# Block access to wp-config.php
<Files wp-config.php>
    Order Allow,Deny
    Deny from all
</Files>

# Block access to sensitive files
<FilesMatch "\.(log|sql|bak|sh|env)$">
    Order Allow,Deny
    Deny from all
</FilesMatch>

# Disable XML-RPC (if not needed)
<Files xmlrpc.php>
    Order Allow,Deny
    Deny from all
</Files>

# Block user enumeration via ?author=N
RewriteEngine On
RewriteCond %{QUERY_STRING} ^author=\d
RewriteRule ^ /? [L,R=301]

# Block PHP execution in uploads
<Directory /var/www/html/wp-content/uploads>
    <FilesMatch "\.php$">
        Order Allow,Deny
        Deny from all
    </FilesMatch>
</Directory>

# Security headers
<IfModule mod_headers.c>
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';"
</IfModule>
```

---

## Nginx Hardening

```nginx
# Block access to sensitive files
location ~* \.(log|sql|bak|sh|env)$ { deny all; }
location = /wp-config.php            { deny all; }
location = /xmlrpc.php               { deny all; }

# Block PHP in uploads
location ~* /wp-content/uploads/.*\.php$ { deny all; }

# Block user enumeration
if ( $query_string ~* "author=\d+" ) { return 301 /; }

# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Rate-limit the login page and REST auth endpoints
limit_req_zone $binary_remote_addr zone=wp_login:10m rate=5r/m;
location = /wp-login.php {
    limit_req zone=wp_login burst=3 nodelay;
    include fastcgi_params;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
}
```

---

## REST API Security Checklist

```php
// 1. Block user enumeration via REST
add_filter( 'rest_endpoints', function( array $endpoints ): array {
    if ( ! is_user_logged_in() ) {
        unset( $endpoints['/wp/v2/users'] );
        unset( $endpoints['/wp/v2/users/(?P<id>[\d]+)'] );
    }
    return $endpoints;
} );

// 2. Every custom route MUST have a real permission_callback — never __return_true
register_rest_route( 'my-plugin/v1', '/data', [
    'methods'             => WP_REST_Server::READABLE,
    'callback'            => [ $this, 'get_data' ],
    'permission_callback' => fn() => current_user_can( 'read' ),
] );

// 3. Sanitize and validate all args
'args' => [
    'post_id' => [
        'required'          => true,
        'type'              => 'integer',
        'sanitize_callback' => 'absint',
        'validate_callback' => fn( $v ) => $v > 0,
    ],
],

// 4. Restrict REST API to authenticated users for non-public endpoints
add_filter( 'rest_authentication_errors', function( $result ) {
    // Allow public endpoints to pass through
    if ( true === $result || is_wp_error( $result ) ) {
        return $result;
    }
    // If not authenticated and accessing non-public route, block
    if ( ! is_user_logged_in() ) {
        return new WP_Error(
            'rest_not_logged_in',
            __( 'Authentication required.', 'my-plugin' ),
            [ 'status' => 401 ]
        );
    }
    return $result;
} );

// 5. Application Passwords for external integrations (7.0+)
// Users generate these under Users > Profile > Application Passwords
// Use HTTP Basic Auth: Authorization: Basic base64(user:app_password)
// Never store raw passwords — they are shown once on creation

// 6. Abilities API endpoints (7.0+) need identical protection
// The MCP adapter does NOT automatically authenticate incoming AI agent requests
// Treat them like any other REST endpoint
```

---

## Input Handling — Complete Reference

```php
// Sanitization functions — use the most specific one that fits
sanitize_text_field( $input )       // strips tags, extra whitespace, invalid UTF-8
sanitize_textarea_field( $input )   // same but preserves newlines
sanitize_email( $input )            // strips invalid email characters
sanitize_url( $input )              // strips invalid URL characters (use esc_url_raw for DB)
sanitize_file_name( $input )        // safe file names
sanitize_key( $input )              // lowercase alphanumeric + dashes/underscores
sanitize_html_class( $input )       // safe CSS class names
wp_kses_post( $input )              // allows safe HTML subset (for rich content)
wp_kses( $input, $allowed_tags )    // custom allowed HTML
absint( $input )                    // positive integer
intval( $input )                    // integer (can be negative)
floatval( $input )                  // float
wp_unslash( $input )                // remove magic-quote slashes (apply before sanitizing)

// Escaping functions — use at output, not storage
esc_html( $value )      // escape for HTML content
esc_attr( $value )      // escape for HTML attributes
esc_url( $value )       // escape for href/src attributes
esc_url_raw( $value )   // escape for database storage
esc_js( $value )        // escape for inline JavaScript
esc_sql( $value )       // escape for SQL — prefer $wpdb->prepare() instead
wp_json_encode( $value )            // safe JSON for inline scripts

// Use wp_unslash() before any sanitization of superglobal data
$title = sanitize_text_field( wp_unslash( $_POST['title'] ?? '' ) );
```

---

## File Upload Security

```php
add_action( 'wp_ajax_my_upload', function(): void {
    check_ajax_referer( 'my_upload_nonce', 'nonce' );

    if ( ! current_user_can( 'upload_files' ) ) {
        wp_send_json_error( 'Insufficient permissions.', 403 );
    }

    // Validate MIME type — never trust the file extension alone
    $file     = $_FILES['file'] ?? null;
    $allowed  = [ 'image/jpeg', 'image/png', 'image/webp', 'image/gif' ];
    $finfo    = new finfo( FILEINFO_MIME_TYPE );
    $detected = $finfo->file( $file['tmp_name'] );

    if ( ! in_array( $detected, $allowed, true ) ) {
        wp_send_json_error( 'File type not allowed.', 415 );
    }

    // Use WordPress's own upload handler — it handles paths, naming, and DB
    require_once ABSPATH . 'wp-admin/includes/file.php';
    require_once ABSPATH . 'wp-admin/includes/image.php';
    require_once ABSPATH . 'wp-admin/includes/media.php';

    $attachment_id = media_handle_upload( 'file', 0 );

    if ( is_wp_error( $attachment_id ) ) {
        wp_send_json_error( $attachment_id->get_error_message(), 500 );
    }

    wp_send_json_success( [ 'attachment_id' => $attachment_id ] );
} );
```

---

## Security Audit Checklist

Run this before every production deploy and periodically on running sites:

```bash
# Core integrity
wp core verify-checksums

# Plugin/theme vulnerability scan (requires wp-cli vulnerability scanner)
wp plugin list --format=json | jq '.[].version'
# Cross-reference with WPScan vulnerability database: https://wpscan.com/api

# Check for inactive plugins (attack surface even when inactive — delete them)
wp plugin list --status=inactive --format=table

# Check for files modified in the last 7 days in core directories (potential compromise)
find /var/www/html/wp-includes /var/www/html/wp-admin \
    -name "*.php" -newer /var/www/html/wp-login.php \
    -not -path "*/node_modules/*" 2>/dev/null

# Verify no PHP files in uploads
find /var/www/html/wp-content/uploads -name "*.php" 2>/dev/null

# Check admin users
wp user list --role=administrator --fields=ID,user_login,user_email,user_registered
```

```php
// PHP: audit admin accounts programmatically
$admins = get_users( [ 'role' => 'administrator', 'fields' => [ 'ID', 'user_login', 'user_email' ] ] );
foreach ( $admins as $admin ) {
    // Flag any admin not in your known-safe list
    if ( ! in_array( $admin->user_login, MY_PLUGIN_KNOWN_ADMINS, true ) ) {
        // Alert via email or logging
        error_log( "Unexpected admin: {$admin->user_login} ({$admin->user_email})" );
    }
}
```

---

## Plugin Security Code Review Checklist

When reviewing a plugin (yours or third-party) for production use:

- [ ] All AJAX handlers check `check_ajax_referer()` before any action
- [ ] All REST routes have a real `permission_callback` (not `__return_true`)
- [ ] All `$_GET`/`$_POST`/`$_REQUEST` values are sanitized before use
- [ ] All `$wpdb` queries use `$wpdb->prepare()`
- [ ] All output is escaped with the correct `esc_*` function
- [ ] `author__not_in` and other `__not_in` params receive arrays, not strings
- [ ] No `eval()`, `base64_decode()` on user input, or obfuscated code
- [ ] File uploads validate MIME type via `finfo`, not just file extension
- [ ] No direct file system writes outside `wp-content/uploads`
- [ ] Options/meta values escaped on output even if sanitized on input
- [ ] Cron callbacks process a bounded batch size (never unlimited loops)
- [ ] No hardcoded credentials, API keys, or secrets in plugin code

*Last Updated: 2026-07-28*
