# WordPress + MariaDB 10.11: Deep Reference

> Load this file when the user's task is specifically about WordPress database
> performance, optimization, or schema work at scale.

---

## Core Table Overview

WordPress uses 12 default tables. The chronic troublemakers at scale:

| Table | Problem | Impact |
|-------|---------|--------|
| `wp_posts` | Stores revisions, attachments, CPTs alongside posts | Grows unbounded without `WP_POST_REVISIONS` limit |
| `wp_postmeta` | EAV schema — one row per metadata key/value | 1M posts = potentially 10M+ rows |
| `wp_options` | All autoloaded options fire on every request | Single bloated row can stall every page |
| `wp_comments` | Spam accumulates even when marked spam | Full table scans on unindexed sorts |
| `wp_usermeta` | Plugin data piles up per user | Similar EAV problems as postmeta |

---

## Step 1: Diagnose Before Touching Anything

```sql
-- Table sizes and row counts for all WP tables
SELECT TABLE_NAME,
       ENGINE,
       TABLE_COLLATION,
       FORMAT(TABLE_ROWS, 0)                              AS approx_rows,
       ROUND(DATA_LENGTH / 1024 / 1024, 2)               AS data_mb,
       ROUND(INDEX_LENGTH / 1024 / 1024, 2)              AS index_mb,
       ROUND(DATA_FREE / 1024 / 1024, 2)                 AS fragmented_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME LIKE 'wp_%'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;

-- Total autoloaded payload (target: < 800 KB)
SELECT ROUND(SUM(LENGTH(option_value)) / 1024, 2) AS total_autoload_kb,
       COUNT(*)                                     AS autoload_count
FROM wp_options
WHERE autoload = 'yes';

-- Top 20 autoloaded offenders
SELECT option_name,
       autoload,
       ROUND(LENGTH(option_value) / 1024, 2) AS size_kb
FROM wp_options
WHERE autoload = 'yes'
ORDER BY LENGTH(option_value) DESC
LIMIT 20;

-- How many orphaned postmeta rows exist?
SELECT COUNT(*) AS orphan_postmeta
FROM wp_postmeta pm
LEFT JOIN wp_posts p ON pm.post_id = p.ID
WHERE p.ID IS NULL;

-- Engine check — all core tables should be InnoDB
SELECT TABLE_NAME, ENGINE
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME IN ('wp_posts','wp_postmeta','wp_options',
                     'wp_comments','wp_commentmeta','wp_usermeta',
                     'wp_users','wp_terms','wp_termmeta',
                     'wp_term_relationships','wp_term_taxonomy',
                     'wp_links')
ORDER BY TABLE_NAME;
```

---

## Step 2: Add Critical Indexes

Always test on a staging clone first. Run during off-peak hours.
MariaDB 10.3+ `ALGORITHM=INPLACE, LOCK=NONE` is truly online.

```sql
-- wp_posts
ALTER TABLE wp_posts
    ADD INDEX IF NOT EXISTS idx_type_status_date  (post_type, post_status, post_date),
    ADD INDEX IF NOT EXISTS idx_author_status      (post_author, post_status),
    ADD INDEX IF NOT EXISTS idx_parent_type        (post_parent, post_type),
    ALGORITHM=INPLACE, LOCK=NONE;

-- wp_postmeta (the single biggest win on large sites)
-- NOTE: WP core already ships `KEY meta_key (meta_key(191))` and `KEY post_id
-- (post_id)`. Check `SHOW INDEX FROM wp_postmeta` before adding this — it's
-- only worth the extra write overhead if you actually query on meta_value.
ALTER TABLE wp_postmeta
    ADD INDEX IF NOT EXISTS idx_meta_key_value (meta_key, meta_value(100)),
    ALGORITHM=INPLACE, LOCK=NONE;

-- wp_options (fixes full-table-scan on every page load)
-- NOTE: WordPress 6.6+ already ships an index on `autoload` and expanded its
-- values beyond yes/no. Run `SHOW INDEX FROM wp_options` first — don't add a
-- duplicate. Also, a single-column index on a 2-3 value column has low
-- cardinality; the optimizer may still choose a full table scan over it when
-- most rows are autoload='yes'. Cleaning up autoload bloat (Step 3) matters
-- more than the index itself.
ALTER TABLE wp_options
    ADD INDEX IF NOT EXISTS idx_autoload (autoload),
    ALGORITHM=INPLACE, LOCK=NONE;

-- wp_comments
ALTER TABLE wp_comments
    ADD INDEX IF NOT EXISTS idx_post_approved (comment_post_ID, comment_approved),
    ADD INDEX IF NOT EXISTS idx_approved_date  (comment_approved, comment_date_gmt),
    ALGORITHM=INPLACE, LOCK=NONE;

-- wp_usermeta
ALTER TABLE wp_usermeta
    ADD INDEX IF NOT EXISTS idx_meta_key_value (meta_key, meta_value(100)),
    ALGORITHM=INPLACE, LOCK=NONE;

-- Full-text on posts (enables MATCH/AGAINST instead of LIKE '%...%')
ALTER TABLE wp_posts
    ADD FULLTEXT INDEX IF NOT EXISTS ft_post_search (post_title, post_content, post_excerpt);
```

---

## Step 3: Clean Autoloaded Options

```sql
-- Safe: remove expired transients
DELETE FROM wp_options
WHERE option_name LIKE '_transient_timeout_%'
  AND option_value < UNIX_TIMESTAMP();

-- Clean up orphaned transient values (the timeout row was already deleted)
DELETE o FROM wp_options o
LEFT JOIN wp_options t
    ON t.option_name = REPLACE(o.option_name, '_transient_', '_transient_timeout_')
WHERE o.option_name LIKE '_transient_%'
  AND NOT o.option_name LIKE '_transient_timeout_%'
  AND t.option_name IS NULL;

-- Same for site transients (multisite)
DELETE FROM wp_options
WHERE option_name LIKE '_site_transient_timeout_%'
  AND option_value < UNIX_TIMESTAMP();

-- Disable autoload on known-safe large values
-- Research each option_name before running!
UPDATE wp_options
SET autoload = 'no'
WHERE option_name IN (
    'rewrite_rules',          -- rebuilds on demand; safe to disable autoload
    'widget_block',           -- only needed on pages that display widgets
    'akismet_alert_data'      -- example plugin data
)
AND autoload = 'yes';
```

---

## Step 4: Orphan and Revision Cleanup

**Always back up first. Run in batches during low-traffic windows.**

```sql
-- Delete orphaned postmeta in 10,000-row batches
-- Repeat until "0 rows affected"
-- NOTE: multi-table DELETE (DELETE t1 FROM t1 JOIN t2 ...) does NOT support
-- LIMIT/ORDER BY in MariaDB/MySQL — it's a syntax error. Batch via a subquery
-- against the single target table instead:
DELETE FROM wp_postmeta
WHERE meta_id IN (
    SELECT meta_id FROM (
        SELECT pm.meta_id
        FROM wp_postmeta pm
        LEFT JOIN wp_posts p ON pm.post_id = p.ID
        WHERE p.ID IS NULL
        LIMIT 10000
    ) AS batch
);

-- Orphaned commentmeta
DELETE FROM wp_commentmeta
WHERE meta_id IN (
    SELECT meta_id FROM (
        SELECT cm.meta_id
        FROM wp_commentmeta cm
        LEFT JOIN wp_comments c ON cm.comment_id = c.comment_ID
        WHERE c.comment_ID IS NULL
        LIMIT 10000
    ) AS batch
);

-- Post revisions older than 90 days (set WP_POST_REVISIONS = 3 in wp-config.php first)
DELETE FROM wp_posts
WHERE post_type = 'revision'
  AND post_date < NOW() - INTERVAL 90 DAY
LIMIT 5000;

-- Spam and trash comments
DELETE FROM wp_comments
WHERE (comment_approved = 'spam' OR comment_approved = 'trash')
  AND comment_date_gmt < NOW() - INTERVAL 30 DAY
LIMIT 10000;

-- Orphaned commentmeta from deleted comments (run after above)
DELETE FROM wp_commentmeta
WHERE meta_id IN (
    SELECT meta_id FROM (
        SELECT cm.meta_id
        FROM wp_commentmeta cm
        LEFT JOIN wp_comments c ON cm.comment_id = c.comment_ID
        WHERE c.comment_ID IS NULL
        LIMIT 10000
    ) AS batch
);
```

---

## Step 5: Convert MyISAM Tables to InnoDB

```sql
-- Generate ALTER statements for any remaining MyISAM tables
SELECT CONCAT('ALTER TABLE `', TABLE_NAME,
              '` ENGINE=InnoDB ROW_FORMAT=DYNAMIC;') AS sql_statement
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND ENGINE = 'MyISAM'
  AND TABLE_NAME LIKE 'wp_%';

-- Run each generated statement during low-traffic window
-- Example:
ALTER TABLE wp_posts ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
```

---

## Step 6: Reclaim Fragmented Space

After large deletes, InnoDB retains the freed space as internal free pages.
`OPTIMIZE TABLE` rebuilds the table and reclaims space (locks table briefly).
For large tables, use `ALTER TABLE` with `ALGORITHM=INPLACE` instead.

```sql
-- After cleanup, reclaim space (causes brief metadata lock)
OPTIMIZE TABLE wp_options;
OPTIMIZE TABLE wp_postmeta;   -- may take minutes on 10M+ rows

-- Online alternative for large tables (reduced locking)
ALTER TABLE wp_posts      ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE wp_postmeta   ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE wp_comments   ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;

-- Update optimizer statistics after major changes
ANALYZE TABLE wp_posts, wp_postmeta, wp_options, wp_comments,
             wp_usermeta, wp_term_relationships;
```

---

## Ongoing Maintenance Queries

```sql
-- Monthly: check for new autoload bloat creep
SELECT ROUND(SUM(LENGTH(option_value)) / 1024, 2) AS autoload_kb
FROM wp_options WHERE autoload = 'yes';

-- Monthly: slow-growing postmeta (spot plugin bloat)
SELECT meta_key,
       COUNT(*)                                    AS row_count,
       ROUND(SUM(LENGTH(meta_value)) / 1024, 2)   AS total_kb
FROM wp_postmeta
GROUP BY meta_key
ORDER BY total_kb DESC
LIMIT 20;

-- Find queries not using indexes (requires slow_query_log AND log_output=TABLE;
-- the recommended config in references/config-and-admin.md uses FILE output,
-- so this table will be empty unless you also set log_output=TABLE, or use
-- mysqldumpslow/pt-query-digest against slow_query_log_file instead)
SELECT * FROM mysql.slow_log
WHERE sql_text NOT LIKE '%SLEEP%'
ORDER BY query_time DESC
LIMIT 20;
```

---

## wp-config.php Settings That Affect the Database

```php
// Limit post revisions to prevent wp_posts bloat
define('WP_POST_REVISIONS', 3);

// Disable revisions entirely (not recommended for editorial sites)
// define('WP_POST_REVISIONS', false);

// Automatically empty trash every 7 days
define('EMPTY_TRASH_DAYS', 7);

// Use persistent object cache (Redis recommended for 1M+ post sites)
// Requires Redis server + object-cache.php drop-in (e.g. from Predis or Redis Object Cache plugin)
define('WP_REDIS_HOST', '127.0.0.1');
define('WP_REDIS_PORT', 6379);
define('WP_REDIS_DATABASE', 0);
define('WP_CACHE', true);
```

---

## Object Caching (Critical at 1M+ Posts)

A persistent object cache (Redis or Memcached) is the highest-impact
optimization for large WordPress sites — it intercepts `wp_options` autoload
queries and postmeta lookups before they hit MariaDB at all.

With Redis + the Redis Object Cache plugin:
- Autoloaded options are fetched once per cache TTL, not per request
- Repeated `get_post_meta()` calls skip the DB entirely
- Database connection count drops dramatically under traffic spikes

Configure `innodb_buffer_pool_size` to hold your working set in RAM (index
data + frequently accessed rows). On a 1M-post site, the `wp_postmeta` index
alone can be several GB.

---

## WP-CLI Database Commands

```bash
# Check database size
wp db size --tables

# Run a SQL query
wp db query "SELECT COUNT(*) FROM wp_posts WHERE post_status = 'publish'"

# Export (single-transaction, compressed)
wp db export --add-drop-table - | gzip > backup.sql.gz

# Optimize all tables
wp db optimize

# Find and replace (handles serialized data correctly)
wp search-replace 'http://old-domain.com' 'https://new-domain.com' --dry-run
wp search-replace 'http://old-domain.com' 'https://new-domain.com'

# Delete all expired transients
wp transient delete --expired --all
```
*Last Updated: 2026-08-03*
