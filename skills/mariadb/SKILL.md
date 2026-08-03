---
name: mariadb
description: >
  MariaDB 10.11+ best practices for schema design, security hardening, query
  optimization, WordPress scaling, and database administration. Use this skill
  whenever the user asks about MariaDB schema design, indexing, query tuning,
  security configuration, WordPress database optimization (wp_options, wp_postmeta,
  large post volumes), InnoDB/Aria engine decisions, replication, backups,
  my.cnf/mariadb.cnf tuning, user/role management, system-versioned tables,
  or general MariaDB DBA tasks. Also trigger when the user mentions MariaDB
  performance problems, slow queries, connection limits, or database bloat.
---

# MariaDB 10.11+ Best Practices

> Scope: MariaDB 10.11 LTS (supported to Feb 2028). Differences from MySQL 8.0
> are called out explicitly — do not assume MySQL and MariaDB behave identically.

---

## Key MariaDB 10.11 Distinctions

- **Query cache still exists in 10.11** but is **disabled by default** (`query_cache_type=OFF`) — it was never removed from MariaDB (unlike MySQL, which removed it in 8.0). Per MariaDB's own docs it "does not scale well in environments with high throughput on multi-core machines" because of a single global mutex that serializes cache access and gets invalidated on every write to a cached table. For a busy WordPress site (many cores, constant writes to `wp_options`/`wp_postmeta`), leave it off and use an application-level cache (Redis object cache) instead.
- **Aria storage engine** ships natively — preferred for internal temp tables and
  read-heavy non-transactional workloads. Benchmark for your workload rather than
  assuming a fixed speedup multiplier.
- **System-versioned tables** (temporal/history tables) are first-class citizens.
- **Invisible columns** are supported (10.3+), useful for schema migration.
- **INVISIBLE** index columns and **descending index** columns are supported.
- **Instant ADD COLUMN** for InnoDB reduces DDL locking (10.3+).
- **PUBLIC pseudo-role** (10.11): grants/revokes apply to all users at once.
- **password_reuse_check plugin** (10.7+): prevents password reuse.
- **ed25519 authentication plugin**: modern, more secure than `mysql_native_password`.
- **GTID support is on by default** in 10.11 (every transaction gets a GTID), but
  replication itself is not automatic — you still configure it explicitly with
  `CHANGE MASTER TO ... MASTER_USE_GTID=slave_pos`.
- Root uses **UNIX socket authentication** by default since 10.4 — no root password
  is the secure default, not a misconfiguration.
- **`mariadb-*` command aliases** (`mariadb-dump`, `mariadb-backup`) are preferred
  over the deprecated `mysql*` names.

---

## Security

### Authentication & Users

**Root is socket-authenticated by default since 10.4 — this is correct and secure.**
Do not set a root password "for security" — that would actually weaken security.

```sql
-- Verify socket auth is in place (default since 10.4)
SELECT user, host, plugin FROM mysql.user WHERE user = 'root';
-- Should show plugin = 'unix_socket' or 'mysql_native_password' (without a remote host)

-- Block root from remote entirely (confirm localhost only)
DELETE FROM mysql.user WHERE user = 'root' AND host != 'localhost';
FLUSH PRIVILEGES;

-- Drop anonymous users if present
DELETE FROM mysql.user WHERE user = '';
DROP DATABASE IF EXISTS test;
FLUSH PRIVILEGES;
```

**Use ed25519 for application users** (more secure than `mysql_native_password`):

```sql
-- Install ed25519 if not loaded
INSTALL SONAME 'auth_ed25519';

-- Create app user with ed25519 authentication
CREATE USER 'wp_app'@'127.0.0.1'
    IDENTIFIED VIA ed25519 USING PASSWORD('use-a-long-random-passphrase-here');

-- Grant minimal privileges for WordPress
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE TEMPORARY TABLES
    ON wordpress_db.* TO 'wp_app'@'127.0.0.1';

-- Read-only replica user
CREATE USER 'wp_readonly'@'127.0.0.1'
    IDENTIFIED VIA ed25519 USING PASSWORD('another-long-passphrase');
GRANT SELECT ON wordpress_db.* TO 'wp_readonly'@'127.0.0.1';

FLUSH PRIVILEGES;
```

### Role-Based Access (10.11 PUBLIC Pseudo-Role)

```sql
-- Create roles
CREATE ROLE 'app_write', 'app_read', 'dba_role';

-- Grant privileges to roles
GRANT SELECT, INSERT, UPDATE, DELETE ON wordpress_db.* TO 'app_write';
GRANT SELECT ON wordpress_db.* TO 'app_read';
GRANT ALL ON *.* TO 'dba_role' WITH GRANT OPTION;

-- Assign roles to users
GRANT 'app_write' TO 'wp_app'@'127.0.0.1';
GRANT 'app_read' TO 'wp_readonly'@'127.0.0.1';

-- Set default role (user must activate role on connect otherwise)
SET DEFAULT ROLE 'app_write' FOR 'wp_app'@'127.0.0.1';

-- PUBLIC pseudo-role (10.11): grant to all current and future users
-- Use carefully — applies to every user
GRANT SELECT ON wordpress_db.options_public TO PUBLIC;
```

### Password Policy

```sql
-- Enable password validation plugin
INSTALL SONAME 'simple_password_check';

-- Enforce reuse protection (10.7+)
INSTALL SONAME 'password_reuse_check';

-- Configure in mariadb.cnf:
-- [mariadb]
-- plugin_load_add = simple_password_check
-- simple_password_check_minimum_length = 16
-- plugin_load_add = password_reuse_check
-- password_reuse_check_interval = 365   # days before password can be reused

-- Password expiry is appropriate for human/DBA accounts, NOT for application
-- service accounts like wp_app — there's no human to rotate the password, and
-- WordPress will start throwing DB connection errors in production the moment
-- it expires. Only apply PASSWORD EXPIRE if you have automated credential
-- rotation wired into wp-config.php deployment.
ALTER USER 'dba_readonly'@'localhost' PASSWORD EXPIRE INTERVAL 180 DAY;
```

### TLS / Encryption in Transit

Require TLS for all non-socket connections:

```sql
-- Require TLS for a user
ALTER USER 'wp_app'@'%' REQUIRE SSL;

-- Or require a specific issuer/subject (stronger)
ALTER USER 'wp_app'@'%'
    REQUIRE SUBJECT '/CN=wp-app/O=MyOrg'
    AND ISSUER '/CN=MyCA/O=MyOrg';
```

In `mariadb.cnf`:

```ini
[mariadb]
ssl_cert   = /etc/mysql/ssl/server-cert.pem
ssl_key    = /etc/mysql/ssl/server-key.pem
ssl_ca     = /etc/mysql/ssl/ca-cert.pem
# Enforce TLS for all remote connections
require_secure_transport = ON
tls_version = TLSv1.2,TLSv1.3
ssl_cipher  = ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
```

> **WordPress caveat**: `require_secure_transport` only applies to TCP
> connections, not UNIX sockets. If DB_HOST is `localhost`, PHP's mysqli talks
> over the socket and is unaffected. But `127.0.0.1` (or any remote host) is a
> TCP connection — enforcing this without also configuring `MYSQL_CLIENT_FLAGS`
> / `mysqli_ssl_set()` (via a drop-in or `WP_CONFIG`) will break the site's DB
> connection outright. Verify how `DB_HOST` is set before enabling this globally.

### Binary Log & At-Rest Encryption

```ini
[mariadb]
# Encrypt binary logs (10.5+) — critical for replication security
encrypt_binlog = ON

# Encrypt InnoDB tablespaces at rest
innodb_encrypt_tables       = ON
innodb_encrypt_log          = ON
innodb_encryption_threads   = 4
plugin_load_add             = file_key_management
file_key_management_filename = /etc/mysql/encryption/keyfile
file_key_management_filekey  = /etc/mysql/encryption/keyfile.key
```

### Audit Logging

```ini
[mariadb]
plugin_load_add = server_audit
server_audit_logging        = ON
server_audit_events         = CONNECT,QUERY_DDL,QUERY_DML_NO_SELECT
server_audit_file_path      = /var/log/mysql/audit.log
server_audit_file_rotate_size = 1073741824  # 1GB
server_audit_file_rotations = 10
```

---

## Schema Design

### Storage Engine Choices

| Engine  | Use when |
|---------|----------|
| InnoDB  | Default for everything transactional (WordPress core tables, WooCommerce) |
| Aria    | Read-heavy non-transactional tables, internal temp tables (MariaDB uses Aria automatically for tmp tables) |
| MyISAM  | Avoid in new designs — no ACID, no row-level locking |
| MEMORY  | Session caches, ephemeral lookup tables where data loss on restart is acceptable |

```sql
-- Always specify InnoDB explicitly for new tables
CREATE TABLE orders (
    order_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    order_date  DATETIME     NOT NULL DEFAULT current_timestamp(),
    total       DECIMAL(12,2) NOT NULL,
    status      ENUM('pending','processing','shipped','delivered','cancelled')
                NOT NULL DEFAULT 'pending',
    INDEX idx_customer_date (customer_id, order_date),
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci
  ROW_FORMAT=DYNAMIC;
```

### Data Types (MariaDB-specific notes)

- `DECIMAL(M,D)` — always for money, never `FLOAT`/`DOUBLE`
- `TINYINT(1)` — boolean; MariaDB respects this convention
- `INET4` / `INET6` (10.10+) — native IP address types; far more efficient than `VARCHAR(45)`
- `UUID` (10.7+) — native UUID type; stores as 16-byte binary internally, displays as text
- `JSON` — stored as `LONGTEXT` in MariaDB (validated); use `JSON_VALID()` in CHECK constraints
- Use `utf8mb4` + `utf8mb4_unicode_ci` everywhere — never `utf8` (which is actually 3-byte in MariaDB/MySQL)

```sql
-- Native IP address type (10.10+)
CREATE TABLE access_log (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    ip_addr    INET6 NOT NULL,    -- accepts both IPv4 and IPv6
    requested  DATETIME(3) NOT NULL DEFAULT current_timestamp(3),
    INDEX idx_ip (ip_addr)
) ENGINE=InnoDB;

-- UUID type (10.7+)
CREATE TABLE events (
    event_id   UUID NOT NULL DEFAULT uuid() PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    payload    JSON,
    CHECK (JSON_VALID(payload)),
    created_at DATETIME NOT NULL DEFAULT current_timestamp()
) ENGINE=InnoDB;
```

### Invisible Columns (10.3+)

Useful for schema migrations and audit fields that shouldn't clutter `SELECT *`:

```sql
ALTER TABLE posts
    ADD COLUMN row_hash CHAR(64) INVISIBLE
        AS (SHA2(CONCAT(title, content), 256)) VIRTUAL;

-- Hidden from SELECT * but queryable explicitly
SELECT post_id, row_hash FROM posts WHERE row_hash = '...';
```

### Instant DDL (10.3+ InnoDB)

Adding columns is instant and does not rebuild the table:

```sql
-- This does NOT lock the table on MariaDB 10.3+ InnoDB
ALTER TABLE orders
    ADD COLUMN loyalty_tier TINYINT UNSIGNED NULL DEFAULT NULL,
    ALGORITHM=INSTANT;
```

> **WordPress caveat**: don't add custom columns to core `wp_*` tables (e.g.
> `wp_posts`, `wp_options`). Core updates and plugins assume the stock schema,
> and `dbDelta()` can drop columns it doesn't recognize during upgrades. Store
> custom data in `postmeta`/`usermeta`/`options`, or create your own dedicated
> table instead.

---

## System-Versioned Tables (Temporal / Audit History)

Built-in row-level history — no triggers or extra code required:

```sql
CREATE TABLE product_prices (
    product_id   INT UNSIGNED NOT NULL,
    price        DECIMAL(10,2) NOT NULL,
    updated_by   INT UNSIGNED,
    PRIMARY KEY (product_id)
) WITH SYSTEM VERSIONING
  ENGINE=InnoDB;

-- Query historical data
SELECT * FROM product_prices
    FOR SYSTEM_TIME AS OF '2025-01-01 00:00:00';

SELECT * FROM product_prices
    FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2025-01-01';

-- See all versions of a row
SELECT *, ROW_START, ROW_END FROM product_prices
    FOR SYSTEM_TIME ALL
    WHERE product_id = 42
    ORDER BY ROW_START;

-- Partition history to keep main table fast
ALTER TABLE product_prices
    PARTITION BY SYSTEM_TIME (
        PARTITION p_history HISTORY,
        PARTITION p_current CURRENT
    );

-- Enable restoring historical data from dumps (10.11+)
SET system_versioning_insert_history = ON;
```

---

## Indexing Strategy

### Fundamentals

- Index columns used in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`
- Most selective column first in composite indexes
- **Prefix indexes** required for `utf8mb4` columns beyond the index key-length
  limit. `ROW_FORMAT=REDUNDANT`/`COMPACT` cap index prefixes at 767 bytes;
  `ROW_FORMAT=DYNAMIC`/`COMPRESSED` (with `innodb_large_prefix`, default on)
  raise that to 3072 bytes — enough to fully index a `VARCHAR(191)` utf8mb4
  column without truncation
- Descending index columns (10.11): `INDEX idx_date_desc (created_at DESC)`

```sql
-- Composite: equality first, range last, then ORDER BY columns
CREATE INDEX idx_posts_type_status_date
    ON wp_posts (post_type, post_status, post_date);

-- Covering index (avoids table lookup entirely)
CREATE INDEX idx_orders_cover
    ON orders (customer_id, status, order_date, total);

-- Full-text search
ALTER TABLE wp_posts
    ADD FULLTEXT INDEX ft_post_content (post_title, post_content);

-- Use MATCH/AGAINST, not LIKE '%keyword%'
SELECT ID, post_title FROM wp_posts
WHERE MATCH(post_title, post_content) AGAINST ('keyword phrase' IN NATURAL LANGUAGE MODE)
  AND post_type = 'post' AND post_status = 'publish';

-- Check index usage and cardinality
SELECT table_name, index_name, column_name, cardinality
FROM information_schema.STATISTICS
WHERE table_schema = DATABASE()
ORDER BY table_name, index_name, seq_in_index;

-- Find unused indexes (run after adequate traffic; requires performance_schema)
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema = DATABASE()
ORDER BY object_name;
```

---

## Query Optimization

### ANALYZE FORMAT=JSON (10.11 includes optimizer time)

```sql
-- See the full execution plan including time spent by the optimizer (10.11)
ANALYZE FORMAT=JSON
SELECT p.ID, p.post_title, pm.meta_value
FROM wp_posts p
INNER JOIN wp_postmeta pm ON p.ID = pm.post_id
WHERE p.post_type = 'post'
  AND p.post_status = 'publish'
  AND pm.meta_key = '_thumbnail_id'
ORDER BY p.post_date DESC
LIMIT 20;
```

### Common Anti-Patterns to Avoid

```sql
-- BAD: function on indexed column — prevents index use
SELECT * FROM wp_posts WHERE YEAR(post_date) = 2025;

-- GOOD: range scan uses the index
SELECT * FROM wp_posts
WHERE post_date >= '2025-01-01' AND post_date < '2026-01-01';

-- BAD: leading wildcard kills full-text performance
SELECT * FROM wp_posts WHERE post_title LIKE '%keyword%';

-- GOOD: use FULLTEXT
SELECT * FROM wp_posts
WHERE MATCH(post_title) AGAINST ('keyword' IN BOOLEAN MODE)
  AND post_status = 'publish';

-- BAD: implicit type coercion (post_id is INT, passing string)
SELECT * FROM wp_postmeta WHERE post_id = '123';

-- GOOD: match types
SELECT * FROM wp_postmeta WHERE post_id = 123;

-- BAD: SELECT * in production queries
SELECT * FROM wp_posts WHERE ID = 42;

-- GOOD: select only needed columns
SELECT ID, post_title, post_status FROM wp_posts WHERE ID = 42;

-- BAD: correlated subquery executed per row
SELECT p.ID FROM wp_posts p
WHERE (SELECT COUNT(*) FROM wp_comments c WHERE c.comment_post_ID = p.ID) > 5;

-- GOOD: JOIN + GROUP BY
SELECT p.ID
FROM wp_posts p
INNER JOIN wp_comments c ON c.comment_post_ID = p.ID
GROUP BY p.ID
HAVING COUNT(*) > 5;
```

### Keyset Pagination for Large Datasets

```sql
-- Offset pagination breaks down at high page numbers (OFFSET 50000 is slow)
-- BAD at scale:
SELECT * FROM wp_posts ORDER BY ID DESC LIMIT 20 OFFSET 10000;

-- GOOD: keyset (cursor) pagination — O(log n) regardless of depth
SELECT ID, post_title, post_date
FROM wp_posts
WHERE post_type = 'post'
  AND post_status = 'publish'
  AND ID < :last_seen_id   -- pass in the last ID from the previous page
ORDER BY ID DESC
LIMIT 20;
```

---

## WordPress-Specific Optimizations

> **Read `references/wordpress.md`** for the full step-by-step guide covering
> diagnosis, indexing, autoload cleanup, orphan removal, engine conversion,
> ongoing maintenance queries, and WP-CLI commands.

Key principles at 1M+ posts:

- **Check existing indexes before adding new ones**: modern WordPress core
  already ships `wp_postmeta (meta_key(191))` and, since 6.6, an index on
  `wp_options (autoload)`. Run `SHOW INDEX` first — don't assume they're missing.
- **Autoload payload < 800 KB**: every page load fetches all autoloaded rows.
  Audit with `SELECT ROUND(SUM(LENGTH(option_value))/1024,2) AS kb FROM wp_options WHERE autoload='yes'`.
- **Delete in batches**: for single-table deletes, `LIMIT 10000` per pass avoids
  long locks. Multi-table `DELETE ... JOIN` does **not** support `LIMIT` — batch
  those with a subquery instead (see `references/wordpress.md`).
- **Set `WP_POST_REVISIONS = 3`** in `wp-config.php` before cleaning revisions.
- **All core WP tables must be InnoDB** with `ROW_FORMAT=DYNAMIC` and `utf8mb4`.
- **Use `ALGORITHM=INPLACE, LOCK=NONE`** on MariaDB 10.3+ for online DDL.
- **Redis object cache** is the single highest-impact optimization for 1M+ post
  sites — it absorbs autoload and postmeta traffic before it hits MariaDB.

---

## Transaction Management

```sql
-- Keep transactions short; avoid user interaction between BEGIN and COMMIT
START TRANSACTION;

UPDATE accounts SET balance = balance - 100.00 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100.00 WHERE account_id = 2;

-- Check for errors in application code before committing
COMMIT;

-- Appropriate isolation level for most WordPress/web workloads:
-- READ COMMITTED reduces gap locking and deadlocks
SET GLOBAL transaction_isolation = 'READ-COMMITTED';

-- Deadlock detection: InnoDB auto-detects; review with:
SHOW ENGINE INNODB STATUS\G
-- Look for the "LATEST DETECTED DEADLOCK" section
```

---

## Configuration, Replication, Backup & Monitoring

> **Read `references/config-and-admin.md`** for complete `mariadb.cnf` blocks
> (production large-site and personal/small-site profiles), monitoring queries,
> maintenance schedule, replication commands, backup procedures, and the full
> MariaDB vs MySQL gotchas table.

Key principles:

- `innodb_buffer_pool_size` = ~70% of dedicated RAM (single highest-impact setting)
- Query cache **exists but is disabled by default** — leave it off at scale; its
  single global mutex doesn't scale on multi-core, write-heavy hosts. Use a
  Redis object cache in front of WordPress instead
- `innodb_flush_neighbors = 0` for SSD/NVMe; `1` for spinning disks
- `aria_pagecache_buffer_size = 256M` — Aria is used for internal temp tables
- `binlog_format = ROW` and `sync_binlog = 1` for safe replication
- `require_secure_transport = ON` + `encrypt_binlog = ON` for security
- `local_infile = OFF` and `symbolic_links = OFF` always
- Use `mariadb-backup` (hot backup, no downtime) for production; `mariadb-dump`
  for smaller DBs or cross-version portability
- **Always test restores** — a backup you've never restored is a guess

---

## JSON Usage (MariaDB-Specific)

MariaDB stores JSON as `LONGTEXT` with full JSON function support.
Use generated columns to index frequently queried JSON paths:

```sql
CREATE TABLE event_log (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    event_type  VARCHAR(50) NOT NULL,
    payload     JSON NOT NULL CHECK (JSON_VALID(payload)),
    user_id     INT UNSIGNED AS (payload->>'$.user_id') STORED,
    created_at  DATETIME NOT NULL DEFAULT current_timestamp(),
    INDEX idx_user_id (user_id),
    INDEX idx_type_ts (event_type, created_at)
) ENGINE=InnoDB;

-- JSON_NORMALIZE (10.7+) — sort keys, remove spaces (useful for deduplication)
SELECT JSON_NORMALIZE('{"b":2,"a":1}');  -- returns '{"a":1,"b":2}'

-- JSON_TABLE — pivot JSON arrays into relational rows
SELECT jt.*
FROM event_log e,
     JSON_TABLE(e.payload, '$' COLUMNS (
         user_id INT PATH '$.user_id',
         action  VARCHAR(100) PATH '$.action'
     )) AS jt
WHERE e.event_type = 'user_action';
```

*Last Updated: 2026-07-28*
