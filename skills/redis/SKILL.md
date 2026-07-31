---
name: redis
description: >
  Redis best practices for the node-redis (npm `redis`) client v5.x, Redis
  server 7.4, connect-redis 9.x session storage, caching patterns, pub/sub,
  transactions, and data structure design. Use this skill for any Redis
  question: connecting, commands, key design, TTL/expiration strategy,
  pipelining, pub/sub, Lua scripting, RedisClientPool, and Express session
  storage via connect-redis. This skill targets the CURRENT versions in use
  (see Version Targets below) — it deliberately does not assume the newer
  node-redis v6 client or Redis server 8.x, since those are not yet in
  production here. Forward-looking notes on v6/8.x are included as clearly
  marked callouts for future migration planning, not as the default guidance.
  @socket.io/redis-adapter is OUT OF SCOPE — that belongs in the socket.io
  skill, not here.
---

# Redis Best Practices

> **Version targets for this skill — update this block when you upgrade:**
> - Redis server: **7.4** (`docker.io/redis:7.4-alpine`)
> - `redis` npm client (node-redis): **5.12.1**
> - `connect-redis`: **9.0.0**
>
> All code examples in this skill are written against these exact versions.
> Forward-looking notes about node-redis v6 and Redis server 8.x are marked
> `> v6+` / `> Redis 8+` — they exist so you know what changes when you
> eventually upgrade, not because they're safe to use today.

---

## Scope Note

`@socket.io/redis-adapter` is intentionally excluded from this skill — Socket.IO's
Redis adapter usage belongs in a dedicated socket.io skill, since its patterns
(pub/sub for cross-instance broadcast, adapter setup) are Socket.IO-specific
rather than general Redis usage.

---

## Core Principles

- Always attach an `'error'` listener before calling `.connect()` — an
  unhandled `error` event on the client crashes the Node.js process
- Use `client.multi()` for atomic multi-command operations, not sequential awaits
- Set an explicit TTL on every key that represents cache or transient data —
  untamed keys without expiration are the most common cause of Redis memory growth
- Prefer specific commands (`hSet`, `sAdd`, `zAdd`) over generic key-value
  storage when the data has structure — Redis's data types exist for a reason
- Never use `KEYS` in production code — always use `SCAN` (or `scanIterator`)
- Design keys with a consistent `namespace:entity:id` convention

---

## Connecting

```ts
import { createClient } from 'redis';

const client = createClient({
  url: process.env.REDIS_URL, // redis://user:pass@host:6379/0
});

// ALWAYS attach the error listener before connecting — an unhandled
// 'error' event will crash the process
client
  .on('error', (err) => console.error('Redis Client Error', err))
  .on('reconnecting', () => console.log('Redis reconnecting...'))
  .on('ready', () => console.log('Redis client ready'));

await client.connect();

// Graceful shutdown
process.on('SIGTERM', async () => {
  await client.destroy(); // preferred over .quit() in v5 — closes immediately
});
```

```ts
// Full connection options
const client = createClient({
  socket: {
    host: 'localhost',
    port: 6379,
    connectTimeout: 5000,
    reconnectStrategy: (retries) => {
      if (retries > 10) return new Error('Too many retries, giving up');
      return Math.min(retries * 100, 3000); // exponential-ish backoff, capped
    },
  },
  password: process.env.REDIS_PASSWORD,
  database: 0,
});
```

### Connection Pooling — `RedisClientPool`

> node-redis v5 extracted pooling into its own dedicated class — this is a
> real API, not a workaround. In v4, pooling only existed as an "isolation
> pool" layered on a single main connection; v5's `RedisClientPool` is
> standalone and doesn't require a main connection at all.

```ts
import { createClientPool } from 'redis';

const pool = await createClientPool({
  url: process.env.REDIS_URL,
})
  .on('error', (err) => console.error('Redis Pool Error', err))
  .connect();

// Use the pool exactly like a regular client — it handles checkout/return internally
await pool.set('key', 'value');
const value = await pool.get('key');

// Configure pool size
const pool2 = await createClientPool(
  { url: process.env.REDIS_URL },
  { minimum: 5, maximum: 20 }
).connect();
```

Use `RedisClientPool` for high-concurrency API servers where a single
connection would serialize commands under load. Use a plain `createClient()`
for simple scripts, workers, or low-concurrency services — pooling adds
overhead that isn't worth it below a certain request volume.

---

## Basic Commands

```ts
// Strings
await client.set('user:1:name', 'Alice');
await client.set('user:1:name', 'Alice', { EX: 3600 }); // expire in 1 hour
await client.setNX('lock:job:42', '1'); // set only if not exists — basic locking primitive
const name = await client.get('user:1:name'); // string | null
await client.del('user:1:name');
await client.incr('page:views');
await client.incrBy('page:views', 5);

// Hashes — ideal for objects with multiple fields
await client.hSet('user:1', { name: 'Alice', email: 'alice@example.com', age: '30' });
const user = await client.hGetAll('user:1'); // Record<string, string>
const name2 = await client.hGet('user:1', 'name');
await client.hDel('user:1', 'age');
await client.hIncrBy('user:1:stats', 'login_count', 1);

// Sets — unique unordered collections
await client.sAdd('user:1:roles', ['admin', 'editor']);
const roles = await client.sMembers('user:1:roles');
const isAdmin = await client.sIsMember('user:1:roles', 'admin');
await client.sRem('user:1:roles', 'editor');

// Sorted Sets — ranked/scored data (leaderboards, priority queues, time-ordered feeds)
await client.zAdd('leaderboard', [{ score: 100, value: 'user:1' }]);
const top10 = await client.zRange('leaderboard', 0, 9, { REV: true });
const rank = await client.zRank('leaderboard', 'user:1');
await client.zIncrBy('leaderboard', 10, 'user:1');

// Lists — ordered collections, good for queues
await client.lPush('queue:jobs', JSON.stringify({ type: 'email', id: 42 }));
const job = await client.rPop('queue:jobs'); // FIFO when paired with lPush
await client.lRange('queue:jobs', 0, -1); // view all without removing
```

---

## TTL and Expiration Strategy

```ts
// Set TTL at write time — the most common and safest pattern
await client.set('session:abc123', JSON.stringify(sessionData), { EX: 1800 });

// Set/update TTL on an existing key
await client.expire('session:abc123', 1800);

// Check remaining TTL
const ttl = await client.ttl('session:abc123'); // seconds, -1 = no TTL, -2 = key doesn't exist

// Remove TTL (make a key persist forever) — use deliberately, rarely by accident
await client.persist('session:abc123');

// Common mistake: forgetting TTL on cache keys entirely
// BAD — this key lives forever unless explicitly deleted
await client.set('cache:product:42', JSON.stringify(product));

// GOOD — always cap cache entries
await client.set('cache:product:42', JSON.stringify(product), { EX: 3600 });
```

---

## Pipelining and Transactions

```ts
// Pipelining — batch commands for network efficiency, NOT atomic
// node-redis auto-pipelines commands issued in the same tick, but you can
// be explicit for clarity and guaranteed batching:
const results = await client
  .multi()
  .set('a', '1')
  .set('b', '2')
  .get('a')
  .execAsPipeline(); // no atomicity guarantee, just batched for network efficiency

// Transactions — MULTI/EXEC, atomic execution
const [setResult, getResult] = await client
  .multi()
  .set('key', 'value')
  .get('another-key')
  .exec(); // ['OK', 'another-value'] — all or nothing

// Optimistic locking with WATCH — abort transaction if watched key changes
await client.watch('account:1:balance');
const balance = Number(await client.get('account:1:balance'));
if (balance < 100) {
  await client.unwatch();
  throw new Error('Insufficient balance');
}
const tx = client.multi().decrBy('account:1:balance', 100);
const result = await tx.exec(); // null if account:1:balance changed since WATCH
if (result === null) {
  throw new Error('Balance changed concurrently — retry');
}
```

> **v5 behavior change from v4**: if the socket disconnects mid-pipeline
> execution, any *unwritten* commands in that pipeline are now discarded
> rather than potentially still executing server-side. This makes pipeline
> failure semantics more predictable, but don't assume commands after a
> disconnect point were skipped without checking results explicitly.

---

## SCAN — Never Use KEYS in Production

```ts
// NEVER do this in production — KEYS blocks the entire server while
// scanning the full keyspace, causing latency spikes for every other client
const allKeys = await client.keys('user:*'); // ❌ blocks Redis, avoid always

// ALWAYS use SCAN instead — non-blocking, cursor-based iteration
for await (const key of client.scanIterator({ MATCH: 'user:*', COUNT: 100 })) {
  console.log(key);
}

// Manual cursor control if you need pause/resume behavior
let cursor = 0;
do {
  const result = await client.scan(cursor, { MATCH: 'user:*', COUNT: 100 });
  cursor = result.cursor;
  for (const key of result.keys) {
    // process key
  }
} while (cursor !== 0);

// Type-specific scan iterators exist too — prefer these when scanning
// a specific data type, they're more efficient
for await (const key of client.hScanIterator('user:1')) { /* hash fields */ }
for await (const member of client.sScanIterator('user:1:roles')) { /* set members */ }
for await (const item of client.zScanIterator('leaderboard')) { /* sorted set members */ }
```

---

## Pub/Sub

```ts
// Publisher — can be your main client
await client.publish('notifications', JSON.stringify({ type: 'new_order', id: 42 }));

// Subscriber — MUST be a dedicated connection; a client in subscribe mode
// cannot run other commands on the same connection
const subscriber = client.duplicate();
await subscriber.connect();

await subscriber.subscribe('notifications', (message) => {
  const payload = JSON.parse(message);
  console.log('Received:', payload);
});

// Pattern subscriptions
await subscriber.pSubscribe('notifications:*', (message, channel) => {
  console.log(`[${channel}]`, message);
});

// Unsubscribe and clean up
await subscriber.unsubscribe('notifications');
await subscriber.destroy();
```

```ts
// RESP3 changes how Pub/Sub messages are delivered internally (push messages
// vs RESP2's interleaved replies) — node-redis v5 handles this transparently
// when RESP: 3 is set, but be aware the wire behavior differs if you're
// debugging with raw protocol inspection tools
```

---

## Lua Scripting

```ts
// For atomic multi-step operations that transactions can't express cleanly
// (transactions can't do conditional logic based on intermediate results —
// Lua scripts run entirely server-side and can)

const script = `
  local current = redis.call('GET', KEYS[1])
  if current == false then
    redis.call('SET', KEYS[1], ARGV[1])
    return 1
  end
  return 0
`;

const result = await client.eval(script, {
  keys: ['lock:resource:1'],
  arguments: ['locked'],
});

// Reusable scripts — define once, call by name
const myScript = client.scriptLoad(script);
// Then: await client.evalSha(sha, { keys: [...], arguments: [...] });
```

---

## Client-Side Caching (RESP3)

> Requires node-redis v5.1.0+ and Redis server 7.4+ — available on your
> current stack (client 5.12.1, server 7.4)

```ts
// Enables the client to cache GET results locally and receive invalidation
// pushes from the server when the underlying key changes — dramatically
// reduces round-trips for hot, rarely-changing keys
const client = createClient({
  RESP: 3,
  clientSideCache: {
    ttl: 0,          // 0 = no local expiration (relies on server invalidation)
    maxEntries: 1000, // cap local cache size
    evictPolicy: 'LRU',
  },
});

// Usage is transparent — no API change needed, caching happens automatically
// under the hood for read commands on cached keys
const value = await client.get('rarely-changing-config-key');
```

Good candidates: config values, feature flags, rarely-changing reference
data. Bad candidates: anything that changes on every write — the
invalidation overhead exceeds the benefit.

---

## Type Mapping (RESP3)

> v5 replaced v4's `commandOptions({ returnBuffers: true })` pattern with
> a cleaner `withTypeMapping()` API

```ts
import { createClient, RESP_TYPES } from 'redis';

// Default: hGetAll returns Record<string, string>
const user = await client.hGetAll('user:1');

// Use Map instead of a plain object
const userMap = await client.withTypeMapping({ [RESP_TYPES.MAP]: Map }).hGetAll('user:1');
// Map<string, string>

// Combine Map + Buffer for binary-safe field values
const userBuffers = await client
  .withTypeMapping({ [RESP_TYPES.MAP]: Map, [RESP_TYPES.BLOB_STRING]: Buffer })
  .hGetAll('user:1');
// Map<string, Buffer>
```

---

## Legacy Mode (Callback API)

```ts
// If migrating from a codebase that used callback-style commands,
// v5 provides an explicit legacy wrapper rather than a global mode flag
const client = createClient();
await client.connect();

const legacyClient = client.legacy(); // callback-style API
legacyClient.set('key', 'value', (err, reply) => {
  if (err) throw err;
  console.log(reply);
});

// Prefer the promise-based client for all new code — legacy() exists
// purely as a migration bridge
```

---

## connect-redis 9.x — Express Session Storage

```ts
import session from 'express-session';
import { RedisStore } from 'connect-redis';
import { createClient } from 'redis';

const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.on('error', (err) => console.error('Redis Client Error', err));
await redisClient.connect();

const redisStore = new RedisStore({
  client: redisClient,
  prefix: 'myapp:sess:',   // namespaces session keys — important on shared Redis instances
  ttl: 86400,               // seconds; used when the session cookie has no explicit expiry
  disableTouch: false,       // false = TTL resets on every request (recommended default)
});

app.use(
  session({
    store: redisStore,
    resave: false,                 // required: don't re-save unchanged sessions
    saveUninitialized: false,      // recommended: don't create sessions until they have data
    secret: [
      process.env.SESSION_SECRET_CURRENT!,
      process.env.SESSION_SECRET_OLD!, // supports secret rotation without invalidating sessions
    ],
    rolling: true,                 // reset cookie expiration on every response
    name: 'sessionId',             // don't use the default 'connect.sid' — avoids fingerprinting
    cookie: {
      secure: process.env.NODE_ENV === 'production', // HTTPS only in prod
      httpOnly: true,
      sameSite: 'strict',
      maxAge: 24 * 60 * 60 * 1000, // 24 hours, in ms — note: ms here, seconds in ttl above
    },
  })
);

// If behind a load balancer/reverse proxy, required for secure cookies to work
app.set('trust proxy', 1);
```

```ts
// Dynamic TTL based on session content
const redisStore = new RedisStore({
  client: redisClient,
  ttl: (sess) => {
    // e.g. longer TTL for "remember me" sessions
    return sess.rememberMe ? 30 * 24 * 60 * 60 : 86400;
  },
});
```

```
Notes:
- If the session cookie has an explicit `expires` date, connect-redis uses
  that as the TTL instead of the `ttl` option.
- `prefix` appends to any prefix already set on the client itself — if your
  client has its own key prefix configured, account for the combined result.
- Use unique prefixes per application when multiple apps share one Redis
  instance — this scopes the bulk operations (`length`, `all`, `clear`)
  that express-session exposes to just your app's session keys.
- express-session does not update `req.session.cookie.expires` until the
  end of the request lifecycle — calling `session.save()` manually before
  that point uses the previous TTL value, not the one about to be set.
```

---

## Key Design Conventions

```ts
// Consistent namespace:entity:id pattern — makes SCAN patterns predictable
// and prevents accidental collisions between features
'user:1:profile'
'user:1:sessions'
'cache:product:42'
'cache:product:42:reviews'
'lock:job:sync-inventory'
'queue:emails:pending'
'rate-limit:api:user:1'

// Avoid overly long keys — they cost memory at scale; keep them
// descriptive but not verbose
// BAD: 'application:module:user-profile-data:user-id-12345:field-name-email'
// GOOD: 'user:12345:email'
```

---

## Common Patterns

### Rate Limiting

```ts
async function checkRateLimit(userId: string, limit: number, windowSeconds: number): Promise<boolean> {
  const key = `rate-limit:${userId}`;
  const count = await client.incr(key);
  if (count === 1) {
    await client.expire(key, windowSeconds); // only set TTL on first request in window
  }
  return count <= limit;
}
```

### Distributed Lock (Simple)

```ts
async function acquireLock(resource: string, ttlMs: number): Promise<boolean> {
  const result = await client.set(`lock:${resource}`, '1', {
    NX: true,           // only set if not already present
    PX: ttlMs,           // expire in milliseconds — lock auto-releases if holder crashes
  });
  return result === 'OK';
}

async function releaseLock(resource: string): Promise<void> {
  await client.del(`lock:${resource}`);
}

// For production distributed locking with correctness guarantees across
// multiple Redis nodes, use a dedicated Redlock implementation rather than
// this simplified single-node pattern.
```

### Cache-Aside

```ts
async function getProduct(id: string): Promise<Product> {
  const cacheKey = `cache:product:${id}`;
  const cached = await client.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const product = await db.products.findById(id);
  await client.set(cacheKey, JSON.stringify(product), { EX: 3600 });
  return product;
}

async function invalidateProductCache(id: string): Promise<void> {
  await client.del(`cache:product:${id}`);
}
```

---

## Redis Server 7.4 — What's Available

Your server (7.4) includes hash field-level TTL, introduced in 7.4:

```ts
// Set an expiration on individual hash fields (not the whole key) — 7.4+
await client.hExpire('user:1', ['sessionToken'], 300); // expire just this field in 5 minutes
await client.hTTL('user:1', ['sessionToken']); // check remaining TTL on the field
await client.hPersist('user:1', ['sessionToken']); // remove field-level TTL
```

---

## Forward-Looking: node-redis v6 (Not Yet Adopted)

> **Do not use these patterns yet** — informational only, for future migration
> planning when you're ready to test against v6.

- **RESP3 becomes the default protocol** (currently opt-in via `RESP: 3`) —
  this is the single biggest behavior change; test Pub/Sub and type mapping
  carefully since RESP3 delivery mechanics differ from RESP2
- **Node.js minimum version bumped** — verify your runtime meets the new
  floor before upgrading
- **Broader Redis 8.8 command coverage** — new array commands, `INCREX`/
  `INCREXBYFLOAT`, `ZINTER`/`ZUNION` `COUNT` aggregator, `XNACK`,
  `CLIENT UNBLOCK`
- **Sentinel & cluster pub/sub reliability fixes** for failover-moved
  connections and sharded topology recovery
- **First-class OpenTelemetry support** — metrics and Node.js
  `diagnostics_channel` integration, initialize before creating clients
- A v5 → v6 migration guide exists in the node-redis repo — read it in full
  before upgrading, don't rely solely on this summary

---

## Forward-Looking: Redis Server 8.x (Not Yet Adopted)

> **Your server is 7.4.** These notes exist so an eventual upgrade to 8.x
> doesn't surprise you. Do not write code assuming these commands/behaviors
> exist until the server is actually upgraded and tested.

### Licensing Change

Redis 8.0+ is licensed under **AGPLv3** (Redis returned to open source in
2025, after the 2024 SSPL/RSAL licensing change). This reverses the 2024
licensing controversy — worth knowing if your organization has open-source
licensing review processes tied to Redis version upgrades.

### Built-In Modules (Previously Separate)

Redis 8.0 merges what used to be separate Redis Stack modules directly into
core Redis: **JSON, Search, TimeSeries, Bloom filter, Cuckoo filter,
Count-min sketch, Top-k, t-digest**, plus a new **Vector set** data type
for similarity search (AI/semantic search use cases). If you ever adopted
Redis Stack modules separately, the 8.x upgrade removes the need for
separate module installation — but audit any module-specific commands
before upgrading, since module handling changes are one of the more
involved parts of a 7.4 → 8.0 migration.

### New Hash Commands (Building on 7.4's Field TTL)

```ts
// Available on Redis 8.0+ server only — will error against your 7.4 server
// HGETEX — fetch a hash field and optionally set an expiration in one call
// HSETEX — set a hash field and optionally set an expiration in one call
// HGETDEL — fetch and delete a hash field atomically
```

### Other Breaking Changes to Review Before Upgrading

- `GETRANGE` now returns an empty bulk when the negative end index is out
  of range (previously different behavior)
- `SCAN` command optimized when matching a specific data type
- New I/O threading implementation (`io-threads` config) — may need tuning
  after upgrade for multi-core throughput
- Improved replication mechanism — test replica behavior in staging before
  a production upgrade, replication internals changed
- CVE-2025-21605, CVE-2025-27151, CVE-2025-46818 and others were fixed
  across the 7.x → 8.0 line — check the full CVE list for your specific
  current patch version when planning the upgrade timeline, since some of
  these may warrant an earlier patch-level bump even before a full 8.x move

**When you're ready to test 8.x**: read the official Redis 8.0 release
notes and breaking changes page in full, and update the version block at
the top of this skill file once the migration is validated and shipped.

---

## Reference Files

Load these when the task goes deeper than the summaries above:

- **`references/data-structures.md`** — deep patterns for each Redis data
  type: hash field TTL details, sorted set ranking patterns, streams
  (`XADD`/`XREAD`), HyperLogLog, geospatial commands
- **`references/production-ops.md`** — connection resilience, monitoring,
  memory management, `MEMORY USAGE`/`OBJECT ENCODING` inspection, cluster
  vs standalone considerations, backup/persistence (RDB vs AOF)

> Socket.IO's Redis adapter (`@socket.io/redis-adapter`): see the **socket.io**
> skill, not this one.

*Last Updated: 2026-07-30*
