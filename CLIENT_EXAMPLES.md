# Client-Side SSE Examples: How to Stream Events from OpenCode

## The Question

> "Show me an example of how it works on the client. Is EventSource a built-in JavaScript object?"

## The Answer

**Yes, `EventSource` IS a built-in browser API**, but OpenCode's SDK **doesn't use it**. Instead, they use `fetch()` with manual SSE parsing. This works in both browsers AND Node.js (EventSource only works in browsers).

---

## What OpenCode Actually Does

### The Auto-Generated SDK Approach

OpenCode's SDK uses `fetch()` and manually parses the SSE stream. This is in the auto-generated code at `packages/sdk/js/src/gen/core/serverSentEvents.gen.ts`:

```typescript
// Simplified version of the actual code
export const createSseClient = ({ url, ...options }) => {
  const createStream = async function* () {
    // 1. Use standard fetch()
    const response = await fetch(url, {
      ...options,
      headers: {
        'Accept': 'text/event-stream',
        ...options.headers
      }
    })

    // 2. Get the readable stream
    const reader = response.body
      .pipeThrough(new TextDecoderStream())
      .getReader()

    let buffer = ""

    // 3. Read chunks from the stream
    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      buffer += value

      // 4. Split on double newlines (SSE event separator)
      const chunks = buffer.split("\n\n")
      buffer = chunks.pop() ?? ""  // Keep incomplete chunk

      // 5. Parse each complete event
      for (const chunk of chunks) {
        const lines = chunk.split("\n")
        const dataLines = []

        for (const line of lines) {
          if (line.startsWith("data:")) {
            dataLines.push(line.replace(/^data:\s*/, ""))
          }
        }

        if (dataLines.length) {
          const rawData = dataLines.join("\n")
          const data = JSON.parse(rawData)  // Parse JSON
          yield data  // Yield to consumer
        }
      }
    }
  }

  return { stream: createStream() }
}
```

**Key points:**
- ✅ Uses `fetch()` (works in browsers AND Node.js)
- ✅ Manual SSE parsing for full control
- ✅ Returns an async generator
- ✅ Automatic JSON parsing
- ✅ Built-in retry logic with exponential backoff

---

## Real Usage Examples

### Example 1: Basic Event Streaming (from Slack integration)

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk'

const client = createOpencodeClient({
  baseUrl: 'http://localhost:4096'
})

// Subscribe to all events
const events = await client.event.subscribe()

// Iterate over events as they arrive
for await (const event of events.stream) {
  console.log('Event type:', event.type)
  console.log('Event data:', event.properties)

  // Handle specific event types
  if (event.type === 'message.part.updated') {
    console.log('Agent said:', event.properties.text)
  }

  if (event.type === 'tool.call.started') {
    console.log('Tool called:', event.properties.toolName)
  }

  if (event.type === 'session.idle') {
    console.log('Session complete!')
    break
  }
}
```

**Real usage from `packages/slack/src/index.ts:22-29`:**
```typescript
const events = await opencode.client.event.subscribe()
for await (const event of events.stream) {
  if (event.type === "message.part.updated") {
    const part = event.properties.part
    if (part.type === "tool") {
      // Find the session for this tool update
      // ... handle tool updates in Slack
    }
  }
}
```

### Example 2: Streaming Events for a Session (from TUI)

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk'

const client = createOpencodeClient({
  baseUrl: 'http://localhost:4096'
})

// Create a session
const session = await client.session.create()
console.log('Session ID:', session.data.id)

// Send a prompt
await client.session.prompt({
  path: { id: session.data.id },
  body: {
    parts: [
      { type: 'text', text: 'List all TypeScript files in this project' }
    ]
  }
})

// Stream events for this specific session
const events = await client.event.subscribe({
  query: {
    directory: '/path/to/project'
  }
})

for await (const event of events.stream) {
  console.log('Event:', event.type)

  if (event.type === 'message.part.updated') {
    // Agent is streaming text
    process.stdout.write(event.properties.text)
  }

  if (event.type === 'session.idle') {
    console.log('\nDone!')
    break
  }
}
```

**Real usage from `packages/opencode/src/cli/cmd/tui/context/sdk.tsx:19-24`:**
```typescript
sdk.event.subscribe().then(async (events) => {
  for await (const event of events.stream) {
    console.log("event", event.type)
    emitter.emit(event.type, event)  // Forward to UI
  }
})
```

### Example 3: Complete Workflow (from Run command)

```typescript
import { createOpencodeClient, createOpencodeServer } from '@opencode-ai/sdk'

// 1. Spawn a server
const server = await createOpencodeServer({
  hostname: '127.0.0.1',
  port: 4096
})

// 2. Create client
const client = createOpencodeClient({
  baseUrl: server.url
})

// 3. Create session
const session = await client.session.create()

// 4. Start listening for events BEFORE sending prompt
const events = await client.event.subscribe()

const eventProcessor = (async () => {
  for await (const event of events.stream) {
    if (event.type === 'message.part.updated') {
      // Stream agent response to stdout
      process.stdout.write(event.properties.text || '')
    }

    if (event.type === 'tool.call.started') {
      console.log(`\n[Running: ${event.properties.toolName}]`)
    }

    if (event.type === 'session.error') {
      console.error('Error:', event.properties.message)
    }

    if (event.type === 'session.idle') {
      console.log('\n✓ Complete')
      break
    }
  }
})()

// 5. Send prompt
await client.session.prompt({
  path: { id: session.data.id },
  body: {
    parts: [
      { type: 'text', text: 'Analyze this codebase' }
    ]
  }
})

// 6. Wait for events to finish
await eventProcessor

// 7. Cleanup
server.close()
```

**Real usage from `packages/opencode/src/cli/cmd/run.ts:147-152`:**
```typescript
const events = await sdk.event.subscribe()
let errorMsg: string | undefined

const eventProcessor = (async () => {
  for await (const event of events.stream) {
    if (event.type === "message.part.updated") {
      // ... process events
    }
  }
})()
```

---

## Browser Example (Using Native EventSource)

While OpenCode's SDK doesn't use it, here's how you COULD use the native browser `EventSource` API directly:

```html
<!DOCTYPE html>
<html>
<head>
  <title>OpenCode Events</title>
</head>
<body>
  <div id="output"></div>

  <script>
    // Yes, EventSource is built-in to browsers!
    const eventSource = new EventSource('http://localhost:4096/event');

    // Listen for all events
    eventSource.onmessage = (e) => {
      const event = JSON.parse(e.data);
      console.log('Event:', event);

      // Display in page
      const div = document.getElementById('output');
      div.innerHTML += `<p>${event.type}: ${JSON.stringify(event.properties)}</p>`;

      // Handle specific events
      if (event.type === 'message.part.updated') {
        div.innerHTML += `<p><strong>Agent:</strong> ${event.properties.text}</p>`;
      }
    };

    // Handle errors
    eventSource.onerror = (err) => {
      console.error('EventSource error:', err);
      eventSource.close();
    };

    // Close when needed
    // eventSource.close();
  </script>
</body>
</html>
```

**EventSource features:**
- ✅ Built into all modern browsers
- ✅ Automatic reconnection
- ✅ Simple API
- ❌ Browser-only (doesn't work in Node.js)
- ❌ Can't customize headers (no Authorization, etc.)
- ❌ Can't use POST requests

**This is why OpenCode uses fetch() instead** - it works everywhere and gives full control.

---

## Python Client Example

```python
from opencode_ai import Opencode

client = Opencode(base_url="http://localhost:4096")

# Create session
session = client.session.create()

# Send prompt
client.session.prompt(
    id=session.id,
    prompt="Analyze this code"
)

# Stream events (synchronous)
for event in client.event.subscribe():
    print(f"Event: {event.type}")

    if event.type == "message.part.updated":
        print(event.properties.text, end='', flush=True)

    if event.type == "session.idle":
        print("\nDone!")
        break

# Or async version
async for event in await client.async_client.event.subscribe():
    print(f"Event: {event.type}")
```

**Real usage from `packages/sdk/python/src/opencode_ai/extras.py:137`:**
```python
with client.stream("GET", "/event", headers={"Accept": "text/event-stream"}, params=params) as r:
    r.raise_for_status()
    buf = ""
    for line_bytes in r.iter_lines():
        line = line_bytes.decode("utf-8")
        if line.startswith("data: "):
            data = json.loads(line[6:])
            yield data
```

---

## Go Client Example

```go
package main

import (
    "context"
    "fmt"
    "github.com/opencode-ai/sdk-go"
)

func main() {
    client := opencode.NewClient(
        opencode.WithBaseURL("http://localhost:4096"),
    )

    ctx := context.Background()

    // Create session
    session, err := client.Session.Create(ctx)
    if err != nil {
        panic(err)
    }

    // Send prompt
    _, err = client.Session.Prompt(ctx, session.ID, &opencode.PromptParams{
        Prompt: "Analyze this code",
    })
    if err != nil {
        panic(err)
    }

    // Stream events
    stream := client.Event.Subscribe(ctx)
    defer stream.Close()

    for stream.Next() {
        event := stream.Current()
        fmt.Printf("Event: %s\n", event.Type)

        if event.Type == "message.part.updated" {
            fmt.Print(event.Properties.Text)
        }

        if event.Type == "session.idle" {
            fmt.Println("\nDone!")
            break
        }
    }

    if err := stream.Err(); err != nil {
        panic(err)
    }
}
```

---

## Raw curl Example (No SDK)

For comparison, here's how you'd do it with just curl and jq:

```bash
#!/bin/bash

# Create session
SESSION_ID=$(curl -s -X POST http://localhost:4096/session | jq -r '.id')
echo "Session: $SESSION_ID"

# Send prompt (in background)
curl -X POST http://localhost:4096/session/$SESSION_ID/message \
  -H "Content-Type: application/json" \
  -d '{"parts":[{"type":"text","text":"List files"}]}' &

# Stream events and parse
curl -N http://localhost:4096/event | while IFS= read -r line; do
  if [[ $line == data:* ]]; then
    # Remove "data: " prefix
    json="${line#data: }"

    # Parse event type
    event_type=$(echo "$json" | jq -r '.type')
    echo "Event: $event_type"

    # Handle specific events
    case "$event_type" in
      "message.part.updated")
        text=$(echo "$json" | jq -r '.properties.text // ""')
        echo -n "$text"
        ;;
      "session.idle")
        echo -e "\nDone!"
        break
        ;;
    esac
  fi
done
```

---

## Advanced: Error Handling and Reconnection

The SDK has built-in retry logic with exponential backoff:

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk'

const client = createOpencodeClient({
  baseUrl: 'http://localhost:4096'
})

// With error handling and retry options
const events = await client.event.subscribe({
  // Custom options for SSE
  sseDefaultRetryDelay: 1000,      // Start with 1s delay
  sseMaxRetryDelay: 30000,          // Max 30s delay
  sseMaxRetryAttempts: 10,          // Retry up to 10 times

  // Error callback
  onSseError: (error) => {
    console.error('Connection error:', error)
  },

  // Event callback (optional - called for each event)
  onSseEvent: (event) => {
    console.log('Raw event:', event)
  }
})

try {
  for await (const event of events.stream) {
    console.log('Event:', event.type)

    if (event.type === 'session.idle') {
      break
    }
  }
} catch (error) {
  console.error('Stream error:', error)
}
```

**Retry behavior:**
- Attempt 1: Wait 1s
- Attempt 2: Wait 2s (exponential backoff)
- Attempt 3: Wait 4s
- Attempt 4: Wait 8s
- Attempt 5: Wait 16s
- Attempt 6: Wait 30s (capped at max)
- ... continues until max attempts

---

## Event Types Reference

Here are the main event types you'll see:

```typescript
type Event =
  | { type: 'server.connected', properties: {} }
  | { type: 'message.part.updated', properties: { text: string, part: Part } }
  | { type: 'tool.call.started', properties: { toolName: string, input: any } }
  | { type: 'tool.call.completed', properties: { output: any } }
  | { type: 'session.idle', properties: {} }
  | { type: 'session.error', properties: { message: string } }
  | { type: 'permission.updated', properties: Permission }

// Example handler
for await (const event of events.stream) {
  switch (event.type) {
    case 'server.connected':
      console.log('Connected!')
      break

    case 'message.part.updated':
      // Agent is generating text
      process.stdout.write(event.properties.text || '')
      break

    case 'tool.call.started':
      console.log(`\n[Tool: ${event.properties.toolName}]`)
      break

    case 'tool.call.completed':
      console.log('[Tool complete]')
      break

    case 'session.idle':
      console.log('\nAgent finished')
      break

    case 'session.error':
      console.error('Error:', event.properties.message)
      break

    case 'permission.updated':
      console.log('Permission request:', event.properties)
      break
  }
}
```

---

## Comparison: EventSource vs fetch()

| Feature | EventSource (built-in) | fetch() + manual parse (SDK) |
|---------|------------------------|------------------------------|
| **Environment** | Browser only | Browser + Node.js + Bun |
| **API** | Simple, high-level | Manual, full control |
| **Headers** | Limited (can't set Authorization) | Full header control |
| **POST requests** | ❌ No (GET only) | ✅ Yes |
| **Retry logic** | Automatic | Manual (but SDK has it) |
| **Reconnection** | Automatic | Manual (but SDK has it) |
| **TypeScript types** | Basic | Full type safety |
| **Error handling** | Limited | Full control |

**EventSource example:**
```javascript
// Simple but limited
const source = new EventSource('http://localhost:4096/event');
source.onmessage = (e) => console.log(JSON.parse(e.data));
```

**fetch() example (what SDK does):**
```javascript
// More code but full control
const response = await fetch('http://localhost:4096/event', {
  headers: {
    'Accept': 'text/event-stream',
    'Authorization': 'Bearer token123'  // Can set any headers!
  }
});

const reader = response.body.getReader();
// ... manual parsing with full control
```

---

## Key Takeaways

### 1. **EventSource IS built-in**, but OpenCode doesn't use it

✅ Yes, `EventSource` is a native browser API
❌ But it's browser-only and limited
✅ OpenCode uses `fetch()` for cross-platform support

### 2. **The SDK uses async generators**

```typescript
// Clean async iteration
for await (const event of events.stream) {
  console.log(event)
}
```

### 3. **Works everywhere**

- ✅ Browser
- ✅ Node.js
- ✅ Bun
- ✅ Deno
- ✅ Edge workers

### 4. **Built-in retry and reconnection**

The SDK automatically retries with exponential backoff if the connection drops.

### 5. **Type-safe**

Full TypeScript types for all events and properties.

---

## Complete Working Example

Here's a full example you can run:

```typescript
// example.ts
import { createOpencodeClient, createOpencodeServer } from '@opencode-ai/sdk'

async function main() {
  // 1. Start server
  console.log('Starting server...')
  const server = await createOpencodeServer({ port: 4096 })
  console.log('Server ready at', server.url)

  // 2. Create client
  const client = createOpencodeClient({ baseUrl: server.url })

  // 3. Create session
  const session = await client.session.create()
  console.log('Session created:', session.data.id)

  // 4. Start event listener
  console.log('Listening for events...')
  const events = await client.event.subscribe()

  const eventTask = (async () => {
    for await (const event of events.stream) {
      if (event.type === 'message.part.updated') {
        process.stdout.write(event.properties.text || '')
      } else if (event.type === 'session.idle') {
        console.log('\n✓ Done')
        break
      } else {
        console.log(`\n[${event.type}]`)
      }
    }
  })()

  // 5. Send prompt
  await client.session.prompt({
    path: { id: session.data.id },
    body: {
      parts: [
        { type: 'text', text: 'List the files in this directory' }
      ]
    }
  })

  // 6. Wait for completion
  await eventTask

  // 7. Cleanup
  server.close()
}

main().catch(console.error)
```

**Run it:**
```bash
bun example.ts
# or
node --loader tsx example.ts
```

---

## References

- **EventSource MDN:** https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- **Fetch API:** https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- **OpenCode SDK Code:** `packages/sdk/js/src/gen/core/serverSentEvents.gen.ts`
- **Real Usage Examples:**
  - Slack integration: `packages/slack/src/index.ts:22-29`
  - TUI: `packages/opencode/src/cli/cmd/tui/context/sdk.tsx:19-24`
  - Run command: `packages/opencode/src/cli/cmd/run.ts:147-152`

---

**Last Updated:** 2025-11-19
