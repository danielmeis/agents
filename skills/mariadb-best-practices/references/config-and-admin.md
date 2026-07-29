# MariaDB 10.11: Configuration, Replication, Backup & Monitoring

> Load this file when the user needs full `mariadb.cnf` settings, replication
> commands, backup procedures, or detailed monitoring queries.

---

## Configuration (`mariadb.cnf`)

### Production WordPress / Large Site (dedicated DB server)

```ini
[mariadb]
# ── Character Set ──────────────────────────────────────────────
character_set_server = utf8mb4
collation_server     = utf8mb4_unicode_ci

# ── InnoDB ─────────────────────────────────────────────────────
innodb_buffer_pool_size       = 8G       # ~70% of dedicated RAM
innodb_buffer_pool_instances  = 8        # 1 per GB, max 8
innodb_log_file_size          = 512M     # 25% of buffer pool
innodb_flush_log_at_trx_commit = 1       # 1 = full ACID
innodb_flush_method           = O_DIRECT # avoid double-buffering (Linux)
innodb_io_capacity            = 2000     # IOPS your SSD sustains
innodb_io_capacity_max        = 4000
innodb_read_io_threads        = 8
innodb_write_io_threads       = 8
innodb_flush_neighbors        = 0        # 0 for SSD/NVMe; 1 for HDD

# ── Aria (internal temp tables use Aria automatically) ─────────
aria_pagecache_buffer_size = 256M

# ── Query Cache (MariaDB only — still valuable for WordPress) ──
query_cache_type  = 1
query_cache_size  = 64M
query_cache_limit = 2M

# ── Connections & Threads ─────────────────────────────────────
max_connections        = 200   # tune to actual peak + 20% headroom
thread_cache_size      = 32
thread_stack           = 256K
table_open_cache       = 4000
table_definition_cache = 2000

# ── Temp tables ───────────────────────────────────────────────
tmp_memory_table_size = 64M
max_heap_table_size   = 64M

# ── Slow query log ────────────────────────────────────────────
slow_query_log                = 1
slow_query_log_file           = /var/log/mysql/slow.log
long_query_time               = 1
log_queries_not_using_indexes = 1

# ── Binary log ────────────────────────────────────────────────
log_bin          = /var/log/mysql/mariadb-bin
binlog_format    = ROW
expire_logs_days = 7
sync_binlog      = 1
encrypt_binlog   = ON

# ── Security ──────────────────────────────────────────────────
local_infile             = OFF
symbolic_links           = OFF
skip_name_resolve        = ON
require_secure_transport = ON
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key  = /etc/mysql/ssl/server-key.pem
ssl_ca   = /etc/mysql/ssl/ca-cert.pem
tls_version = TLSv1.2,TLSv1.3

# ── Plugins ───────────────────────────────────────────────────
plugin_load_add = server_audit
server_audit_logging  = ON
server_audit_events   = CONNECT,QUERY_DDL
server_audit_file_path = /var/log/mysql/audit.log

# ── At-rest encryption ────────────────────────────────────────
innodb_encrypt_tables      = ON
innodb_encrypt_log         = ON
innodb_encryption_threads  = 4
plugin_load_add            = file_key_management
file_key_management_filename = /etc/mysql/encryption/keyfile
file_key_management_filekey  = /etc/mysql/encryption/keyfile.key
```

### Personal / Small WordPress Site (shared or low-RAM VPS)

```ini
[mariadb]
character_set_server       = utf8mb4
collation_server           = utf8mb4_unicode_ci
innodb_buffer_pool_size    = 256M
innodb_log_file_size       = 64M
query_cache_type           = 1
query_cache_size           = 32M
max_connections            = 50
tmp_memory_table_size      = 16M
max_heap_table_size        = 16M
aria_pagecache_buffer_size = 32M
slow_query_log             = 1
long_query_time            = 2
```

---

## Key Variables to Monitor

```sql
-- How many connections have we peaked at? (tune max_connections accordingly)
SHOW GLOBAL STATUS LIKE 'Max_used_connections';

-- Buffer pool hit rate (target > 99%)
SELECT ROUND(
    (1 - (
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        NULLIF(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0)
    )) * 100, 2) AS buffer_pool_hit_pct;

-- Query cache hit rate (if enabled)
SHOW GLOBAL STATUS LIKE 'Qcache%';
-- Qcache_hits / (Qcache_hits + Qcache_inserts) = hit rate

-- Threads and connection pressure
SHOW GLOBAL STATUS LIKE 'Threads_%';

-- Slow queries since startup
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- Table fragmentation
SELECT TABLE_NAME,
       ROUND(DATA_FREE/1024/1024, 2) AS free_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_FREE > 10*1024*1024   -- > 10 MB fragmented
ORDER BY DATA_FREE DESC;

-- InnoDB internals (locks, deadlocks, buffer usage)
SHOW ENGINE INNODB STATUS\G
```

---

## Maintenance Schedule

```sql
-- Weekly: refresh optimizer statistics
ANALYZE TABLE wp_posts, wp_postmeta, wp_options, wp_comments,
             wp_usermeta, wp_term_relationships;

-- Monthly: reclaim fragmented space
-- Small tables: OPTIMIZE is fine (brief lock)
OPTIMIZE TABLE wp_options;

-- Large tables: online rebuild (InnoDB, LOCK=NONE)
ALTER TABLE wp_posts     ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE wp_postmeta  ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE wp_comments  ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
```

---

## Replication (GTID — Default in 10.11)

```sql
-- Verify GTID mode
SHOW VARIABLES LIKE 'gtid_strict_mode';

-- Check replica health (use SHOW REPLICA STATUS, not SHOW SLAVE STATUS)
SHOW REPLICA STATUS\G

-- Replica lag (performance_schema)
SELECT TIMESTAMPDIFF(SECOND,
    MAX(LAST_APPLIED_TRANSACTION_END_APPLY_TIMESTAMP),
    NOW()) AS lag_seconds
FROM performance_schema.replication_applier_status_by_worker;

-- Cleanly promote a replica to primary
STOP REPLICA;
RESET REPLICA ALL;
```

**Read/write splitting for WordPress**: use ProxySQL, MaxScale, HyperDB, or
LudicrousDB. Never serve reads from a replica with known replication lag
immediately after a write that the reader depends on.

---

## Backup

```bash
# Physical (hot) backup — preferred for multi-GB databases
mariadb-backup --backup \
    --target-dir=/backups/$(date +%F_%H%M) \
    --user=mariadb-backup \
    --password='...' \
    --parallel=4

# Prepare the backup before restoring
mariadb-backup --prepare --target-dir=/backups/2025-01-01_0200

# Logical dump — smaller DBs, cross-version portability
mariadb-dump \
    --single-transaction \
    --quick \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --set-gtid-purged=OFF \
    wordpress_db | gzip > /backups/wp_$(date +%F).sql.gz

# Always test restores — schedule a monthly restore drill to a separate instance
```

---

## Security Audit Queries

Run these on a schedule (e.g. monthly) to catch privilege drift:

```sql
-- Users with dangerous global privileges
SELECT user, host, Super_priv, Grant_priv, File_priv
FROM mysql.user
WHERE Super_priv = 'Y' OR Grant_priv = 'Y' OR File_priv = 'Y';

-- Accounts with no password that aren't socket-authenticated
SELECT user, host, plugin, password
FROM mysql.user
WHERE (password = '' OR password IS NULL)
  AND plugin NOT IN ('unix_socket', 'gssapi');

-- Anonymous accounts (should be empty)
SELECT user, host FROM mysql.user WHERE user = '';

-- Confirm dangerous globals are off
SHOW GLOBAL VARIABLES WHERE
    Variable_name IN ('local_infile','symbolic_links','skip_networking');
```

---

## Quick Reference: MariaDB 10.11 vs MySQL 8.0

| Topic | MariaDB 10.11 | MySQL 8.0 |
|-------|--------------|-----------|
| Query cache | Available & beneficial | Removed |
| Root auth | UNIX socket (no password) | Password required |
| JSON storage | `LONGTEXT` with validation | Binary JSON type |
| Native UUID type | Yes (10.7+) | No |
| INET4/INET6 types | Yes (10.10+) | No |
| System-versioned tables | Yes (10.3+) | No |
| Instant ADD COLUMN | Yes (10.3+) | Yes (8.0+) |
| Aria engine | Yes | No |
| ed25519 auth | Yes | No |
| `SHOW REPLICA STATUS` | 10.5+ (SLAVE deprecated) | Both work |
| `mariadb-dump` | Preferred name | `mysqldump` |
| Replication GTID default | On (10.11) | Configurable |

*Last Updated: 2026-07-28*
