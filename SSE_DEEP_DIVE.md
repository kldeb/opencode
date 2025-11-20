# Server-Sent Events (SSE) Deep Dive

## How OpenCode Streams Events with Plain HTTP

### The Question

> "How do they stream events using REST only?"
> `curl http://localhost:4096/session/{id}/event  # Stream events!`

### The Answer: Server-Sent Events (SSE)

Server-Sent Events is a **W3C web standard** (not a custom OpenCode thing) that enables real-time server-to-client streaming over plain HTTP.

---

## The Protocol

### What You See in curl

```bash
$ curl -N http://localhost:4096/session/abc123/event

data: {"type":"server.connected","properties":{}}

data: {"type":"message.part.updated","properties":{"text":"Hello"}}

data: {"type":"message.part.updated","properties":{"text":" world"}}

data: {"type":"tool.call","properties":{"toolName":"read","input":{"file":"README.md"}}}

data: {"type":"tool.result","properties":{"output":"..."}}

data: {"type":"session.idle","properties":{}}

# Connection stays open... waiting for more events...
```

**Note:** The `-N` flag tells curl to disable buffering, so you see events in real-time.

### What's Happening at the HTTP Level

#### 1. Client Request (curl)

```http
GET /session/abc123/event HTTP/1.1
Host: localhost:4096
Accept: text/event-stream
Cache-Control: no-cache
```

#### 2. Server Response Headers

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no

```

**Critical headers:**
- `Content-Type: text/event-stream` - Tells client this is SSE
- `Connection: keep-alive` - Connection stays open
- `Cache-Control: no-cache` - Don't cache events

#### 3. Server Streams Data

```
data: {"type":"server.connected","properties":{}}

data: {"type":"message.part.updated","properties":{"text":"Hello"}}

data: {"type":"message.part.updated","properties":{"text":" world"}}

```

**SSE format rules:**
- Each event starts with `data: `
- Events end with double newline (`\n\n`)
- JSON can be embedded in the data field
- Server can send events whenever it wants
- Connection stays open indefinitely

---

## Code Implementation

### Server Side (OpenCode)

From `packages/opencode/src/server/server.ts`:

```typescript
import { streamSSE } from "hono/streaming"

app.get("/event", async (c) => {
  log.info("event connected")

  return streamSSE(c, async (stream) => {
    // 1. Send immediate "hello" event
    stream.writeSSE({
      data: JSON.stringify({
        type: "server.connected",
        properties: {},
      }),
    })

    // 2. Subscribe to internal event bus
    const unsub = Bus.subscribeAll(async (event) => {
      // 3. Every time an event happens, push it to the stream
      await stream.writeSSE({
        data: JSON.stringify(event),
      })
    })

    // 4. Keep connection open until client disconnects
    await new Promise<void>((resolve) => {
      stream.onAbort(() => {
        unsub()           // Clean up subscription
        resolve()         // End the promise
        log.info("event disconnected")
      })
    })
  })
})
```

**Flow:**

```
Client connects
    ↓
Send "connected" event immediately
    ↓
Subscribe to internal event bus
    ↓
Wait for events...
    ↓
Event happens → writeSSE() → Client receives it
    ↓
Wait for events...
    ↓
Event happens → writeSSE() → Client receives it
    ↓
Client disconnects → onAbort() → Cleanup
```

### Client Side (JavaScript/Browser)

SSE is **built into browsers** via the `EventSource` API:

```javascript
// Create connection
const eventSource = new EventSource('http://localhost:4096/event');

// Listen for messages
eventSource.onmessage = (e) => {
  const event = JSON.parse(e.data);
  console.log('Event received:', event);

  if (event.type === 'message.part.updated') {
    console.log('Agent said:', event.properties.text);
  }
};

// Handle errors
eventSource.onerror = (err) => {
  console.error('Connection error:', err);
  eventSource.close();
};

// Close when done
// eventSource.close();
```

### Client Side (Node.js/SDK)

In Node.js, you parse the SSE stream manually:

```javascript
const response = await fetch('http://localhost:4096/session/abc123/event');

const reader = response.body.getReader();
const decoder = new TextDecoder();

let buffer = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  buffer += decoder.decode(value, { stream: true });

  // Split on double newlines (event separator)
  const events = buffer.split('\n\n');
  buffer = events.pop() || ''; // Keep incomplete event in buffer

  for (const event of events) {
    if (event.startsWith('data: ')) {
      const data = event.slice(6); // Remove "data: " prefix
      const parsed = JSON.parse(data);
      console.log('Event:', parsed);
    }
  }
}
```

### Client Side (Python)

```python
import requests
import json

response = requests.get(
    'http://localhost:4096/session/abc123/event',
    stream=True  # Important: enable streaming
)

for line in response.iter_lines():
    if line.startswith(b'data: '):
        data = json.loads(line[6:])  # Remove b'data: ' prefix
        print(f"Event: {data}")

        if data['type'] == 'session.idle':
            print("Session complete!")
            break
```

---

## Why SSE Instead of WebSockets?

### Comparison Table

| Feature | SSE (OpenCode) | WebSockets |
|---------|---------------|------------|
| **Direction** | Server → Client only | Bidirectional |
| **Protocol** | HTTP (works with curl!) | Custom protocol (ws://) |
| **Reconnection** | Automatic (built-in) | Manual implementation |
| **Firewalls** | Always works (HTTP) | Often blocked |
| **Proxies** | Standard HTTP proxies work | Need special proxy config |
| **Browser API** | `EventSource` (native) | `WebSocket` (native) |
| **curl support** | ✅ Yes | ❌ No |
| **Debugging** | Standard HTTP tools | Special WebSocket tools |
| **Complexity** | Very simple | More complex |

### For OpenCode's Use Case

OpenCode only needs **server → client** streaming:
- Server needs to push events to client
- Client sends requests via separate REST endpoints
- No need for client → server streaming

**Perfect use case for SSE!**

```
Client                          Server
  │                               │
  │──── POST /session/id/message ─→│  (REST request)
  │                               │
  │←─── data: {"type":"..."}  ────│  (SSE event)
  │←─── data: {"type":"..."}  ────│  (SSE event)
  │←─── data: {"type":"..."}  ────│  (SSE event)
  │                               │
  │──── POST /session/id/abort ──→│  (REST request)
  │                               │
```

---

## Real-World Example

Let's trace a complete interaction:

### Step 1: Create a session

```bash
$ SESSION_ID=$(curl -s -X POST http://localhost:4096/session | jq -r '.id')
$ echo $SESSION_ID
abc123xyz
```

### Step 2: Start listening for events (in background)

```bash
$ curl -N http://localhost:4096/session/$SESSION_ID/event &
data: {"type":"server.connected","properties":{}}

# Waiting for events...
```

### Step 3: Send a prompt

```bash
$ curl -X POST http://localhost:4096/session/$SESSION_ID/message \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What files are in this directory?"}'
```

### Step 4: Watch events stream in

```
data: {"type":"message.part.updated","properties":{"text":"I'll"}}

data: {"type":"message.part.updated","properties":{"text":" check"}}

data: {"type":"message.part.updated","properties":{"text":" the"}}

data: {"type":"message.part.updated","properties":{"text":" directory"}}

data: {"type":"tool.call.started","properties":{"toolName":"bash","input":{"command":"ls -la"}}}

data: {"type":"tool.call.completed","properties":{"output":"total 48\ndrwxr-xr-x  12 user  staff   384 Nov 19 10:00 .\n..."}}

data: {"type":"message.part.updated","properties":{"text":"The directory contains:\n- ARCHITECTURE.md\n- README.md\n..."}}

data: {"type":"session.idle","properties":{}}
```

All of this is **plain HTTP**, no special protocols!

---

## Advanced SSE Features (Not Used by OpenCode)

The SSE spec supports additional features:

### Event IDs (for reconnection)

```
id: 12345
data: {"type":"message"}

id: 12346
data: {"type":"message"}
```

Client can reconnect with `Last-Event-ID` header to resume from event 12346.

### Event Types

```
event: chat-message
data: {"text":"Hello"}

event: system-alert
data: {"level":"warning"}
```

Client can listen for specific event types:

```javascript
eventSource.addEventListener('chat-message', (e) => {
  console.log('Chat:', e.data);
});
```

### Retry Interval

```
retry: 10000
data: {"type":"message"}
```

Tells client to wait 10 seconds before reconnecting.

### OpenCode Uses Simple Format

OpenCode keeps it simple - just `data: <json>` with no IDs or event types. The event type is **inside** the JSON payload:

```
data: {"type":"message.part.updated","properties":{...}}
```

This makes it easier to consume with curl and simple clients.

---

## Why This Design is Brilliant

### 1. Works with curl

```bash
# No special tools needed!
curl -N http://localhost:4096/session/abc123/event
```

### 2. Works in any browser

```javascript
// Built-in browser API
const events = new EventSource('/session/abc123/event');
events.onmessage = (e) => console.log(e.data);
```

### 3. Works through corporate firewalls

- It's just HTTP on port 4096
- No special ports or protocols
- Proxies handle it like normal HTTP

### 4. Easy to debug

- Open browser DevTools → Network tab → See all events
- Use curl to inspect
- Standard HTTP debugging tools work

### 5. Infrastructure friendly

- Load balancers: Standard HTTP load balancing
- Reverse proxies: nginx, Caddy, Traefik all support it
- CDNs: Cloudflare, AWS CloudFront support SSE
- Monitoring: Standard HTTP metrics

### 6. Automatic reconnection

Browsers automatically reconnect if connection drops - no code needed!

---

## Common Pitfalls & Solutions

### Pitfall 1: Buffering

**Problem:** Some proxies buffer the response, so events arrive in batches instead of real-time.

**Solution:** Set headers to disable buffering:

```typescript
c.header('X-Accel-Buffering', 'no')  // For nginx
c.header('Cache-Control', 'no-cache')
```

### Pitfall 2: Timeouts

**Problem:** Load balancers might close idle connections.

**Solution:** Send periodic "heartbeat" events:

```typescript
const heartbeat = setInterval(() => {
  stream.writeSSE({ comment: 'heartbeat' })
}, 30000)  // Every 30 seconds
```

### Pitfall 3: Browser Connection Limits

**Problem:** Browsers limit SSE connections per domain (usually 6).

**Solution:**
- Use HTTP/2 (much higher limits)
- Share connections across tabs
- Close connections when not needed

---

## Comparison to Other Streaming Approaches

### 1. WebSockets

```javascript
// WebSocket
const ws = new WebSocket('ws://localhost:4096/events');
ws.onmessage = (e) => console.log(e.data);
```

**Pros:** Bidirectional, lower overhead
**Cons:** Can't use curl, special protocol, firewall issues

### 2. Long Polling

```javascript
async function poll() {
  const response = await fetch('/events?lastId=123');
  const events = await response.json();
  // Process events
  poll(); // Poll again
}
```

**Pros:** Works everywhere
**Cons:** Inefficient, latency, complex to implement

### 3. gRPC Streaming

```javascript
const call = client.streamEvents({});
call.on('data', (event) => console.log(event));
```

**Pros:** Efficient, bidirectional
**Cons:** Requires protobuf, can't use curl, complex setup

### 4. SSE (OpenCode's Choice)

```javascript
const events = new EventSource('/events');
events.onmessage = (e) => console.log(e.data);
```

**Pros:** Simple, curl-able, HTTP-based, auto-reconnect
**Cons:** Unidirectional only (but that's fine for this use case!)

---

## Complete Working Example

Here's a minimal SSE server in Node.js to demonstrate:

```javascript
// server.js
import express from 'express';

const app = express();

app.get('/events', (req, res) => {
  // Set SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Send initial event
  res.write('data: {"type":"connected"}\n\n');

  // Simulate events every second
  const interval = setInterval(() => {
    res.write(`data: {"type":"ping","time":"${new Date().toISOString()}"}\n\n`);
  }, 1000);

  // Cleanup on disconnect
  req.on('close', () => {
    clearInterval(interval);
    console.log('Client disconnected');
  });
});

app.listen(3000, () => {
  console.log('SSE server on http://localhost:3000/events');
});
```

**Test it:**

```bash
# Terminal 1: Start server
node server.js

# Terminal 2: Connect with curl
curl -N http://localhost:3000/events

# Output:
# data: {"type":"connected"}
#
# data: {"type":"ping","time":"2025-11-19T10:00:00.000Z"}
#
# data: {"type":"ping","time":"2025-11-19T10:00:01.000Z"}
#
# ...keeps streaming...
```

---

## Summary

**How OpenCode streams events with REST only:**

1. **Server-Sent Events (SSE)** - Web standard, not custom
2. **Plain HTTP GET request** - No special protocol
3. **Content-Type: text/event-stream** - Tells client it's SSE
4. **Connection stays open** - Server pushes events as they happen
5. **Works with curl** - No SDK required to test
6. **Built into browsers** - `EventSource` API
7. **Infrastructure friendly** - Standard HTTP tooling works

**The beauty:**

```bash
# It's just HTTP!
curl -N http://localhost:4096/session/abc123/event

# But it streams events in real-time
data: {"type":"message.part.updated","properties":{"text":"Hello"}}
data: {"type":"message.part.updated","properties":{"text":" world"}}
...
```

No WebSockets. No gRPC. No custom protocols. Just **HTTP doing what it was designed to do**.

That's the genius of OpenCode's "obsessively reduce friction" philosophy - they used a **web standard** that's been around since 2006, works everywhere, and can be tested with curl.

---

## References

- **SSE Spec:** https://html.spec.whatwg.org/multipage/server-sent-events.html
- **MDN EventSource:** https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- **OpenCode Server Code:** `packages/opencode/src/server/server.ts:1805-1827`
- **Hono Streaming Docs:** https://hono.dev/helpers/streaming

---

**Last Updated:** 2025-11-19
