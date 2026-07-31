# Redis Production Operations — Deep Reference

> Load this file for connection resilience, monitoring, memory management,
> and persistence configuration. Targets Redis server 7.4 and node-redis
> client 5.12.1.

---

## Connection Resilience

```ts
import { createClient } from 'redis';

const client = createClient({
  url: process.env.REDIS_URL,
  socket: {
    connectTimeout: 5000,
    reconnectStrategy: (retries, cause) => {
      // Log the cause for debugging transient vs persistent failures
      console.error(`Redis reconnect attempt ${retries}:`, cause.message);

      if (retries > 20) {
        // Stop retrying after 20 attempts — let the app's health check
        // catch this and restart, rather than retrying forever silently
        return new Error('Redis connection failed after 20 retries');
      }

      // Exponential backoff with jitter, capped at 3s
      const delay = Math.min(Math.pow(2, retries) * 50, 3000);
      const jitter = Math.random() * 100;
      return delay + jitter;
    },
  },
});

client.on('error', (err) => {
  // NEVER let this go unhandled — an uncaught 'error' event crashes Node.js
  console.error('Redis Client Error', err);
  // Consider alerting here for production — Redis being down is often
  // a critical incident, not a log line to ignore
});

client.on('reconnecting', () => {
  console.warn('Redis client reconnecting...');
});

client.on('ready', () => {
  console.info('Redis client ready');
});

client.on('end', () => {
  console.warn('Redis client connection closed');
});
```

### Health Checks

```ts
// Simple liveness check for a health endpoint
async function isRedisHealthy(): Promise<boolean> {
  try {
    const pong = await client.ping();
    return pong === 'PONG';
  } catch {
    return false;
  }
}

// Express health check route
app.get('/health/redis', async (req, res) => {
  const healthy = await isRedisHealthy();
  res.status(healthy ? 200 : 503).json({ redis: healthy ? 'up' : 'down' });
});
```

### Graceful Shutdown

```ts
async function gracefulShutdown() {
  console.log('Shutting down...');
  try {
    // .destroy() closes immediately — use when you don't need pending
    // commands to complete. Prefer this in most shutdown paths.
    await client.destroy();
  } catch (err) {
    console.error('Error during Redis shutdown', err);
  }
  process.exit(0);
}

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

---

## Monitoring

```ts
// INFO command — comprehensive server stats
const info = await client.info();
// Sections: server, clients, memory, persistence, stats, replication, cpu, cluster, keyspace

const memoryInfo = await client.info('memory');
const statsInfo = await client.info('stats');
```

```bash
# redis-cli commands for operational visibility (run against the server directly)

# Real-time command monitoring (use sparingly — has performance overhead)
redis-cli monitor

# Slow query log — commands exceeding a configured threshold
redis-cli slowlog get 10
redis-cli config set slowlog-log-slower-than 10000  # microseconds

# Current client connections
redis-cli client list

# Memory usage summary
redis-cli info memory

# Keyspace stats (key counts per database)
redis-cli info keyspace
```

```ts
// Programmatic slow log access
const slowLogs = await client.sendCommand(['SLOWLOG', 'GET', '10']);
```

---

## Memory Management

```ts
// Check memory usage of a specific key — useful for identifying bloat
const bytes = await client.sendCommand(['MEMORY', 'USAGE', 'user:1']);

// Check the internal encoding Redis chose for a key — affects memory
// efficiency significantly for small collections
const encoding = await client.sendCommand(['OBJECT', 'ENCODING', 'user:1:roles']);
// Small sets/hashes/lists use compact encodings (listpack, intset) —
// they automatically convert to full encodings past configured thresholds
```

```ini
# redis.conf settings that affect memory encoding thresholds
# (relevant when deciding whether many small hashes/sets are memory-efficient)
hash-max-listpack-entries 128
hash-max-listpack-value 64
set-max-listpack-entries 128
set-max-intset-entries 512
zset-max-listpack-entries 128
list-max-listpack-size 128
```

```ini
# Eviction policy — critical for cache-heavy workloads where Redis
# might fill up. Without an eviction policy, Redis rejects writes
# when it hits maxmemory instead of evicting old data.
maxmemory 2gb
maxmemory-policy allkeys-lru   # evict least-recently-used keys across all keys
# Other options:
# volatile-lru      — LRU eviction, only among keys with a TTL set
# allkeys-lfu        — least-frequently-used, all keys
# volatile-lfu        — LFU, only keys with TTL
# volatile-ttl         — evict keys with the shortest remaining TTL first
# noeviction             — reject writes when full (default — often wrong for caches)
```

```
Choosing a maxmemory-policy:
- Pure cache use case, no persistent data in Redis → allkeys-lru or allkeys-lfu
- Mixed use case (some keys are cache, some are durable state) →
  volatile-lru/volatile-lfu, and ensure durable keys never get a TTL
- If you're unsure, allkeys-lru is the safest general default for
  a cache-primary Redis instance
```

---

## Persistence: RDB vs AOF

```ini
# RDB — point-in-time snapshots, compact, faster restarts, but can lose
# data written since the last snapshot on a crash
save 900 1      # snapshot if at least 1 write in 900 seconds
save 300 10     # snapshot if at least 10 writes in 300 seconds
save 60 10000   # snapshot if at least 10000 writes in 60 seconds

# AOF — append-only log of every write, more durable, larger on disk,
# slower restarts (replays the whole log)
appendonly yes
appendfsync everysec   # fsync every second — good balance of durability/perf
# appendfsync always   — fsync every write, safest but slowest
# appendfsync no        — let the OS decide, fastest but least durable
```

```
Recommendation for session/cache use cases (matches connect-redis and
general caching patterns in the main skill): RDB-only is often sufficient
— if Redis crashes and loses the last snapshot's worth of cache/session
data, the application degrades gracefully (cache repopulates, users
re-authenticate) rather than losing anything irrecoverable.

Recommendation if Redis holds anything you can't afford to lose (queues
without an external durable source, counters that can't be recomputed):
enable AOF with appendfsync everysec as a minimum durability bar.
```

```bash
# Manual snapshot trigger
redis-cli bgsave

# Check last save status
redis-cli lastsave

# AOF rewrite (compacts the log file)
redis-cli bgrewriteaof
```

---

## Docker / Container Considerations

```yaml
# docker-compose.yml — production-oriented Redis 7.4 service definition
services:
  redis:
    image: docker.io/redis:7.4-alpine
    command: >
      redis-server
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
      --save 900 1
    volumes:
      - redis-data:/data
    ports:
      - '6379:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 2.5g   # headroom above maxmemory for Redis's own overhead

volumes:
  redis-data:
```

```bash
# If running with a password, pass it via env or secrets, never in the
# command line where it's visible in `docker inspect` / process listings
```

```yaml
services:
  redis:
    image: docker.io/redis:7.4-alpine
    environment:
      REDIS_PASSWORD_FILE: /run/secrets/redis_password
    secrets:
      - redis_password
    command: >
      sh -c 'redis-server --requirepass "$$(cat /run/secrets/redis_password)"'
secrets:
  redis_password:
    file: ./secrets/redis_password.txt
```

---

## Standalone vs Cluster — When to Consider Clustering

```
Standalone (your current setup, redis:7.4-alpine) is appropriate when:
- Working dataset fits comfortably in one node's memory
- Throughput requirements are within a single instance's capacity
- Simpler operational model is a priority (one node to monitor, back up, patch)

Consider Redis Cluster when:
- Dataset exceeds what one node can hold in memory
- You need horizontal write scaling beyond one node's throughput ceiling
- High availability requires automatic failover across multiple nodes
  (note: for standalone HA without full clustering, Redis Sentinel is a
  lighter-weight option — provides automatic failover between a primary
  and replicas without sharding the keyspace)
```

```ts
// node-redis cluster client — different API surface from createClient()
// per the main skill's node-redis notes, cluster support has historically
// had rougher edges (missing convenience methods some createClient() users
// expect, less mature TypeScript support) — evaluate carefully, and
// consider whether Sentinel-based HA meets your actual need before
// reaching for full clustering
import { createCluster } from 'redis';

const cluster = createCluster({
  rootNodes: [
    { url: 'redis://node1:6379' },
    { url: 'redis://node2:6379' },
    { url: 'redis://node3:6379' },
  ],
});
await cluster.connect();
```

---

## Security Checklist

```ini
# redis.conf hardening baseline
requirepass <strong-random-password>   # always set a password
bind 127.0.0.1 -::1                     # bind only to needed interfaces, never 0.0.0.0 publicly
protected-mode yes                       # refuse external connections without auth
rename-command FLUSHALL ""              # disable dangerous commands entirely if not needed
rename-command FLUSHDB ""
rename-command CONFIG ""                 # or rename to something non-obvious if you still need it
```

```ts
// Application-side: never construct Redis commands from unsanitized user
// input via sendCommand() or eval() — while Redis doesn't have SQL-style
// injection in the traditional sense, building command arrays or Lua
// scripts from raw user input is still a risk surface worth treating
// carefully, especially with eval()/evalSha()

// BAD: building a Lua script string via concatenation of user input
const unsafeScript = `redis.call('SET', KEYS[1], '${userInput}')`; // ❌

// GOOD: pass user input as ARGV, never interpolate into the script body
const script = `redis.call('SET', KEYS[1], ARGV[1])`;
await client.eval(script, { keys: ['some-key'], arguments: [userInput] }); // ✅
```

---

## ACLs (Access Control) — Beyond a Single Password

```bash
# Create a limited-privilege user for a specific service — instead of
# every service sharing the same requirepass credential
redis-cli ACL SETUSER app-readonly on '>strongpassword' '~cache:*' +get +exists -@all

# ~cache:*     — only allow access to keys matching this pattern
# +get +exists — explicitly allow only these commands
# -@all         — deny everything else by default, then allow-list from there
```

```ts
// Connect as a specific ACL user
const client = createClient({
  url: 'redis://app-readonly:strongpassword@localhost:6379',
});
```

Use ACLs to scope each service's Redis access to only the keys and commands
it actually needs — particularly valuable if multiple services or teams
share one Redis instance, since it limits blast radius if one service's
credentials or code has a bug.

*Last Updated: 2026-07-30*
