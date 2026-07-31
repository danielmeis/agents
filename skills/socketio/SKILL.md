---
name: socketio
description: >
  Socket.IO 4.8.x best practices for real-time WebSocket applications:
  authentication middleware, rooms and namespaces, broadcasting scope,
  backpressure and volatile events, per-event rate limiting, horizontal
  scaling with @socket.io/redis-adapter, connection monitoring, and — most
  critically — correct test teardown to prevent Jest workers hanging on
  open socket/HTTP-server handles. Use this skill for any Socket.IO
  question: connection lifecycle, event design, CORS configuration, auth on
  handshake, disconnect handling, and production hardening. If you see or
  are asked about "A worker process has failed to exit gracefully" or Jest
  tests hanging in a WebSocket test suite, go straight to the Test Teardown
  section — this is a recurring, previously-diagnosed issue in this
  environment and the fix is a specific, required afterAll/afterEach order.
  @socket.io/redis-adapter is covered here (not in the redis skill) since
  its usage is Socket.IO-specific pub/sub wiring, not general Redis usage.
---

# Socket.IO 4.8.x Best Practices

> Current target: **socket.io 4.8.3**. Context: real-time order flow for a
> restaurant SaaS (diners ordering from phones) — connection reliability,
> room-scoped broadcasts (per-table, per-order, per-restaurant), and clean
> test teardown all matter more than they would in a lower-stakes app.

---

## ⚠️ If You're Here Because of a Hanging Jest Worker — Read This First

If you see:
```
A worker process has failed to exit gracefully and has been force exited.
This is likely caused by tests leaking due to improper teardown.
```

**Do not add `--forceExit` or `forceExit: true` to Jest config.** That hides
the real problem — an actual leaked handle (open socket, open HTTP server,
or a running interval) that will also leak in production under the same
conditions, just less visibly. Go straight to the
[Test Teardown](#test-teardown-critical) section below.

---

## Core Principles

- Authenticate on the handshake via `io.use()` middleware — never after
  `connection` fires, and never trust unauthenticated sockets to self-report
  identity in event payloads
- Scope every broadcast with rooms — `io.emit()` to everyone is almost
  always wrong once you have more than one tenant/table/order in flight
- Every event handler that accepts client input needs rate limiting —
  Socket.IO does not rate limit anything by default
- Use acknowledgments (`callback`) for events where the client needs to
  know the server actually received and processed them — Socket.IO's
  default delivery is at-most-once, not guaranteed
- Plan for horizontal scaling from day one if you'll ever run more than one
  Node.js process — retrofitting the Redis adapter after rooms/broadcast
  logic is written without it is more painful than starting with it
- Every `afterAll`/`afterEach` in a WebSocket test suite follows the exact
  teardown order in this skill — no exceptions, no `--forceExit`

---

## Server Setup

```ts
import express from 'express';
import { createServer } from 'node:http';
import { Server } from 'socket.io';

const app = express();
const httpServer = createServer(app);

const io = new Server(httpServer, {
  cors: {
    origin: process.env.ALLOWED_ORIGINS?.split(',') ?? ['http://localhost:3000'],
    credentials: true,
  },
  transports: ['websocket', 'polling'], // polling as fallback, not primary
  pingInterval: 25000,
  pingTimeout: 20000,
  connectionStateRecovery: {
    // Recovers missed events after a brief disconnect (e.g. mobile network
    // blip) — valuable for the diner-ordering use case where a phone
    // losing signal for a few seconds shouldn't drop order status updates
    maxDisconnectionDuration: 2 * 60 * 1000,
    skipMiddlewares: false, // re-run auth middleware on recovery, don't skip it
  },
});

httpServer.listen(process.env.PORT ?? 3000);
```

**Never use `origin: '*'` with `credentials: true`** — browsers reject this
combination, and even where they don't, wildcard CORS on an authenticated
WebSocket endpoint is a real security gap. Always enumerate allowed origins
explicitly, from an env var or config, never hardcoded inline for production.

---

## Authentication on Handshake

> The example below is written for a system handling payments and orders —
> treat every gap listed as a real requirement, not a nice-to-have.

```ts
import jwt from 'jsonwebtoken';
import { createClient } from 'redis';

const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.on('error', (err) => console.error('Redis Client Error', err));
await redisClient.connect();

// Secret rotation: verify against the current secret first, fall back to
// the previous one — mirrors the connect-redis session secret pattern
// elsewhere in this skill. Rotating a single static secret invalidates
// every connected session at once; this avoids that.
const JWT_SECRETS = [
  process.env.JWT_SECRET_CURRENT!,
  process.env.JWT_SECRET_PREVIOUS, // optional — omit once rotation window closes
].filter(Boolean) as string[];

function verifyToken(token: string): JwtPayload {
  let lastError: Error | undefined;
  for (const secret of JWT_SECRETS) {
    try {
      return jwt.verify(token, secret, {
        // CRITICAL: always pin the allowed algorithm(s) explicitly.
        // Without this, a token crafted with `alg: none`, or an RS256
        // token replayed against an HS256 verifier using the public key
        // as the HMAC secret, can bypass verification entirely — this is
        // a real, previously-exploited vulnerability class across
        // multiple JWT library ecosystems, not a theoretical concern.
        algorithms: ['HS256'],
        // Reject tokens minted for a different service/environment, even
        // if they happen to be signed with a secret this server also has
        issuer: 'restaurant-saas-api',
        audience: 'restaurant-saas-websocket',
      }) as JwtPayload;
    } catch (err) {
      lastError = err as Error;
    }
  }
  throw lastError ?? new Error('Token verification failed');
}

// Basic per-IP rate limiting on connection attempts, BEFORE jwt.verify()
// runs — an unauthenticated flood of connection attempts still costs CPU
// on verification; gate it before that cost is paid, not after
const connectionAttempts = new Map<string, { count: number; windowStart: number }>();
const MAX_ATTEMPTS_PER_WINDOW = 20;
const WINDOW_MS = 60_000;

function isRateLimited(ip: string): boolean {
  const now = Date.now();
  const entry = connectionAttempts.get(ip);
  if (!entry || now - entry.windowStart > WINDOW_MS) {
    connectionAttempts.set(ip, { count: 1, windowStart: now });
    return false;
  }
  entry.count++;
  return entry.count > MAX_ATTEMPTS_PER_WINDOW;
}

io.use(async (socket, next) => {
  const ip = socket.handshake.address;
  if (isRateLimited(ip)) {
    return next(new Error('Too many connection attempts'));
  }

  const token = socket.handshake.auth?.token;
  if (!token) {
    return next(new Error('Authentication required'));
  }

  try {
    const payload = verifyToken(token);

    // Revocation check — JWTs can't be invalidated before their natural
    // expiry by design, so maintain an explicit blocklist for logout and
    // for revoking access mid-session (e.g. an employee's shift ends, an
    // account is suspended). Keyed by the token's unique ID (jti claim),
    // not the whole token, and TTL'd to match the token's own expiry so
    // the blocklist entry doesn't outlive the token it's blocking.
    const isRevoked = await redisClient.exists(`revoked-token:${payload.jti}`);
    if (isRevoked) {
      return next(new Error('Session has been revoked'));
    }

    socket.data.userId = payload.userId;
    socket.data.role = payload.role;
    socket.data.restaurantId = payload.restaurantId;
    next();
  } catch {
    // Deliberately generic — do not leak whether the failure was
    // expiry, signature mismatch, or malformed structure. Never log the
    // raw token value; log only a hash or the userId claim if extractable
    // without full verification, for correlation without exposure.
    next(new Error('Invalid or expired token'));
  }
});

io.on('connection', (socket) => {
  console.log(`User ${socket.data.userId} connected`);
});
```

```ts
// Revoking a token on logout or account suspension — store by jti, TTL'd
// to the token's remaining lifetime so entries self-expire and the
// blocklist never grows unbounded
async function revokeToken(jti: string, expiresAt: number): Promise<void> {
  const ttlSeconds = Math.max(0, expiresAt - Math.floor(Date.now() / 1000));
  if (ttlSeconds > 0) {
    await redisClient.set(`revoked-token:${jti}`, '1', { EX: ttlSeconds });
  }
}

// When issuing tokens (outside this skill's scope, typically in your auth
// service), always include a unique jti claim so it can be targeted for
// revocation later:
// jwt.sign({ userId, role, restaurantId }, secret, {
//   algorithm: 'HS256', issuer: 'restaurant-saas-api',
//   audience: 'restaurant-saas-websocket', expiresIn: '2h', jwtid: crypto.randomUUID(),
// });
```

```ts
// TypeScript: type socket.data for full type safety across all handlers
interface SocketData {
  userId: string;
  role: 'diner' | 'staff' | 'admin';
  restaurantId: string;
}

interface ClientToServerEvents {
  'order:place': (payload: PlaceOrderPayload, callback: (result: OrderResult) => void) => void;
  'order:cancel': (orderId: string) => void;
}

interface ServerToClientEvents {
  'order:status_changed': (payload: OrderStatusPayload) => void;
  'order:error': (message: string) => void;
}

const io = new Server<ClientToServerEvents, ServerToClientEvents, {}, SocketData>(httpServer, {
  /* ... */
});
// Now socket.data, socket.emit(), and socket.on() are all fully typed —
// catches event name typos and payload shape mismatches at compile time
```

```ts
// Client-side — pass the token at connect time, not as a separate event
// after connecting (that window between connect and auth is a real gap)
import { io } from 'socket.io-client';

const socket = io('https://api.example.com', {
  auth: { token: getStoredAuthToken() },
  transports: ['websocket', 'polling'],
});

// Handle auth failures explicitly — don't let them fail silently
socket.on('connect_error', (err) => {
  if (err.message === 'Authentication required' || err.message === 'Invalid or expired token') {
    // redirect to login, refresh token, etc.
  }
  if (err.message === 'Too many connection attempts') {
    // back off before retrying — don't let socket.io-client's automatic
    // reconnection hammer the rate limiter further
  }
});
```

### Re-authenticating on Token Refresh

```ts
// Client: update the auth payload and force a reconnect when the token refreshes
function refreshSocketAuth(newToken: string) {
  socket.auth = { token: newToken };
  socket.disconnect().connect();
}
```

### Summary — What Makes This Production-Grade

| Gap in a naive implementation | Fix shown above |
|---|---|
| No algorithm restriction → algorithm confusion attack | Explicit `algorithms: ['HS256']` |
| No issuer/audience check → cross-service token replay | `issuer` + `audience` validation |
| Static secret → full session invalidation on rotation | Array of current + previous secrets |
| No revocation → can't force logout or ban mid-session | Redis blocklist keyed by `jti`, TTL'd to expiry |
| No pre-auth throttling → CPU exhaustion via connection flood | Per-IP rate limit before `jwt.verify()` runs |
| Verbose error messages → information leakage | Generic client-facing errors, no raw token logging |

---

## Rooms — Scope Every Broadcast

Rooms are the single most important tool for not accidentally broadcasting
one diner's order to every connected client on the platform.

```ts
io.on('connection', (socket) => {
  const { restaurantId, role } = socket.data;

  // Every socket joins a room scoped to their restaurant — this alone
  // prevents cross-tenant leakage of order events
  socket.join(`restaurant:${restaurantId}`);

  if (role === 'staff') {
    socket.join(`restaurant:${restaurantId}:staff`);
  }

  socket.on('order:place', async (payload, callback) => {
    const order = await createOrder(payload, socket.data.userId);

    // Join a room scoped to this specific order so the diner gets
    // status updates without needing to be in a restaurant-wide room
    socket.join(`order:${order.id}`);

    // Notify kitchen staff only — not every connected socket
    io.to(`restaurant:${restaurantId}:staff`).emit('order:new', order);

    callback({ success: true, orderId: order.id });
  });
});

// Elsewhere — e.g. a REST endpoint or background job updates order status
async function updateOrderStatus(orderId: string, status: OrderStatus) {
  await db.orders.update({ where: { id: orderId }, data: { status } });
  // Only the diner(s) watching this specific order receive the update —
  // not the whole restaurant, not the whole platform
  io.to(`order:${orderId}`).emit('order:status_changed', { orderId, status });
}
```

```ts
// NEVER do this for anything tenant/order-scoped — reaches every
// connected socket across every restaurant on the platform
io.emit('order:status_changed', { orderId, status }); // ❌

// Broadcasting to everyone except the sender
socket.broadcast.emit('user:typing', { userId: socket.data.userId }); // still unscoped — same problem, scope with .to() first
socket.to(`order:${orderId}`).broadcast.emit(/* ... */); // scoped + excludes sender
```

```ts
// Leaving rooms — clean up on disconnect, and on explicit state changes
socket.on('disconnect', () => {
  // Socket.IO automatically removes the socket from all rooms on
  // disconnect — you don't need to manually leave every room here.
  // Only handle app-level cleanup (e.g. marking a user as offline)
});

// Explicit leave when a diner navigates away from an order's tracking screen
socket.on('order:stop_watching', (orderId: string) => {
  socket.leave(`order:${orordId}`);
});
```

---

## Namespaces — When Rooms Aren't Enough

Use namespaces for genuinely different connection contexts with different
auth rules or event sets — not as a substitute for rooms within one context.

```ts
// Separate namespace for restaurant staff/kitchen displays vs diner apps —
// different auth requirements, different event vocabulary, different
// connection volume characteristics
const dinerNamespace = io.of('/diner');
const staffNamespace = io.of('/staff');

dinerNamespace.use((socket, next) => {
  // Diner-specific auth: lighter weight, phone-based session tokens
});

staffNamespace.use((socket, next) => {
  // Staff-specific auth: requires an employee role claim, shorter token TTL
});

staffNamespace.on('connection', (socket) => {
  socket.join(`restaurant:${socket.data.restaurantId}:kitchen`);
});
```

```ts
// Client connects to a specific namespace explicitly
const socket = io('https://api.example.com/diner', { auth: { token } });
```

**Rule of thumb**: if the distinction is "which subset of currently
connected users should get this message," use a room. If the distinction
is "this is a fundamentally different kind of client with different auth
and events," use a namespace.

---

## Backpressure and High-Volume Events

```ts
// volatile — fire-and-forget, drop if the connection isn't immediately
// ready to send. Good for data where only the latest value matters
// (e.g. live kitchen queue position, not order status transitions)
socket.volatile.emit('kitchen:queue_position', { position: 4 });

// Do NOT use volatile for anything the client must reliably receive —
// order confirmations, payment status, anything with business consequences
// if dropped. Use a normal emit with acknowledgment instead for those.
```

```ts
// Real backpressure risk: a slow client (bad connection, backgrounded
// mobile app) causes Socket.IO's internal write buffer for that socket to
// grow unbounded if you keep emitting to it. volatile.emit() only helps
// when the transport itself isn't writable yet — it does NOT prevent
// buffer growth from a genuinely slow consumer that IS still connected.

// For high-frequency updates (e.g. live order queue position updates
// every second), throttle at the source rather than relying on volatile
// alone:
import { throttle } from 'lodash';

const emitQueuePosition = throttle((restaurantId: string, position: number) => {
  io.to(`restaurant:${restaurantId}:staff`).emit('kitchen:queue_position', { position });
}, 1000, { leading: true, trailing: true });

// Call emitQueuePosition() as often as the underlying data changes —
// actual emits to clients are capped at once per second
```

```ts
// Batch multiple discrete updates into one emit when they arrive close
// together, instead of emitting each individually
class BatchedEmitter {
  private queue: OrderStatusPayload[] = [];
  private timer: NodeJS.Timeout | null = null;

  add(payload: OrderStatusPayload, room: string) {
    this.queue.push(payload);
    if (!this.timer) {
      this.timer = setTimeout(() => this.flush(room), 100); // 100ms batching window
    }
  }

  private flush(room: string) {
    if (this.queue.length > 0) {
      io.to(room).emit('order:batch_status_changed', this.queue);
      this.queue = [];
    }
    this.timer = null;
  }
}
```

For sustained high-volume scenarios, consider moving the event source
through a proper message queue (BullMQ, etc.) rather than emitting directly
from request handlers — this decouples event production rate from Socket.IO
emit rate and gives you retry/backoff for free.

---

## Rate Limiting Per Event Type

Socket.IO has no built-in rate limiting. Every handler accepting client
input needs it — a malicious or buggy client can otherwise flood a single
event as fast as the network allows.

```ts
// Simple token-bucket rate limiter, scoped per-socket, per-event-type
class RateLimiter {
  private buckets = new Map<string, { tokens: number; lastRefill: number }>();

  constructor(
    private maxTokens: number,
    private refillIntervalMs: number
  ) {}

  tryConsume(key: string): boolean {
    const now = Date.now();
    let bucket = this.buckets.get(key);

    if (!bucket) {
      bucket = { tokens: this.maxTokens, lastRefill: now };
      this.buckets.set(key, bucket);
    }

    const elapsed = now - bucket.lastRefill;
    const refillAmount = Math.floor(elapsed / this.refillIntervalMs) * this.maxTokens;
    if (refillAmount > 0) {
      bucket.tokens = Math.min(this.maxTokens, bucket.tokens + refillAmount);
      bucket.lastRefill = now;
    }

    if (bucket.tokens > 0) {
      bucket.tokens--;
      return true;
    }
    return false;
  }

  // Call this during test/server teardown — see Test Teardown section
  clear() {
    this.buckets.clear();
  }
}

const orderPlaceLimiter = new RateLimiter(5, 60_000); // 5 per minute

io.on('connection', (socket) => {
  socket.on('order:place', async (payload, callback) => {
    const key = `${socket.data.userId}:order:place`;
    if (!orderPlaceLimiter.tryConsume(key)) {
      return callback({ success: false, error: 'Too many orders placed — please wait' });
    }
    // ... proceed with order placement
  });
});
```

```ts
// Different events need different limits — a chat-style "typing" event
// tolerates much higher frequency than "order:place"
const typingLimiter = new RateLimiter(20, 10_000);   // 20 per 10s — chatty, low stakes
const orderLimiter   = new RateLimiter(5, 60_000);    // 5 per minute — high stakes, abuse-prone
const paymentLimiter  = new RateLimiter(3, 300_000);   // 3 per 5 minutes — very sensitive
```

```ts
// Periodic cleanup of stale buckets prevents unbounded memory growth
// from users who connect once and never come back. Store the interval
// handle — you'll need to clear it in tests (see Test Teardown).
function startRateLimitCleanup(limiter: RateLimiter, maxAgeMs: number) {
  return setInterval(() => {
    // prune buckets older than maxAgeMs
  }, 5 * 60_000);
}

const cleanupInterval = startRateLimitCleanup(orderLimiter, 30 * 60_000);

// MUST be callable from test teardown:
export function stopRateLimitCleanup() {
  clearInterval(cleanupInterval);
}
```

---

## Connection Monitoring

```ts
// Track connection count — essential for capacity planning and detecting
// runaway reconnect loops on the client side
let connectionCount = 0;

io.on('connection', (socket) => {
  connectionCount++;
  console.log(`Connections: ${connectionCount}`);

  socket.on('disconnect', (reason) => {
    connectionCount--;
    // Log the reason — distinguishes clean disconnects from network
    // issues, server-initiated disconnects, and ping timeouts
    console.log(`Socket ${socket.id} disconnected: ${reason}`);
  });
});

// Expose current count for monitoring/health endpoints
app.get('/health/websocket', (req, res) => {
  res.json({
    connections: connectionCount,
    // io.engine.clientsCount is the authoritative source if you didn't
    // track it manually, or want to cross-check your own counter
    engineClientCount: io.engine.clientsCount,
  });
});
```

```ts
// Per-user connection limits — prevent a single compromised or buggy
// client from opening unbounded connections
const MAX_CONNECTIONS_PER_USER = 5;
const userConnections = new Map<string, Set<string>>(); // userId -> Set<socketId>

io.on('connection', (socket) => {
  const { userId } = socket.data;
  const existing = userConnections.get(userId) ?? new Set();

  if (existing.size >= MAX_CONNECTIONS_PER_USER) {
    socket.emit('error', 'Connection limit exceeded');
    socket.disconnect(true);
    return;
  }

  existing.add(socket.id);
  userConnections.set(userId, existing);

  socket.on('disconnect', () => {
    existing.delete(socket.id);
    if (existing.size === 0) userConnections.delete(userId);
  });
});
```

```ts
// Idle timeout — disconnect sockets with no activity for too long
// (separate from Socket.IO's own ping/pong keepalive, which only proves
// the transport is alive, not that the user is actually engaged)
const IDLE_TIMEOUT_MS = 30 * 60_000;

io.on('connection', (socket) => {
  let idleTimer = setTimeout(() => socket.disconnect(true), IDLE_TIMEOUT_MS);

  const resetIdleTimer = () => {
    clearTimeout(idleTimer);
    idleTimer = setTimeout(() => socket.disconnect(true), IDLE_TIMEOUT_MS);
  };

  socket.onAny(resetIdleTimer); // any incoming event resets the timer

  socket.on('disconnect', () => clearTimeout(idleTimer));
});
```

---

## Horizontal Scaling with `@socket.io/redis-adapter`

Required the moment you run more than one Node.js process/instance — without
it, a diner connected to server A never receives events emitted from server B.

```ts
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const pubClient = createClient({ url: process.env.REDIS_URL });
// A Redis client in subscribe mode cannot publish — this is exactly why
// two separate clients are required, not a limitation you can work around
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

const io = new Server(httpServer, {
  cors: { /* ... */ },
  adapter: createAdapter(pubClient, subClient),
});

// From here, io.to(room).emit(), io.emit(), and broadcast all transparently
// route through Redis — no changes needed to any of the room/emit code
// shown elsewhere in this skill
```

```ts
// Sharded adapter — for Redis 7.0+, better cluster performance via
// sharded pub/sub channels instead of regular pub/sub
import { createShardedAdapter } from '@socket.io/redis-adapter';
io.adapter(createShardedAdapter(pubClient, subClient));
// Your server (redis:7.4-alpine per the redis skill) supports this
```

**Sticky sessions**: if any client falls back to HTTP long-polling (not
pure WebSocket), your load balancer must route repeat requests from the
same client to the same server process — polling relies on multiple HTTP
requests hitting the same instance. Pure WebSocket connections don't have
this requirement since the connection is a single persistent TCP stream.
With `transports: ['websocket', 'polling']`, budget for this in your load
balancer config (e.g. `ip_hash` in nginx, or cookie-based affinity) even
though `polling` is only the fallback.

```ts
// Alternative to sticky sessions if your infra makes them painful:
// force WebSocket-only, no polling fallback, and accept that very
// restrictive corporate proxies blocking WebSocket will fail to connect
const io = new Server(httpServer, {
  transports: ['websocket'], // no polling fallback — removes the sticky
                              // session requirement entirely, at the cost
                              // of dropping clients that can't do WebSocket
});
```

---

## Test Teardown (CRITICAL)

> This is a previously-diagnosed, recurring issue in this codebase. Follow
> this exactly — deviating from the order below is the most common cause
> of a Jest worker hanging on WebSocket tests.

**Never use `--forceExit` or `forceExit: true` in Jest config for WebSocket
tests.** It hides the real leak instead of fixing it — the same leak exists
in production, just without a visible symptom.

If you see:
```
A worker process has failed to exit gracefully and has been force exited.
This is likely caused by tests leaking due to improper teardown.
```

Run `npx jest --detectOpenHandles` to identify the leak. Common causes and
required fixes:

### 1. Socket.IO server not closed in the right order

`io.close()` must complete before closing the HTTP server. Required
teardown order in `afterAll`:

```ts
afterAll(async () => {
  stopRateLimitCleanup();                                            // 1. stop cleanup intervals (see Rate Limiting section)
  await new Promise<void>((resolve) => io.close(() => resolve()));   // 2. drain sockets
  httpServer.closeAllConnections();                                   // 3. kill keep-alive connections
  await new Promise<void>((resolve) => httpServer.close(() => resolve())); // 4. close server
  httpServer.removeAllListeners();
  io.removeAllListeners();
});

afterEach(() => {
  if (socket?.connected) socket.disconnect();
});
```

Every step matters and the order matters:
- **Step 1 first**: any interval/timeout your app code started (rate limit
  bucket cleanup, idle timers, batched emitters) must stop before anything
  else, or it can fire during teardown and re-create handles you just closed
- **Step 2 before step 3/4**: closing `io` first lets it gracefully notify
  connected sockets and clean up its internal state; closing the HTTP
  server first can leave Socket.IO's internals referencing a dead server
- **`closeAllConnections()`**: Node's HTTP server `.close()` alone waits for
  existing keep-alive connections to end naturally — this can hang
  indefinitely in tests where a client never explicitly closes its
  connection. `closeAllConnections()` forces them shut immediately.
- **`removeAllListeners()`**: prevents listener accumulation across test
  files if the same `io`/`httpServer` instance construction pattern repeats
  across suites and Jest doesn't fully garbage collect between them

### 2. Rate-limit cleanup interval

Any `setInterval` your app code starts (see the Rate Limiting section's
`stopRateLimitCleanup()` pattern) must be stopped explicitly in `afterAll`,
first, before closing `io`/`httpServer`. An `setInterval` handle left
running is exactly the kind of open handle `--detectOpenHandles` flags, and
it's invisible unless you're specifically looking for it because the test
itself still "passes."

### 3. Failed `expect()` inside a `done`-style callback

```ts
// BAD — if the expect() throws, done() is never called, and Jest waits
// for the timeout, then reports a hang rather than the actual assertion failure
it('receives order confirmation', (done) => {
  socket.on('order:confirmed', (payload) => {
    expect(payload.orderId).toBe('123'); // if this throws, done() never runs
    done();
  });
  socket.emit('order:place', mockOrder);
});

// GOOD — wrap in try/catch and pass the error to done() explicitly
it('receives order confirmation', (done) => {
  socket.on('order:confirmed', (payload) => {
    try {
      expect(payload.orderId).toBe('123');
      done();
    } catch (err) {
      done(err as Error);
    }
  });
  socket.emit('order:place', mockOrder);
});

// BETTER — convert to async/await with a Promise wrapper, avoids the
// done-callback footgun entirely
it('receives order confirmation', async () => {
  const payload = await new Promise<OrderConfirmedPayload>((resolve) => {
    socket.once('order:confirmed', resolve);
    socket.emit('order:place', mockOrder);
  });
  expect(payload.orderId).toBe('123');
});
```

### Full Test Setup Pattern

```ts
import { createServer } from 'node:http';
import { Server, type Socket as ServerSocket } from 'socket.io';
import { io as ioc, type Socket as ClientSocket } from 'socket.io-client';
import { AddressInfo } from 'node:net';

describe('order events', () => {
  let io: Server;
  let httpServer: ReturnType<typeof createServer>;
  let clientSocket: ClientSocket;
  let serverSocket: ServerSocket;
  let port: number;

  beforeAll((done) => {
    httpServer = createServer();
    io = new Server(httpServer);

    httpServer.listen(() => {
      port = (httpServer.address() as AddressInfo).port;
      io.on('connection', (socket) => {
        serverSocket = socket;
      });
      clientSocket = ioc(`http://localhost:${port}`);
      clientSocket.on('connect', done);
    });
  });

  afterAll(async () => {
    stopRateLimitCleanup();
    await new Promise<void>((resolve) => io.close(() => resolve()));
    httpServer.closeAllConnections();
    await new Promise<void>((resolve) => httpServer.close(() => resolve()));
    httpServer.removeAllListeners();
    io.removeAllListeners();
  });

  afterEach(() => {
    if (clientSocket?.connected) clientSocket.disconnect();
  });

  it('places an order and receives confirmation', async () => {
    const payload = await new Promise<OrderConfirmedPayload>((resolve) => {
      clientSocket.once('order:confirmed', resolve);
      clientSocket.emit('order:place', mockOrder);
    });
    expect(payload.orderId).toBeDefined();
  });
});
```

See `docs/api/websocket/websocket-guide.md` → Troubleshooting for the full
breakdown of this issue's original diagnosis, if further detail is needed
beyond what's captured here.

---

## Reference Files

Load these when the task goes deeper than the summaries above:

- **`references/error-handling.md`** — disconnect reason handling, engine-level
  errors, reconnection strategy tuning, acknowledgment timeouts, graceful
  degradation when Redis (adapter) is unavailable
- **`references/production-checklist.md`** — CORS hardening details, payload
  size limits, `maxHttpBufferSize`, observability/metrics wiring, deployment
  behind a reverse proxy (nginx/ALB WebSocket config), engine.io security
  settings

> Basic Redis client usage (non-Socket.IO): see the **redis** skill.

*Last Updated: 2026-07-31*
