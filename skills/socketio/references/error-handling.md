# Socket.IO 4.8.x — Error Handling & Resilience Reference

> Load this file for disconnect reason handling, reconnection tuning,
> acknowledgment timeouts, and graceful degradation patterns.

---

## Disconnect Reasons — What They Mean and What to Do

```ts
socket.on('disconnect', (reason, details) => {
  switch (reason) {
    case 'io server disconnect':
      // Server explicitly called socket.disconnect() — e.g. rate limit,
      // connection cap, or auth revocation. Client will NOT auto-reconnect
      // for this reason; if reconnection is desired, the client must call
      // socket.connect() manually.
      console.log('Server forcibly disconnected this socket');
      break;

    case 'io client disconnect':
      // Client explicitly called socket.disconnect() — intentional,
      // e.g. user closed the app or logged out. No action needed.
      break;

    case 'ping timeout':
      // Client didn't respond to a ping within pingTimeout — usually a
      // network issue (poor mobile signal, backgrounded app throttled by
      // OS). Client WILL auto-reconnect. Relevant for the diner-ordering
      // use case: this is the most common disconnect reason on mobile.
      console.log('Ping timeout — likely network issue, auto-reconnecting');
      break;

    case 'transport close':
      // Underlying connection was closed (e.g. user's device went to
      // sleep, network changed from WiFi to cellular). Client WILL
      // auto-reconnect.
      break;

    case 'transport error':
      // Connection encountered an error (e.g. server restarted, TLS
      // issue). Client WILL auto-reconnect.
      break;

    default:
      console.log('Unhandled disconnect reason:', reason);
  }

  // `details` (when present) can include the raw close event data —
  // useful for deeper debugging of transport-level issues
});
```

```ts
// Client-side: distinguish reconnectable vs terminal disconnects
socket.on('disconnect', (reason) => {
  if (reason === 'io server disconnect') {
    // Server rejected us on purpose — don't blindly reconnect without
    // understanding why (could be an auth issue that will just repeat)
    showReconnectPrompt(); // let the user decide, or refresh auth first
  }
  // For all other reasons, socket.io-client's built-in reconnection
  // logic handles it automatically — no action needed
});
```

---

## Reconnection Strategy Tuning

```ts
// Client-side reconnection config
const socket = io('https://api.example.com', {
  auth: { token: getStoredAuthToken() },
  reconnection: true,               // default true, explicit for clarity
  reconnectionAttempts: Infinity,    // keep trying — appropriate for a
                                       // mobile ordering app where losing
                                       // connection permanently is worse
                                       // than retrying indefinitely
  reconnectionDelay: 1000,            // initial delay before first retry
  reconnectionDelayMax: 5000,          // cap on exponential backoff
  randomizationFactor: 0.5,             // jitter — prevents thundering herd
                                          // if many clients disconnect at once
                                          // (e.g. server restart)
  timeout: 20000,                        // connection attempt timeout
});

socket.io.on('reconnect_attempt', (attempt) => {
  console.log(`Reconnection attempt ${attempt}`);
});

socket.io.on('reconnect', (attempt) => {
  console.log(`Reconnected after ${attempt} attempts`);
  // Good place to re-sync state that might have changed while disconnected
  // — e.g. re-fetch current order status via REST as a backstop, in case
  // connectionStateRecovery didn't cover the full gap
});

socket.io.on('reconnect_failed', () => {
  // All reconnection attempts exhausted (only reachable if
  // reconnectionAttempts is finite) — show a persistent offline indicator
  showOfflineBanner();
});
```

---

## Connection State Recovery — Gaps to Handle

```ts
// connectionStateRecovery (server config, shown in main SKILL.md) replays
// missed room memberships and buffered events after a brief disconnect —
// but it has limits:
// - Only covers gaps up to `maxDisconnectionDuration`
// - Does NOT persist across a full server restart unless using a
//   compatible adapter with persistence (the in-memory default does not survive restarts)
// - Does NOT recover events emitted to rooms the socket hadn't joined yet
//   at disconnect time

io.on('connection', (socket) => {
  if (socket.recovered) {
    // Connection state was successfully recovered — buffered events
    // during the gap have already been replayed to this socket
    console.log(`Socket ${socket.id} recovered previous session`);
  } else {
    // Either a fresh connection, or the gap exceeded maxDisconnectionDuration
    // — treat this as a cold start. For the diner-ordering case, this is
    // where you'd re-sync current order status via a REST call or an
    // explicit "resync" event, rather than assuming the client has an
    // up-to-date view
    console.log(`Socket ${socket.id} is a fresh connection or recovery expired`);
  }
});
```

```ts
// Client-side: always request a state resync on cold-start reconnects,
// don't assume connectionStateRecovery covered everything
socket.on('connect', () => {
  if (!socket.recovered) {
    fetchCurrentOrderStatus(); // REST fallback to ensure UI is accurate
  }
});
```

---

## Acknowledgment Timeouts

```ts
// Client: always set a timeout on emits expecting an acknowledgment —
// an unresponsive server (or one that silently drops the event) otherwise
// leaves the callback pending forever
socket.timeout(5000).emit('order:place', payload, (err, response) => {
  if (err) {
    // err is set specifically on TIMEOUT — the server never acknowledged
    // within 5000ms. Distinguish this from a business-logic failure
    // (which the server would send back as a normal response with an
    // error field, not by failing to respond at all)
    showError('Order submission timed out — please try again');
    return;
  }
  if (!response.success) {
    showError(response.error); // business-logic rejection, not a timeout
    return;
  }
  showOrderConfirmed(response.orderId);
});
```

```ts
// Server: always call the acknowledgment callback, even on error paths —
// a handler that returns early without calling callback() causes the
// client to hit its timeout unnecessarily, turning a fast business-logic
// rejection into a slow, confusing failure
socket.on('order:place', async (payload, callback) => {
  try {
    const validation = validateOrderPayload(payload);
    if (!validation.valid) {
      return callback({ success: false, error: validation.error }); // ✅ always called
    }
    const order = await createOrder(payload, socket.data.userId);
    callback({ success: true, orderId: order.id });
  } catch (err) {
    console.error('Order placement failed', err);
    callback({ success: false, error: 'Internal error — please try again' }); // ✅ still called
  }
});
```

---

## Graceful Degradation When Redis (Adapter) Is Unavailable

```ts
// If using @socket.io/redis-adapter and Redis becomes unreachable, the
// adapter's underlying client emits its own 'error' events — handle them,
// or an unhandled error can crash the process (same rule as the redis
// skill's connection setup)
pubClient.on('error', (err) => {
  console.error('Redis pub client error (Socket.IO adapter)', err);
  // Consider: does your app need to keep serving single-instance
  // functionality (same-server room broadcasts still work without Redis;
  // only cross-server broadcast breaks) or should it fail health checks
  // and let orchestration route traffic elsewhere?
});
subClient.on('error', (err) => {
  console.error('Redis sub client error (Socket.IO adapter)', err);
});
```

```
Important nuance: if the Redis connection backing the adapter drops,
Socket.IO does NOT automatically fail connections on the affected server —
clients already connected to that specific instance can still emit/receive
events scoped to sockets on the SAME instance. What breaks is cross-instance
delivery — a diner on server A won't get an update emitted from server B
until Redis connectivity is restored. Decide deliberately whether this
degraded-but-partially-functional state is acceptable for your SLA, or
whether the health check should fail hard to force traffic away from the
affected instance.
```

---

## Handling Malformed or Unexpected Payloads

```ts
// Never trust client payload shape, even with TypeScript event typing —
// TypeScript types are compile-time only and don't validate runtime data
// from a client that could be running old code, a modified client, or
// simply sending garbage
import { z } from 'zod';

const PlaceOrderSchema = z.object({
  restaurantId: z.string().uuid(),
  items: z.array(z.object({
    menuItemId: z.string().uuid(),
    quantity: z.number().int().positive().max(50),
  })).min(1).max(100),
  tableNumber: z.string().max(10).optional(),
});

socket.on('order:place', async (rawPayload, callback) => {
  const parsed = PlaceOrderSchema.safeParse(rawPayload);
  if (!parsed.success) {
    return callback({ success: false, error: 'Invalid order payload' });
  }
  const order = await createOrder(parsed.data, socket.data.userId);
  callback({ success: true, orderId: order.id });
});
```

---

## Catching Uncaught Errors in Event Handlers

```ts
// An unhandled promise rejection inside an async event handler does NOT
// crash the Socket.IO connection by default, but it DOES produce an
// unhandled rejection at the process level — which, depending on Node.js
// version and configuration, can crash the whole process
io.on('connection', (socket) => {
  // BAD: unhandled rejection if createOrder() throws
  socket.on('order:place', async (payload, callback) => {
    const order = await createOrder(payload, socket.data.userId); // ❌ no try/catch
    callback({ success: true, orderId: order.id });
  });
});

// GOOD: wrap every async handler, or use a wrapper utility
function safeHandler<T extends unknown[]>(
  handler: (...args: T) => Promise<void>
) {
  return async (...args: T) => {
    try {
      await handler(...args);
    } catch (err) {
      console.error('Unhandled error in socket event handler', err);
      // Optionally notify the client that something went wrong, if the
      // last argument is a callback function
      const maybeCallback = args[args.length - 1];
      if (typeof maybeCallback === 'function') {
        (maybeCallback as (r: unknown) => void)({ success: false, error: 'Internal error' });
      }
    }
  };
}

socket.on('order:place', safeHandler(async (payload, callback) => {
  const order = await createOrder(payload, socket.data.userId);
  callback({ success: true, orderId: order.id });
}));
```

---

## Engine-Level Errors

```ts
// io.engine surfaces lower-level connection errors before a Socket.IO
// 'connection' event even fires — useful for catching malformed
// handshakes, transport negotiation failures, etc.
io.engine.on('connection_error', (err) => {
  console.error('Engine.IO connection error:', {
    code: err.code,
    message: err.message,
    context: err.context,
  });
});
```

*Last Updated: 2026-07-31*
