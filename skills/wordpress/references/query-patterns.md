# WordPress Query Patterns at Scale (1M+ Posts)

> Load this file when the task involves complex WP_Query scenarios, custom table
> strategies, batch operations, or diagnosing slow database queries on large sites.

---

## Diagnosing Slow Queries

Before optimizing, measure. Enable query logging in `wp-config.php` on staging:

```php
// wp-config.php (staging only — never production)
define( 'SAVEQUERIES', true );

// Then in a template or shutdown hook:
add_action( 'shutdown', function(): void {
    if ( ! defined( 'SAVEQUERIES' ) || ! SAVEQUERIES ) {
        return;
    }
    global $wpdb;
    $slow = array_filter( $wpdb->queries, fn( $q ) => $q[1] > 0.05 ); // > 50ms
    if ( $slow ) {
        error_log( 'SLOW WP QUERIES: ' . print_r( $slow, true ) );
    }
} );
```

For production, use the MariaDB slow query log (`long_query_time = 1` in
`mariadb.cnf`) — see the **mariadb-best-practices** skill.

---

## The WP_Query Sub-Query Map

Every default `WP_Query` fires multiple queries. Know which to disable:

| Argument | What it removes | When safe to disable |
|----------|----------------|----------------------|
| `'no_found_rows' => true` | `SELECT COUNT(*)` for pagination | When you don't need `$query->found_posts` |
| `'update_post_meta_cache' => false` | Primes meta cache for all returned posts | When you won't call `get_post_meta()` |
| `'update_post_term_cache' => false` | Primes term cache for all returned posts | When you won't access term data |
| `'fields' => 'ids'` | Fetches only IDs, not full post objects | When you only need IDs |
| `'cache_results' => false` | Skips storing results in object cache | For one-off admin queries only |

---

## Pattern: ID-First Queries

Fetch IDs cheaply, then hydrate only what you need:

```php
// Step 1: get IDs with minimal overhead
$ids = ( new WP_Query( [
    'post_type'              => 'post',
    'post_status'            => 'publish',
    'posts_per_page'         => 20,
    'fields'                 => 'ids',
    'no_found_rows'          => true,
    'update_post_meta_cache' => false,
    'update_post_term_cache' => false,
] ) )->posts;

// Step 2: batch-prime caches for only the fields you actually need
update_postmeta_cache( $ids );       // primes meta for all IDs in one query
update_object_term_cache( $ids, 'post' );  // primes terms in one query

// Step 3: use cached data — no per-post DB hits
foreach ( $ids as $id ) {
    $title     = get_the_title( $id );           // from post cache
    $thumbnail = get_post_thumbnail_id( $id );   // from meta cache (primed above)
    $cats      = get_the_category( $id );        // from term cache (primed above)
}
```

---

## Pattern: Custom Table for High-Cardinality Meta

When a meta key exists on millions of posts and you filter/sort by it,
move it to its own indexed table. `wp_postmeta` is an EAV table —
it was not designed for indexed range queries on large datasets.

```php
// Plugin activation: create the custom table
function my_plugin_create_scores_table(): void {
    global $wpdb;
    $charset_collate = $wpdb->get_charset_collate();
    $sql = "CREATE TABLE IF NOT EXISTS {$wpdb->prefix}post_scores (
        post_id    BIGINT UNSIGNED NOT NULL,
        score      DECIMAL(8,2)   NOT NULL DEFAULT 0.00,
        updated_at DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP
                                           ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (post_id),
        KEY idx_score (score)
    ) {$charset_collate};";
    require_once ABSPATH . 'wp-admin/includes/upgrade.php';
    dbDelta( $sql );
}
register_activation_hook( MY_PLUGIN_FILE, 'my_plugin_create_scores_table' );

// Write to both tables (keep wp_postmeta in sync for plugin compatibility)
function my_plugin_update_score( int $post_id, float $score ): void {
    global $wpdb;
    $wpdb->replace(
        "{$wpdb->prefix}post_scores",
        [ 'post_id' => $post_id, 'score' => $score ],
        [ '%d', '%f' ]
    );
    // Keep meta in sync for any code that reads it directly
    update_post_meta( $post_id, '_score', $score );
}

// Two-step query: custom table → WP_Query
function get_top_scoring_posts( float $min_score, int $limit = 20 ): array {
    // Check cache first
    $cache_key = "top_posts_{$min_score}_{$limit}_v1";
    $cached = wp_cache_get( $cache_key, 'my_plugin' );
    if ( false !== $cached ) {
        return $cached;
    }

    global $wpdb;
    $ids = $wpdb->get_col( $wpdb->prepare(
        "SELECT post_id FROM {$wpdb->prefix}post_scores
         WHERE score >= %f
         ORDER BY score DESC
         LIMIT %d",
        $min_score,
        $limit
    ) );

    if ( empty( $ids ) ) {
        return [];
    }

    $posts = ( new WP_Query( [
        'post_type'              => 'post',
        'post_status'            => 'publish',
        'post__in'               => $ids,
        'orderby'                => 'post__in',
        'posts_per_page'         => count( $ids ),
        'no_found_rows'          => true,
        'update_post_term_cache' => false,
    ] ) )->posts;

    wp_cache_set( $cache_key, $posts, 'my_plugin', 5 * MINUTE_IN_SECONDS );
    return $posts;
}
```

---

## Pattern: Taxonomy-Driven Architecture

For large sites, taxonomies outperform meta queries for filtering because term
relationships are in their own indexed table. Design for this:

```php
// Use taxonomies for filterable attributes, not meta
// BAD at scale: filtering by meta key
$query = new WP_Query( [ 'meta_key' => '_color', 'meta_value' => 'red' ] );

// GOOD: filtering by taxonomy term (uses term_relationships index)
$query = new WP_Query( [
    'post_type'   => 'product',
    'post_status' => 'publish',
    'tax_query'   => [ [
        'taxonomy'         => 'product_color',
        'field'            => 'slug',
        'terms'            => 'red',
        'include_children' => false,
    ] ],
    'no_found_rows'  => true,
    'posts_per_page' => 24,
] );

// Flatten term hierarchies before querying — never rely on include_children
// on trees deeper than 2 levels on large datasets
function get_term_tree_ids( int $term_id, string $taxonomy ): array {
    $children = get_term_children( $term_id, $taxonomy );
    return array_merge( [ $term_id ], is_array( $children ) ? $children : [] );
}
```

---

## Pattern: Safe Batch Processing

Never process unlimited records in a single request or cron job:

```php
/**
 * Process posts in batches with a cursor — safe for 1M+ post tables.
 *
 * @param callable $callback Receives a WP_Post. Return false to stop early.
 * @param int      $batch_size Number of posts per batch (keep ≤ 100).
 */
function my_plugin_batch_posts( callable $callback, int $batch_size = 50 ): void {
    $last_id = 0;

    do {
        global $wpdb;
        $ids = array_map( 'intval', $wpdb->get_col( $wpdb->prepare(
            "SELECT ID FROM {$wpdb->posts}
             WHERE post_type   = 'post'
               AND post_status = 'publish'
               AND ID > %d
             ORDER BY ID ASC
             LIMIT %d",
            $last_id,
            $batch_size
        ) ) );

        if ( empty( $ids ) ) {
            break;
        }

        $posts = ( new WP_Query( [
            'post_type'              => 'post',
            'post_status'            => 'publish',
            'post__in'               => $ids,
            'orderby'                => 'ID',
            'order'                  => 'ASC',
            'posts_per_page'         => count( $ids ),
            'no_found_rows'          => true,
            'update_post_term_cache' => false,
        ] ) )->posts;

        foreach ( $posts as $post ) {
            $continue = $callback( $post );
            if ( false === $continue ) {
                return;
            }
        }

        $last_id = (int) end( $ids );

        // Prevent memory exhaustion on large runs
        wp_cache_flush_group( 'posts' );
        wp_cache_flush_group( 'post_meta' );

    } while ( count( $ids ) === $batch_size );
}
```

---

## Pattern: Counting Without COUNT(*)

`found_posts` fires a second `SELECT COUNT(*)` on the full result set.
On large tables this is expensive. Use these alternatives:

```php
// For "do any results exist?" — use EXISTS logic
global $wpdb;
$has_results = (bool) $wpdb->get_var( $wpdb->prepare(
    "SELECT EXISTS(
        SELECT 1 FROM {$wpdb->posts}
        WHERE post_type = %s AND post_status = 'publish'
        LIMIT 1
    )",
    'product'
) );

// For approximate counts (fast, good enough for display)
$approx = (int) $wpdb->get_var( $wpdb->prepare(
    "SELECT TABLE_ROWS FROM information_schema.TABLES
     WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = %s",
    $wpdb->posts
) );

// For exact counts on specific post types: maintain a counter in options/cache
// rather than computing it on every request
function my_plugin_get_post_count( string $post_type ): int {
    $cache_key = "post_count_{$post_type}";
    $count = wp_cache_get( $cache_key, 'my_plugin' );
    if ( false !== $count ) {
        return (int) $count;
    }
    $counts = wp_count_posts( $post_type );
    $count  = (int) ( $counts->publish ?? 0 );
    wp_cache_set( $cache_key, $count, 'my_plugin', 10 * MINUTE_IN_SECONDS );
    return $count;
}
```

---

## Pattern: Avoiding Duplicate Queries

A common source of hidden N+1 problems in WordPress:

```php
// BAD: get_posts() re-queries when called on a WP_Query object's posts property
$query = new WP_Query( $args );
$posts = get_posts( $args );  // fires a second identical query

// GOOD: use the posts property directly
$query = new WP_Query( $args );
$posts = $query->posts;

// BAD: calling get_post_meta() inside a loop without priming the cache
foreach ( $post_ids as $id ) {
    $meta = get_post_meta( $id, '_my_key', true );  // N DB queries
}

// GOOD: prime once, then read from cache
update_postmeta_cache( $post_ids );
foreach ( $post_ids as $id ) {
    $meta = get_post_meta( $id, '_my_key', true );  // 0 DB queries
}
```

---

## Pattern: Search at Scale

WordPress's built-in `s` parameter in `WP_Query` generates a `LIKE '%term%'`
query — full table scan on large datasets.

```php
// BAD at scale: triggers LIKE '%..%' on post_title and post_content
$query = new WP_Query( [ 's' => 'search term' ] );

// GOOD option 1: MariaDB FULLTEXT (requires fulltext index on wp_posts)
// See mariadb-best-practices skill for index creation
global $wpdb;
$ids = $wpdb->get_col( $wpdb->prepare(
    "SELECT ID FROM {$wpdb->posts}
     WHERE MATCH(post_title, post_content) AGAINST (%s IN BOOLEAN MODE)
       AND post_type   = 'post'
       AND post_status = 'publish'
     LIMIT 50",
    $search_term
) );
$query = new WP_Query( [
    'post__in'       => $ids ?: [ 0 ],
    'orderby'        => 'post__in',
    'posts_per_page' => count( $ids ),
    'no_found_rows'  => true,
] );

// GOOD option 2: Dedicated search engine (best for 1M+ posts)
// ElasticPress (Elasticsearch) or Algolia — offloads search entirely from MariaDB
// Integration hooks into 'posts_pre_query' to intercept WP_Query
add_filter( 'posts_pre_query', 'my_plugin_elasticsearch_intercept', 10, 2 );
```

---

## WP-CLI: Batch Operations

```bash
# Run a custom WP-CLI command for batch processing (doesn't time out like HTTP)
wp eval-file scripts/backfill-scores.php --url=https://example.com

# Export/import with minimal memory
wp export --post_type=post --post_status=publish --filename_format=export-{date}-{n}.xml

# Search-replace (handles serialized data correctly)
wp search-replace 'http://old.example.com' 'https://new.example.com' \
    --skip-columns=guid \
    --dry-run

# Regenerate all image sizes in batches
wp media regenerate --yes --batch-size=50

# Count posts by type
wp post list --post_type=post --post_status=publish --format=count
```
*Last Updated: 2026-07-28*
