---
name: wordpress
description: >
  WordPress 7.0+ development best practices with PHP 8.3+, large-scale
  performance (1M+ posts), MariaDB 10.11+, and security hardening. Use this
  skill whenever the user asks about WordPress plugin or theme development,
  WP_Query, custom post types, hooks/filters, block editor, PHP-only blocks,
  REST API, the Abilities API, WP AI Client, DataViews, Connectors API, caching
  strategy, security (nonces, capabilities, escaping, sanitization), background
  processing, WP-CLI, deployment, or any task involving WordPress architecture
  or scaling. Also trigger for questions about slow WordPress queries, database
  bloat, transients, object caching, or any WordPress + MariaDB performance issue.
  For deep MariaDB configuration (my.cnf, replication, index DDL), defer to the
  mariadb-best-practices skill.
---

# WordPress 7.0+ Best Practices

> Scope: WordPress 7.0.2+ (current as of July 2026), PHP 8.3+, MariaDB 10.11+.
> The WP2Shell exploit (CVE-2026-60137 + CVE-2026-63030) was actively exploited
> in the wild as of July 2026 — see the Security section for mandatory guidance.

---

## What's New in WordPress 7.0 (Developer Summary)

- **PHP minimum**: 7.4 (end-of-life — run PHP 8.3+; 7.4 will be dropped soon)
- **PHP-only block registration**: build blocks with zero JavaScript/build pipeline
- **DataViews**: React-based replacement for `WP_List_Table` on core admin screens;
  plugins that hook into legacy list tables via PHP filters need updates
- **Block editor always iframed**: CSS/JS that assumed access to the outer admin
  frame from inside the editor is broken by design — use `postMessage` APIs
- **WP AI Client** (`wordpress/wp-ai-client`): provider-agnostic AI interface;
  use `wp_ai_client_prompt()` to chain requests to external LLMs
- **Abilities API**: registry for defining/executing actions in the browser;
  exposes WordPress capabilities to AI agents via a built-in MCP adapter
- **Connectors API Hub**: centralized `Settings > Connectors` for AI provider
  keys — never store third-party API keys in plugin settings tables
- **Block Bindings API**: pattern overrides now work with any custom block, not
  just core blocks
- **Breadcrumbs block**, **Icons block**, **responsive Grid block** added to core
- **Navigation overlays**: fully customizable as template parts (block themes only)
- **Interactivity API router changed**: requires code updates if you used it pre-7.0

---

## Core Principles

- `declare(strict_types=1);` at the top of every PHP file
- PHP 8.3+ features: typed class constants, `#[\Override]`, readonly classes,
  readonly properties, named arguments, enums, fibers
- Lowercase with hyphens for directories: `wp-content/plugins/my-plugin/`
- **Never modify core WordPress files**
- Use hooks (`add_action`, `add_filter`) for all extension — never monkey-patch
- Use `WP_Query` (or the REST API) for data retrieval — never raw `$wpdb->query`
  for post/meta reads unless you have profiled a specific need
- All database writes through `$wpdb->prepare()` — no exceptions
- Use `wp_cache_*` / object cache before any database read in hot paths

---

## Security

### ⚠️ Critical: WP2Shell — Update to 7.0.2 Immediately

WordPress 7.0.0 and 7.0.1 are affected by a pre-authentication RCE exploit chain
(WP2Shell) actively exploited in the wild since July 18, 2026:

- **CVE-2026-60137**: SQL injection in `WP_Query` via the `author__not_in` parameter
  when passed as a string instead of an array
- **CVE-2026-63030**: REST API batch-route confusion at `/wp-json/batch/v1` that
  bypasses authentication checks and routes requests to the vulnerable handler

**Fixed in 7.0.2** (released July 17, 2026). Verify your version:
```bash
wp core version
# Must be 7.0.2 or higher
wp core update   # if not already patched
```

**Defensive coding lesson from WP2Shell**: never pass user-controlled values to
`author__not_in` (or any `__not_in` parameter) as strings — always cast to
arrays of integers:

```php
// VULNERABLE (string-typed user input reaches author__not_in)
$args = [ 'author__not_in' => $_GET['exclude_author'] ];

// SAFE: always cast and intval-filter before passing to WP_Query
$exclude = array_map( 'intval', (array) $_GET['exclude_author'] );
$args = [ 'author__not_in' => array_filter( $exclude ) ];
```

### Nonces

```php
// Generate
$nonce = wp_create_nonce( 'my_action_nonce' );

// Verify — do this before ANY state-changing action
if ( ! check_ajax_referer( 'my_action_nonce', 'nonce', false ) ) {
    wp_send_json_error( [ 'message' => 'Invalid nonce.' ], 403 );
}

// Form verification
if ( ! isset( $_POST['_wpnonce'] ) ||
     ! wp_verify_nonce( sanitize_text_field( wp_unslash( $_POST['_wpnonce'] ) ), 'save_settings' ) ) {
    wp_die( esc_html__( 'Security check failed.', 'my-plugin' ) );
}
```

### Capability Checks

```php
// Always check capabilities before doing privileged work
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'Insufficient permissions.', 'my-plugin' ), 403 );
}

// REST API: always include permission_callback
register_rest_route( 'my-plugin/v1', '/settings', [
    'methods'             => 'POST',
    'callback'            => [ $this, 'update_settings' ],
    'permission_callback' => fn() => current_user_can( 'manage_options' ),
    'args'                => [
        'option_value' => [
            'required'          => true,
            'sanitize_callback' => 'sanitize_text_field',
            'validate_callback' => fn( $v ) => is_string( $v ) && strlen( $v ) <= 255,
        ],
    ],
] );

// Never do this:
'permission_callback' => '__return_true',  // ← open to the world
```

### Sanitization, Escaping, Validation

```php
// INPUT: sanitize on the way in
$title   = sanitize_text_field( wp_unslash( $_POST['title'] ?? '' ) );
$content = wp_kses_post( wp_unslash( $_POST['content'] ?? '' ) );
$email   = sanitize_email( $_POST['email'] ?? '' );
$url     = esc_url_raw( $_POST['redirect_url'] ?? '' );
$int_val = absint( $_POST['count'] ?? 0 );

// OUTPUT: escape on the way out (every echo)
echo esc_html( $title );
echo esc_attr( $attribute_value );
echo esc_url( $link );
echo wp_kses_post( $rich_content );
echo esc_js( $js_value );
```

### Database Queries — Prepared Statements Always

```php
global $wpdb;

// SAFE: use prepare() for every parameterized query
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT post_id, meta_value
         FROM {$wpdb->postmeta}
         WHERE meta_key = %s
           AND post_id IN (" . implode( ',', array_fill( 0, count( $ids ), '%d' ) ) . ")",
        array_merge( [ 'featured_image' ], array_map( 'intval', $ids ) )
    ),
    ARRAY_A
);

// Schema changes: always use dbDelta()
function my_plugin_create_tables(): void {
    global $wpdb;
    $charset_collate = $wpdb->get_charset_collate();
    $sql = "CREATE TABLE {$wpdb->prefix}my_data (
        id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
        post_id     BIGINT UNSIGNED NOT NULL,
        score       DECIMAL(8,2) NOT NULL DEFAULT 0.00,
        created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (id),
        KEY idx_post_id (post_id)
    ) $charset_collate;";
    require_once ABSPATH . 'wp-admin/includes/upgrade.php';
    dbDelta( $sql );
}
```

### REST API Hardening (WordPress 7.0+ context)

```php
// Block user enumeration — a common first step in attacks
add_filter( 'rest_endpoints', function( array $endpoints ): array {
    if ( ! is_user_logged_in() ) {
        unset( $endpoints['/wp/v2/users'] );
        unset( $endpoints['/wp/v2/users/(?P<id>[\d]+)'] );
    }
    return $endpoints;
} );

// Rate-limit authentication endpoints at the server level (nginx/Apache),
// not just application level — WordPress itself has no built-in rate limiting

// Disable XML-RPC if not needed (massive attack surface)
add_filter( 'xmlrpc_enabled', '__return_false' );

// Abilities API endpoints (7.0+) need the same protection patterns as any route
// Always supply a permission_callback — do not assume the AI framework handles it
```

---

## WP_Query at Scale (1M+ Posts)

This is the most important section for large WordPress sites. The wrong query
against a million-post database does not just run slowly — it can take the
server down entirely.

### The Golden Rules

```php
// NEVER do this — fetches every post in the database
$query = new WP_Query( [ 'posts_per_page' => -1 ] );

// NEVER use offset for pagination — it forces MySQL to scan and discard rows
$query = new WP_Query( [ 'offset' => 50000, 'posts_per_page' => 20 ] );

// ALWAYS set posts_per_page explicitly (hard cap in your code, not just UI)
// ALWAYS specify post_type and post_status — allows index use
// ALWAYS use keyset pagination for deep paging (see below)
```

### Disable Unnecessary Sub-Queries

A default `WP_Query` fires up to 5 queries. Strip out what you don't need:

```php
$query = new WP_Query( [
    'post_type'              => 'post',
    'post_status'            => 'publish',
    'posts_per_page'         => 20,
    // Remove the COUNT(*) query — only set false when you don't need pagination
    'no_found_rows'          => true,
    // Skip priming the meta cache — only skip if you won't call get_post_meta()
    'update_post_meta_cache' => false,
    // Skip priming the term cache — only skip if you won't need term data
    'update_post_term_cache' => false,
    // Return only IDs when you just need IDs — 10x less data transferred
    'fields'                 => 'ids',
] );
```

### Avoid `post__not_in` at Scale

```php
// BAD at scale — forces a full index scan and poor cache hit rates
$query = new WP_Query( [
    'post_type'   => 'post',
    'post__not_in' => $large_exclusion_array,
] );

// GOOD — filter after retrieval, or redesign the query entirely
// If you must exclude, keep the exclusion list tiny (< 10 IDs)
```

### Meta Queries — the Biggest Scale Killer

```php
// BAD: meta_value queries with no index support, scanning millions of rows
$query = new WP_Query( [
    'meta_query' => [
        [ 'key' => '_featured', 'value' => '1', 'compare' => '=' ],
        [ 'key' => '_score',    'value' => '80', 'compare' => '>' , 'type' => 'NUMERIC' ],
    ],
] );

// BETTER: filter by taxonomy or post field first (indexed), then meta
$query = new WP_Query( [
    'post_type'   => 'post',
    'post_status' => 'publish',
    'tax_query'   => [ [ 'taxonomy' => 'featured_content', 'field' => 'slug', 'terms' => 'yes' ] ],
    // Now meta_query operates on a much smaller result set
    'meta_query'  => [ [ 'key' => '_score', 'value' => '80', 'compare' => '>', 'type' => 'NUMERIC' ] ],
] );

// BEST for heavy meta filtering: custom table + two-step query
// Step 1: get IDs from your indexed custom table
global $wpdb;
$ids = $wpdb->get_col( $wpdb->prepare(
    "SELECT post_id FROM {$wpdb->prefix}post_scores
     WHERE score > %f AND is_featured = 1
     ORDER BY score DESC LIMIT 100",
    80.0
) );

// Step 2: fetch those posts via WP_Query (uses the primary key index)
$query = new WP_Query( [
    'post_type'           => 'post',
    'post_status'         => 'publish',
    'post__in'            => $ids,
    'orderby'             => 'post__in',
    'no_found_rows'       => true,
    'posts_per_page'      => count( $ids ),
] );
```

### Keyset Pagination (Required at Scale)

```php
// BAD at scale: OFFSET forces MariaDB to scan and discard N rows
$query = new WP_Query( [ 'paged' => 500, 'posts_per_page' => 20 ] );

// GOOD: keyset pagination — O(log n) regardless of depth
// Pass the last seen ID from the previous page
function get_posts_after( int $last_id, int $per_page = 20 ): WP_Query {
    global $wpdb;
    // Use direct SQL for true keyset efficiency; WP_Query can't do this natively
    $ids = $wpdb->get_col( $wpdb->prepare(
        "SELECT ID FROM {$wpdb->posts}
         WHERE post_type   = 'post'
           AND post_status = 'publish'
           AND ID < %d
         ORDER BY ID DESC
         LIMIT %d",
        $last_id,
        $per_page
    ) );

    return new WP_Query( [
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'post__in'       => $ids ?: [ 0 ],
        'orderby'        => 'post__in',
        'no_found_rows'  => true,
        'posts_per_page' => $per_page,
    ] );
}
```

### Taxonomy Queries at Scale

```php
// Add a date constraint when querying by taxonomy on large archives
// Without date_query, MariaDB must sort all matching posts before limiting
$query = new WP_Query( [
    'post_type'   => 'post',
    'post_status' => 'publish',
    'tax_query'   => [ [
        'taxonomy' => 'category',
        'field'    => 'term_id',
        'terms'    => $term_id,
    ] ],
    // Date constraint lets the optimizer use type_status_date index first
    'date_query'  => [ [ 'after' => '1 month ago' ] ],
    'posts_per_page' => 20,
    'no_found_rows'  => true,
] );

// Avoid include_children => true on deep term hierarchies — it generates
// a separate query for every child term
'tax_query' => [ [
    'taxonomy'         => 'category',
    'field'            => 'term_id',
    'terms'            => $term_ids,   // flatten the hierarchy yourself
    'include_children' => false,       // explicit, not default
] ],
```

---

## Caching Strategy

### Object Cache First

```php
// Always check cache before querying
function get_featured_post_ids(): array {
    $cache_key   = 'featured_post_ids_v1';
    $cache_group = 'my_plugin';

    $ids = wp_cache_get( $cache_key, $cache_group );
    if ( false !== $ids ) {
        return $ids;
    }

    global $wpdb;
    $ids = array_map( 'intval', $wpdb->get_col(
        "SELECT post_id FROM {$wpdb->prefix}featured_posts
         WHERE active = 1 ORDER BY priority ASC LIMIT 50"
    ) );

    // Cache for 5 minutes; invalidate in the save_post hook
    wp_cache_set( $cache_key, $ids, $cache_group, 5 * MINUTE_IN_SECONDS );
    return $ids;
}

// Invalidate cache when relevant data changes
add_action( 'save_post', function( int $post_id ): void {
    wp_cache_delete( 'featured_post_ids_v1', 'my_plugin' );
} );
```

### Transients — Use with Caution at Scale

```php
// Transients ARE autoloaded from wp_options unless they have a short TTL
// and Redis/Memcached is handling them. Without a persistent cache backend,
// every non-expiring transient loads on every request.

// SAFE: always set an expiry
set_transient( 'my_expensive_result', $result, HOUR_IN_SECONDS );

// NEVER create non-expiring transients
set_transient( 'my_data', $result );  // ← autoloaded forever — DO NOT DO THIS

// With Redis (persistent object cache), transients are stored in Redis,
// NOT in wp_options — this is the preferred setup for large sites
```

### Fragment Caching

```php
function render_popular_posts(): string {
    $cache_key = 'popular_posts_html_v2';
    $html = wp_cache_get( $cache_key, 'my_plugin' );
    if ( false !== $html ) {
        return $html;
    }

    ob_start();
    // ... expensive template render ...
    $html = ob_get_clean();

    wp_cache_set( $cache_key, $html, 'my_plugin', 10 * MINUTE_IN_SECONDS );
    return $html;
}
```

---

## Custom Post Types & Taxonomies

```php
// Register CPT with performance in mind
add_action( 'init', function(): void {
    register_post_type( 'product', [
        'public'       => true,
        'label'        => __( 'Products', 'my-plugin' ),
        'supports'     => [ 'title', 'editor', 'thumbnail', 'custom-fields' ],
        'show_in_rest' => true,   // required for block editor
        'has_archive'  => true,
        'rewrite'      => [ 'slug' => 'products', 'with_front' => false ],
        // Only register capabilities you actually check
        'capability_type' => 'post',
        'map_meta_cap'    => true,
    ] );

    register_taxonomy( 'product_cat', 'product', [
        'public'            => true,
        'hierarchical'      => true,
        'show_in_rest'      => true,
        'rewrite'           => [ 'slug' => 'product-category' ],
    ] );
} );
```

---

## PHP-Only Block Registration & WP AI Client (WordPress 7.0+)

> **Read `references/block-editor.md`** for full block registration patterns
> (PHP-only and full JS), Interactivity API migration from pre-7.0, iframed
> editor postMessage patterns, Block Bindings API, WP AI Client / Abilities API,
> Connectors API, and DataViews JS integration.

Key principles:

- `register_block_type()` with `render_callback` and no editor script generates
  Inspector controls automatically for `integer`, `boolean`, `string` (with `enum`),
  and colour attribute types — no JavaScript required
- `wp_ai_client_prompt()` is provider-agnostic; users configure their provider
  in `Settings > Connectors` — your plugin never holds API keys
- Always cache AI responses (`wp_cache_set`) — calls are expensive and slow
- The block editor is **always iframed in 7.0+**; use `postMessage` for
  cross-frame communication and always validate `event.origin`
- `register_wp_ability()` exposes actions to AI agents — treat them exactly
  like REST endpoints with a real `permission_callback`

---

## Background Processing

```php
// Register a custom cron schedule
add_filter( 'cron_schedules', function( array $schedules ): array {
    $schedules['every_five_minutes'] = [
        'interval' => 5 * MINUTE_IN_SECONDS,
        'display'  => __( 'Every 5 Minutes', 'my-plugin' ),
    ];
    return $schedules;
} );

// Schedule the event once (call from plugin activation)
function my_plugin_schedule_jobs(): void {
    if ( ! wp_next_scheduled( 'my_plugin_process_queue' ) ) {
        wp_schedule_event( time(), 'every_five_minutes', 'my_plugin_process_queue' );
    }
}
register_activation_hook( __FILE__, 'my_plugin_schedule_jobs' );

// Process in small batches — never process unlimited records in cron
add_action( 'my_plugin_process_queue', function(): void {
    global $wpdb;
    $batch = $wpdb->get_results( $wpdb->prepare(
        "SELECT id, post_id FROM {$wpdb->prefix}my_queue
         WHERE status = 'pending'
         ORDER BY id ASC
         LIMIT %d",
        50   // never remove this limit
    ) );
    foreach ( $batch as $item ) {
        // process each item
        $wpdb->update(
            "{$wpdb->prefix}my_queue",
            [ 'status' => 'done' ],
            [ 'id' => (int) $item->id ],
            [ '%s' ],
            [ '%d' ]
        );
    }
} );

// Clean up on deactivation
register_deactivation_hook( __FILE__, function(): void {
    wp_clear_scheduled_hook( 'my_plugin_process_queue' );
} );
```

---

## Asset Enqueueing

```php
add_action( 'wp_enqueue_scripts', function(): void {
    // Only load on pages that need it
    if ( ! is_singular( 'product' ) ) {
        return;
    }

    wp_enqueue_style(
        'my-plugin-product',
        MY_PLUGIN_URL . 'assets/css/product.css',
        [],
        MY_PLUGIN_VERSION
    );

    wp_enqueue_script(
        'my-plugin-product',
        MY_PLUGIN_URL . 'assets/js/product.js',
        [ 'jquery' ],
        MY_PLUGIN_VERSION,
        [ 'strategy' => 'defer', 'in_footer' => true ]  // 7.0+ loading strategy
    );

    // Pass data to JS securely
    wp_add_inline_script(
        'my-plugin-product',
        'const myPlugin = ' . wp_json_encode( [
            'nonce'  => wp_create_nonce( 'my_product_action' ),
            'ajaxUrl'=> admin_url( 'admin-ajax.php' ),
            'postId' => get_the_ID(),
        ] ),
        'before'
    );
} );
```

---

## Internationalization

```php
// Load text domain
add_action( 'init', function(): void {
    load_plugin_textdomain( 'my-plugin', false, dirname( plugin_basename( __FILE__ ) ) . '/languages/' );
} );

// All user-facing strings must be wrapped
$message = sprintf(
    /* translators: %s: post title */
    __( 'Post "%s" has been updated.', 'my-plugin' ),
    esc_html( $post_title )
);

// Plurals
$label = sprintf(
    /* translators: %d: number of posts */
    _n( '%d post found.', '%d posts found.', $count, 'my-plugin' ),
    number_format_i18n( $count )
);
```

---

## Testing

```php
// PHPUnit with WordPress test suite (WP_UnitTestCase)
class My_Plugin_Query_Test extends WP_UnitTestCase {

    public function test_get_featured_post_ids_returns_array(): void {
        // Arrange: create test posts
        $post_id = self::factory()->post->create( [ 'post_status' => 'publish' ] );
        update_post_meta( $post_id, '_featured', '1' );

        // Act
        $ids = get_featured_post_ids();

        // Assert
        $this->assertIsArray( $ids );
        $this->assertContainsOnly( 'int', $ids );
    }

    public function test_nonce_verification_rejects_invalid_nonce(): void {
        $_POST['nonce'] = 'invalid_nonce';
        $result = check_ajax_referer( 'my_action', 'nonce', false );
        $this->assertFalse( $result );
    }
}
```

---

## DataViews (WordPress 7.0+)

DataViews replaces `WP_List_Table` on Posts, Pages, and Media screens.
Plugins that add columns via `manage_{$post_type}_posts_columns` or inject
custom admin CSS targeting `.wp-list-table` will need updates.

```php
// Legacy approach (still works but not shown in DataViews screens):
add_filter( 'manage_post_posts_columns', function( array $cols ): array {
    $cols['my_score'] = __( 'Score', 'my-plugin' );
    return $cols;
} );

// For DataViews compatibility, register your data via the REST API instead
// and use the DataViews/DataForm JS components to render custom columns.
// See references/block-editor.md for JavaScript patterns.
```

---

## Reference Files

Load these when the task goes deeper than the summaries above:

- **`references/query-patterns.md`** — advanced `WP_Query` patterns, custom
  table strategies, and batch processing for 1M+ post environments
- **`references/security.md`** — full security checklist, hardening recipes,
  REST API lockdown, and XML-RPC/user-enumeration mitigations
- **`references/block-editor.md`** — block development (PHP-only + full JS),
  Interactivity API patterns, DataViews integration, Abilities API, WP AI Client
- **`references/deployment.md`** — WP-CLI workflows, staging-to-production
  process, wp-config.php hardening, and `wp-config.php` constants for large sites

> For MariaDB-level work (index DDL, `mariadb.cnf`, replication, backups),
> use the **mariadb-best-practices** skill instead.

*Last Updated: 2026-07-28*
