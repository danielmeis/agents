# Socket.IO 4.8.x — Production Checklist Reference

> Load this file for CORS hardening, payload limits, reverse proxy
> configuration, and observability wiring.

---

## CORS Hardening

```ts
const io = new Server(httpServer, {
  cors: {
    origin: (origin, callback) => {
      const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') ?? [];
      // Allow no-origin requests (mobile apps, curl, some native WebView
      // contexts don't send an Origin header) only if you specifically
      // need to support them — otherwise reject
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error('Not allowed by CORS'));
      }
    },
    credentials: true,
    methods: ['GET', 'POST'], // Socket.IO's polling transport uses these
  },
});
```

```
Never:
- origin: '*' combined with credentials: true (browsers reject this
  combination outright, and treating it as a "works around CORS" hack
  if you drop credentials is its own security regression)
- origin: true (reflects the request's own Origin header back — equivalent
  to allowing everyone)
- Hardcoding origins inline for production — always source from env/config
  so a deploy doesn't require a code change to adjust allowed origins
```

---

## Payload Size Limits

```ts
// maxHttpBufferSize caps the size of a single message — default is 1MB.
// Set explicitly based on your actual payload needs; don't leave it at
// an arbitrary default without considering your worst-case legitimate
// payload (e.g. an order with many line items, or an image upload
// mistakenly routed through the socket instead of a REST/S3 upload path)
const io = new Server(httpServer, {
  maxHttpBufferSize: 1e6, // 1MB — appropriate for order payloads; do NOT
                            // raise this to accommodate file uploads —
                            // route those through REST/object storage instead
});
```

```ts
// Defense in depth: validate payload size/shape at the application layer
// too, not just relying on the transport-level cap — a payload just under
// maxHttpBufferSize but with a pathological item count is still a problem
const PlaceOrderSchema = z.object({
  items: z.array(orderItemSchema).max(100), // explicit cap, independent of byte size
});
```

---

## Deployment Behind a Reverse Proxy

### nginx

```nginx
# WebSocket upgrade headers — required, Socket.IO will silently fall back
# to polling-only (or fail entirely) without these
location /socket.io/ {
    proxy_pass http://backend_upstream;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # WebSocket connections are long-lived — default proxy timeouts will
    # kill idle-but-legitimate connections
    proxy_read_timeout 86400s;
    proxy_send_timeout 86400s;

    # Required ONLY if not using the Redis adapter and running multiple
    # backend instances — routes repeat requests from the same client to
    # the same upstream, needed for polling fallback to function at all
    ip_hash;
}

upstream backend_upstream {
    ip_hash; # or configure via a shared session store approach instead
    server backend1:3000;
    server backend2:3000;
}
```

### AWS ALB

```
- Enable "Stickiness" on the target group if not using the Redis adapter
  with WebSocket-only transport (see main SKILL.md's note on this tradeoff)
- Idle timeout: set higher than Socket.IO's pingInterval + pingTimeout
  combined, or the ALB will kill idle connections before Socket.IO's own
  keepalive has a chance to. Default ALB idle timeout is 60s — Socket.IO's
  default pingInterval (25s) + pingTimeout (20s) = 45s should fit, but
  verify against your actual configured values, and add margin.
- Target group protocol: HTTP1, not HTTP2 — WebSocket upgrade doesn't
  work the same way over HTTP/2 multiplexed connections in most ALB setups
```

---

## Engine.IO / Transport-Level Security

```ts
const io = new Server(httpServer, {
  // Restrict allowed transports — dropping long-polling entirely removes
  // an entire class of transport-level attack surface, at the cost of
  // clients behind restrictive proxies that block raw WebSocket
  transports: ['websocket'],

  // Reject connections that don't upgrade within a reasonable window
  upgradeTimeout: 10000,

  // Explicit path — don't leave this discoverable/guessable if you're
  // running Socket.IO alongside other services on the same domain
  path: '/socket.io/',

  // Disable the older EIO=3 protocol if you have no legacy clients —
  // reduces the negotiation surface
  allowEIO3: false,
});
```

---

## Observability

```ts
// Socket.IO ships no built-in metrics — instrument explicitly

let connectionCount = 0;
let totalEventsReceived = 0;
let totalEventsEmitted = 0;

io.on('connection', (socket) => {
  connectionCount++;

  // Wrap the socket's emit to count outbound events (careful: this
  // pattern intercepts application-level emits, not internal Engine.IO
  // packets)
  const originalEmit = socket.emit.bind(socket);
  socket.emit = ((...args: Parameters<typeof originalEmit>) => {
    totalEventsEmitted++;
    return originalEmit(...args);
  }) as typeof socket.emit;

  socket.onAny(() => {
    totalEventsReceived++;
  });

  socket.on('disconnect', () => {
    connectionCount--;
  });
});

// Expose via a metrics endpoint (Prometheus-style, or your APM of choice)
app.get('/metrics/websocket', (req, res) => {
  res.json({
    activeConnections: connectionCount,
    engineClientCount: io.engine.clientsCount,
    totalEventsReceived,
    totalEventsEmitted,
  });
});
```

```ts
// Structured connection lifecycle logging — invaluable when diagnosing
// production issues after the fact, since WebSocket issues are hard to
// reproduce on demand
io.on('connection', (socket) => {
  logger.info('socket_connected', {
    socketId: socket.id,
    userId: socket.data.userId,
    restaurantId: socket.data.restaurantId,
    transport: socket.conn.transport.name,
  });

  socket.on('disconnect', (reason) => {
    logger.info('socket_disconnected', {
      socketId: socket.id,
      userId: socket.data.userId,
      reason,
      durationMs: Date.now() - socket.handshake.issued,
    });
  });
});
```

```ts
// Room size monitoring — useful for spotting an unexpectedly large room
// (e.g. a bug that's putting every socket in one room instead of scoping
// correctly — exactly the kind of bug the Rooms section in the main
// SKILL.md is meant to prevent, but monitoring catches it if it slips
// through)
async function getRoomSize(room: string): Promise<number> {
  const sockets = await io.in(room).fetchSockets();
  return sockets.length;
}
```

---

## Memory Leak Prevention Checklist

```
- Every setInterval/setTimeout started for app-level logic (rate limit
  cleanup, idle timers, batched emitters) has a corresponding cleanup path
  — both in normal operation (e.g. on disconnect) and in test teardown
  (see main SKILL.md's Test Teardown section)
- Event listeners added via socket.on() inside a connection handler are
  scoped to that socket and cleaned up automatically on disconnect — but
  listeners added to io itself (not a per-socket socket) persist for the
  server's lifetime and accumulate if added repeatedly (e.g. inside a
  connection handler by mistake instead of once at startup)
- Maps/Sets tracking per-user or per-socket state (like the connection
  limit tracking in the main SKILL.md) MUST have corresponding delete
  calls on disconnect — an unbounded Map keyed by socket.id that's never
  pruned is a slow, hard-to-spot leak
- clientSideCache and RedisClientPool-style pooling patterns (see the
  redis skill) have their own lifecycle — make sure their cleanup is
  wired into the same shutdown sequence as the Socket.IO server itself
```

```ts
// BAD: listener added inside connection handler — a new listener is
// registered on `io` itself every single time ANY socket connects,
// accumulating without bound
io.on('connection', (socket) => {
  io.on('some-global-event', handleGlobalEvent); // ❌ leaks — registers again per connection
});

// GOOD: register io-level listeners once, outside any per-connection scope
io.on('some-global-event', handleGlobalEvent); // ✅ registered once at startup
io.on('connection', (socket) => {
  socket.on('order:place', handleOrderPlace); // ✅ per-socket, cleaned up on disconnect
});
```

---

## Graceful Server Shutdown (Production, Not Just Tests)

```ts
// The same teardown discipline from the Test Teardown section in the
// main SKILL.md applies in production — a server that doesn't drain
// connections cleanly on deploy/restart drops in-flight orders
async function gracefulShutdown() {
  console.log('Shutting down — draining WebSocket connections...');

  // Notify connected clients before disconnecting them, if your client
  // handles this event to show a "reconnecting" state instead of an
  // unexplained drop
  io.emit('server:shutting_down');

  stopRateLimitCleanup();

  await new Promise<void>((resolve) => io.close(() => resolve()));
  httpServer.closeAllConnections();
  await new Promise<void>((resolve) => httpServer.close(() => resolve()));

  await pubClient?.quit();
  await subClient?.quit();

  process.exit(0);
}

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

*Last Updated: 2026-07-31*
