# Redis Data Structures — Deep Reference

> Load this file for deep patterns on hash field TTL, sorted sets, streams,
> HyperLogLog, and geospatial commands. Targets Redis server 7.4 and
> node-redis client 5.12.1 (see main SKILL.md for version scope).

---

## Hash Field TTL (Redis 7.4+)

Your server version (7.4) introduced per-field expiration on hashes — a
genuinely new capability, not available on 7.2 or earlier. This is useful
for storing multiple pieces of user state with different lifetimes in one
hash instead of scattering them across separate keys.

```ts
// Set TTL on specific fields within a hash — other fields in the same
// hash are unaffected and don't expire
await client.hSet('user:1', {
  profile: JSON.stringify({ name: 'Alice' }),
  otpCode: '482913',
  sessionToken: 'abc123',
});

await client.hExpire('user:1', ['otpCode'], 300);       // expires in 5 min
await client.hExpire('user:1', ['sessionToken'], 1800);  // expires in 30 min
// 'profile' field has no TTL — persists until the hash key itself is deleted

// Check remaining TTL on a field
const ttls = await client.hTTL('user:1', ['otpCode', 'sessionToken']);
// Returns an array matching field order; -1 = no TTL, -2 = field doesn't exist

// Remove field-level TTL — field persists like normal again
await client.hPersist('user:1', ['sessionToken']);

// Set expiration only if field has no existing TTL
// mode is a plain string argument (NX/XX/GT/LT), not an options object
await client.hExpire('user:1', ['otpCode'], 300, 'NX');

// Practical use case: OTP codes and session tokens colocated with profile
// data, each with independent lifetimes, without needing separate keys
// or a background cleanup job
```

```ts
// Millisecond precision variants also exist
await client.hpExpire('user:1', ['otpCode'], 300000);   // ms
await client.hpTTL('user:1', ['otpCode']);                // remaining ms
```

---

## Sorted Sets — Ranking and Time-Ordered Patterns

```ts
// Leaderboard pattern
await client.zAdd('game:leaderboard', [
  { score: 1500, value: 'player:1' },
  { score: 2200, value: 'player:2' },
  { score: 1800, value: 'player:3' },
]);

// Top N (highest score first)
const top3 = await client.zRange('game:leaderboard', 0, 2, { REV: true });

// Rank of a specific member (0-indexed, ascending by default)
const rank = await client.zRevRank('game:leaderboard', 'player:1'); // rank from top

// Score of a specific member
const score = await client.zScore('game:leaderboard', 'player:1');

// Increment score atomically (e.g. adding points)
await client.zIncrBy('game:leaderboard', 50, 'player:1');

// Range by score — useful for "players between 1000-2000 points"
const midTier = await client.zRangeByScore('game:leaderboard', 1000, 2000);

// Remove a member
await client.zRem('game:leaderboard', 'player:3');
```

```ts
// Time-ordered feed pattern — use timestamp as score
async function addFeedItem(userId: string, itemId: string, timestamp: number) {
  await client.zAdd(`feed:${userId}`, [{ score: timestamp, value: itemId }]);
  // Trim to keep only the most recent 100 items — prevents unbounded growth
  await client.zRemRangeByRank(`feed:${userId}`, 0, -101);
}

async function getRecentFeedItems(userId: string, count: number) {
  return client.zRange(`feed:${userId}`, 0, count - 1, { REV: true });
}
```

```ts
// Sliding window rate limiting using sorted sets (more precise than simple counters)
async function slidingWindowRateLimit(
  userId: string,
  limit: number,
  windowMs: number
): Promise<boolean> {
  const key = `rate-limit:sliding:${userId}`;
  const now = Date.now();
  const windowStart = now - windowMs;

  const tx = client.multi();
  tx.zRemRangeByScore(key, 0, windowStart);       // drop entries outside the window
  tx.zAdd(key, [{ score: now, value: `${now}-${Math.random()}` }]); // unique member
  tx.zCard(key);                                    // count remaining entries
  tx.expire(key, Math.ceil(windowMs / 1000));       // cleanup if key goes idle

  const results = await tx.exec();
  const count = results[2] as number;
  return count <= limit;
}
```

---

## Streams — Event Logs and Message Queues

```ts
// Add an entry — '*' auto-generates a unique, time-ordered ID
const id = await client.xAdd('events:orders', '*', {
  type: 'order_created',
  orderId: '42',
  amount: '99.99',
});

// Read entries from the beginning
const entries = await client.xRange('events:orders', '-', '+');

// Read new entries as they arrive (blocking read)
const newEntries = await client.xRead(
  { key: 'events:orders', id: '$' },  // '$' = only new entries from now
  { BLOCK: 5000 }                       // block up to 5s waiting for new data
);

// Consumer groups — for distributing work across multiple consumers
await client.xGroupCreate('events:orders', 'order-processors', '0');

const groupEntries = await client.xReadGroup(
  'order-processors',
  'worker-1',
  { key: 'events:orders', id: '>' },  // '>' = only entries never delivered to this group
  { COUNT: 10 }
);

// Acknowledge processed entries
if (groupEntries) {
  for (const stream of groupEntries) {
    for (const entry of stream.messages) {
      await processOrder(entry.message);
      await client.xAck('events:orders', 'order-processors', entry.id);
    }
  }
}

// Trim the stream to prevent unbounded growth
await client.xTrim('events:orders', 'MAXLEN', 10000); // keep last ~10k entries
```

```
Streams vs Lists vs Pub/Sub for queue-like workloads:
- Lists (LPUSH/RPOP): simple FIFO, no replay, no consumer groups
- Pub/Sub: fire-and-forget, no persistence, subscribers must be connected
  at publish time or they miss the message
- Streams: persistent, replayable, consumer groups with acknowledgment
  and at-least-once delivery — the right choice for anything resembling
  a real job queue or event log
```

---

## HyperLogLog — Approximate Cardinality Counting

For "how many unique X" questions where exact precision isn't required and
memory efficiency matters (constant ~12KB regardless of set size):

```ts
// Track unique visitors without storing every visitor ID
await client.pfAdd('unique-visitors:2026-07-30', ['user:1', 'user:2', 'user:1']);
const uniqueCount = await client.pfCount('unique-visitors:2026-07-30');
// ~2 (duplicates don't inflate the count), with ~0.81% standard error

// Merge multiple HLLs (e.g. combining daily counts into a monthly estimate)
await client.pfMerge('unique-visitors:2026-07', [
  'unique-visitors:2026-07-01',
  'unique-visitors:2026-07-02',
  // ...
]);
```

Use this instead of a `SADD`-based set when you only need the count, not
the actual member list — a Set storing millions of unique IDs costs
megabytes; a HyperLogLog costs kilobytes regardless of cardinality.

---

## Geospatial Commands

```ts
// Store locations (longitude, latitude, member name)
await client.geoAdd('stores', [
  { longitude: -122.4194, latitude: 37.7749, member: 'store:sf' },
  { longitude: -73.9857, latitude: 40.7484, member: 'store:nyc' },
]);

// Find stores within a radius
const nearby = await client.geoSearch(
  'stores',
  { longitude: -122.4, latitude: 37.77 },
  { radius: 50, unit: 'km' }
);

// Distance between two members
const distance = await client.geoDist('stores', 'store:sf', 'store:nyc', 'km');

// Get coordinates of a member
const position = await client.geoPos('stores', 'store:sf');
```

---

## Bitmaps — Compact Flag Storage

```ts
// Track daily active users with one bit per user ID — extremely memory-efficient
await client.setBit('active-users:2026-07-30', 1042, 1); // mark user 1042 active

const wasActive = await client.getBit('active-users:2026-07-30', 1042);

// Count total active users that day
const activeCount = await client.bitCount('active-users:2026-07-30');

// Bitwise AND across multiple days — users active on ALL specified days
await client.bitOp('AND', 'active-both-days', [
  'active-users:2026-07-29',
  'active-users:2026-07-30',
]);
const bothDaysCount = await client.bitCount('active-both-days');
```

*Last Updated: 2026-07-30*
