# OpenCode Architecture Case Study

> "opencode server is a simple rest api - no websocket, no fancy rpc protocol - so you can just curl it or hit it in your browser. experience building things for devs teaches you that unless you obsessively reduce friction things never get adopted." — Dax (OpenCode Developer)

## Table of Contents

- [Executive Summary](#executive-summary)
- [Design Philosophy](#design-philosophy)
- [Architecture Overview](#architecture-overview)
- [The Simple REST API Approach](#the-simple-rest-api-approach)
- [Client-Server Architecture](#client-server-architecture)
- [SDK Design](#sdk-design)
- [Core Components](#core-components)
- [Technology Stack](#technology-stack)
- [Integration Patterns](#integration-patterns)
- [Key Takeaways](#key-takeaways)

---

## Executive Summary

**OpenCode** is a 100% open-source AI coding agent built for the terminal. It's a provider-agnostic alternative to Claude Code, supporting multiple AI providers (Anthropic, OpenAI, Google, local models, etc.). The project demonstrates exceptional architectural decisions that prioritize developer experience, extensibility, and simplicity.

### Key Differentiators

1. **Simple REST API** - No WebSockets, no complex RPC protocols, just HTTP
2. **Provider Agnostic** - Works with any LLM provider
3. **Client-Server Architecture** - Server can run locally or remotely
4. **Out-of-the-box LSP Support** - Deep IDE integration capabilities
5. **Multi-Protocol Support** - ACP for editors, MCP for tool extensions
6. **Terminal-First** - Built by terminal enthusiasts for terminal users

---

## Design Philosophy

### 1. Obsessive Friction Reduction

The core philosophy is captured in Dax's quote: "unless you obsessively reduce friction things never get adopted." This manifests in several ways:

- **Plain HTTP REST** instead of WebSockets or gRPC
- **OpenAPI-first** design that auto-generates SDKs
- **Simple curl commands** work out of the box
- **Browser-accessible** endpoints for debugging
- **Zero ceremony** client initialization

### 2. Developer Experience First

```bash
# Testing the API is as simple as:
curl http://localhost:4096/config

# Creating a session:
curl -X POST http://localhost:4096/session

# Sending a prompt:
curl -X POST http://localhost:4096/session/{id}/message \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Hello, world!"}'
```

This simplicity means:
- **No SDK required** for experimentation
- **Easy debugging** with standard HTTP tools
- **Instant comprehension** of how it works
- **Language agnostic** - any HTTP client works

### 3. Separation of Concerns

The architecture cleanly separates:
- **Server** (`packages/opencode/src/server/`) - REST API, session management, agent orchestration
- **Client** (`packages/sdk/`) - Thin wrappers around HTTP in multiple languages
- **UI** (`packages/desktop/`, `packages/console/`) - Multiple frontend options
- **Protocols** (`src/acp/`, `src/mcp/`) - Standard integration protocols

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER INTERFACES                          │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│ CLI/TUI      │ Web Console  │ Desktop App  │ Editor (Zed/VSC)  │
│ (OpenTUI)    │ (SolidStart) │ (Solid.js)   │ (via ACP)         │
└──────┬───────┴──────┬───────┴──────┬───────┴────────┬───────────┘
       │              │              │                │
       └──────────────┴──────────────┴────────────────┘
                      │
                      ▼
       ┌──────────────────────────────────┐
       │      SDK Layer (Client)          │
       │  JS/TS │ Go │ Python │ ACP      │
       └──────────────┬───────────────────┘
                      │ HTTP REST + SSE
                      ▼
       ┌──────────────────────────────────┐
       │     OpenCode Server (Hono)       │
       │  ┌────────────────────────────┐  │
       │  │   REST API Endpoints       │  │
       │  │   - Session Management     │  │
       │  │   - Config                 │  │
       │  │   - Project/File Ops       │  │
       │  │   - Event Streaming (SSE)  │  │
       │  └────────────────────────────┘  │
       └──────────────┬───────────────────┘
                      │
       ┌──────────────┴───────────────────┐
       │                                  │
       ▼                                  ▼
┌──────────────┐                 ┌─────────────────┐
│ Agent System │                 │  Tool Registry  │
├──────────────┤                 ├─────────────────┤
│ - Session    │                 │ - Bash          │
│ - Prompt     │ ◄───────────► │ - Edit/Write    │
│ - Message    │                 │ - Read/Glob     │
│ - Compaction │                 │ - Grep/Search   │
└──────┬───────┘                 │ - LSP           │
       │                         │ - MCP Tools     │
       ▼                         └─────────────────┘
┌─────────────────┐
│ Provider System │
├─────────────────┤
│ - Anthropic     │
│ - OpenAI        │
│ - Google        │
│ - Amazon        │
│ - Azure         │
│ - Local Models  │
└─────────────────┘
```

### Directory Structure

```
/home/user/opencode/
├── packages/
│   ├── opencode/          # Core CLI application (main package)
│   │   ├── src/
│   │   │   ├── server/    # REST API server (Hono)
│   │   │   ├── session/   # Session management
│   │   │   ├── agent/     # Agent system
│   │   │   ├── tool/      # Tool registry & implementations
│   │   │   ├── provider/  # LLM provider integrations
│   │   │   ├── acp/       # Agent Client Protocol
│   │   │   ├── mcp/       # Model Context Protocol
│   │   │   ├── lsp/       # Language Server Protocol
│   │   │   └── cli/       # CLI implementation
│   │   └── package.json
│   ├── sdk/
│   │   ├── js/            # JavaScript/TypeScript SDK
│   │   ├── go/            # Go SDK
│   │   └── python/        # Python SDK
│   ├── console/           # Web console application
│   │   ├── app/           # SolidStart web app
│   │   ├── core/          # Business logic, DB access
│   │   └── function/      # Serverless functions
│   ├── web/               # Documentation site (Astro + Starlight)
│   ├── desktop/           # Desktop application (Solid.js)
│   ├── ui/                # Shared UI components
│   └── util/              # Shared utilities
└── sdks/vscode/           # VS Code extension
```

---

## The Simple REST API Approach

### Why REST Over WebSockets/RPC?

The decision to use plain REST + Server-Sent Events (SSE) instead of WebSockets or gRPC is **brilliant** and demonstrates deep understanding of developer adoption:

#### 1. **Discoverability**

```bash
# Get the API documentation
curl http://localhost:4096/doc

# Returns OpenAPI 3.1 spec - can be opened in Swagger UI
```

The entire API is self-documenting through OpenAPI. Developers can:
- View it in browsers
- Import into Postman/Insomnia
- Generate clients in any language
- Understand the API without reading code

#### 2. **Debuggability**

```bash
# Check server status
curl http://localhost:4096/config

# List sessions
curl http://localhost:4096/session

# Get a specific session
curl http://localhost:4096/session/abc123
```

Debugging is trivial with curl, browser DevTools, or any HTTP client. No special tools needed.

#### 3. **Language Agnostic**

Any language with HTTP support (which is **all of them**) can use OpenCode:

```javascript
// JavaScript
fetch('http://localhost:4096/session', { method: 'POST' })
```

```python
# Python
import requests
requests.post('http://localhost:4096/session')
```

```go
// Go
http.Post("http://localhost:4096/session", "application/json", nil)
```

```bash
# Bash
curl -X POST http://localhost:4096/session
```

#### 4. **Infrastructure Friendly**

REST APIs work seamlessly with:
- **Load balancers** - Standard HTTP load balancing
- **Reverse proxies** - Nginx, Caddy, Traefik
- **API gateways** - AWS API Gateway, Kong, etc.
- **Caching** - Standard HTTP caching headers
- **Authentication** - Standard OAuth, JWT, API keys
- **Monitoring** - Standard HTTP metrics

WebSockets require special handling for all of these.

### Server-Sent Events for Real-Time Updates

For real-time updates, OpenCode uses **Server-Sent Events (SSE)** instead of WebSockets:

```bash
# Stream all events for a session
curl http://localhost:4096/session/abc123/event
```

**Why SSE over WebSockets?**

1. **Unidirectional** - Server → Client only (perfect for this use case)
2. **HTTP-based** - Works through firewalls, proxies
3. **Automatic reconnection** - Built into the standard
4. **Simpler** - No handshake, no binary protocols
5. **Browser native** - `EventSource` API built-in

```javascript
// Browser usage
const events = new EventSource('http://localhost:4096/session/abc123/event');
events.onmessage = (e) => {
  const data = JSON.parse(e.data);
  console.log('Event:', data);
};
```

### Core API Endpoints

The server exposes a comprehensive REST API (see `packages/opencode/src/server/server.ts:85-1862`):

#### Configuration
- `GET /config` - Get configuration
- `PATCH /config` - Update configuration
- `GET /config/providers` - List available LLM providers

#### Sessions
- `GET /session` - List all sessions
- `POST /session` - Create a new session
- `GET /session/:id` - Get session details
- `DELETE /session/:id` - Delete a session
- `PATCH /session/:id` - Update session properties

#### Messages & Prompts
- `GET /session/:id/message` - List messages in a session
- `POST /session/:id/message` - Send a prompt (user message)
- `POST /session/:id/command` - Execute a command
- `POST /session/:id/shell` - Run a shell command

#### Events (Real-time)
- `GET /event` - Instance-level event stream (SSE)
- `GET /global/event` - Global event stream (SSE)
- `GET /session/:id/event` - Session-specific event stream (SSE)

#### File Operations
- `GET /file` - List files and directories
- `GET /file/content` - Read a file
- `GET /file/status` - Get file status (git)

#### Search
- `GET /find` - Search text in files (ripgrep)
- `GET /find/file` - Find files by name
- `GET /find/symbol` - Find workspace symbols (LSP)

#### Tools & Integrations
- `GET /experimental/tool/ids` - List available tool IDs
- `GET /experimental/tool` - List tools with schemas
- `GET /lsp` - LSP server status
- `GET /mcp` - MCP server status
- `POST /mcp` - Add MCP server dynamically

#### Documentation
- `GET /doc` - OpenAPI specification

---

## Client-Server Architecture

### Instance Isolation

One of the most clever architectural decisions is **Instance isolation** (`packages/opencode/src/server/server.ts:166-175`):

```typescript
.use(async (c, next) => {
  const directory = c.req.query("directory")
                 ?? c.req.header("x-opencode-directory")
                 ?? process.cwd()
  return Instance.provide({
    directory,
    init: InstanceBootstrap,
    async fn() {
      return next()
    },
  })
})
```

**What this means:**

1. Each request can specify a working directory
2. Different clients can work on different projects simultaneously
3. One server can handle multiple projects
4. Context is isolated per directory

**Usage:**

```bash
# Work on project A
curl -H "x-opencode-directory: /home/user/project-a" \
  http://localhost:4096/session

# Work on project B
curl -H "x-opencode-directory: /home/user/project-b" \
  http://localhost:4096/session
```

### Server Modes

OpenCode can run in three modes:

#### 1. Interactive TUI (Default)

```bash
opencode
```

Starts the terminal UI with an embedded server.

#### 2. Headless Server

```bash
opencode serve --port 4096
```

Runs just the HTTP server, no UI. Perfect for:
- Remote access
- Mobile clients
- Web frontends
- Automation

#### 3. Ephemeral (Run mode)

```bash
opencode run "explain this code"
```

Starts server, executes command, shuts down.

---

## SDK Design

### Auto-Generated from OpenAPI

The SDK is **not hand-written** - it's auto-generated from the OpenAPI spec (`packages/sdk/js/package.json:18`):

```json
{
  "devDependencies": {
    "@hey-api/openapi-ts": "0.81.0"
  }
}
```

**Generation Process:**

1. Server exposes OpenAPI spec at `/doc`
2. SDK generation tool reads the spec
3. Type-safe clients generated for JS/Go/Python
4. Updated on every API change

**Benefits:**

- **Always in sync** - SDK matches server 100%
- **Type safety** - Full TypeScript types
- **Zero maintenance** - No manual updates needed
- **Consistency** - Same API across all languages

### JavaScript/TypeScript SDK

**Installation:**

```bash
npm install @opencode-ai/sdk
```

**Usage:**

```typescript
import { createOpencodeClient, createOpencodeServer } from '@opencode-ai/sdk'

// Option 1: Connect to existing server
const client = createOpencodeClient({
  baseUrl: 'http://localhost:4096',
  directory: '/path/to/project'
})

// Option 2: Spawn a server
const server = await createOpencodeServer({
  hostname: '127.0.0.1',
  port: 4096
})
const client = createOpencodeClient({ baseUrl: server.url })

// Use the client
const sessions = await client.session.list()
const session = await client.session.create()
await client.session.prompt({
  id: session.id,
  prompt: 'Hello, world!'
})

// Stream events
const events = await client.session.events(session.id)
for await (const event of events) {
  console.log(event)
}

// Clean up
server.close()
```

### Key SDK Features

#### 1. **Server Spawning** (`packages/sdk/js/src/server.ts:21-88`)

```typescript
export async function createOpencodeServer(options?: ServerOptions) {
  const proc = spawn(`opencode`, [`serve`, `--hostname=${options.hostname}`, `--port=${options.port}`], {
    signal: options.signal,
    env: {
      ...process.env,
      OPENCODE_CONFIG_CONTENT: JSON.stringify(options.config ?? {}),
    },
  })

  // Wait for server to be ready by parsing stdout
  const url = await new Promise<string>((resolve, reject) => {
    proc.stdout?.on("data", (chunk) => {
      if (line.startsWith("opencode server listening")) {
        const match = line.match(/on\s+(https?:\/\/[^\s]+)/)
        resolve(match[1]!)
      }
    })
  })

  return {
    url,
    close() {
      proc.kill()
    },
  }
}
```

This allows **embedded usage**:

```typescript
// Your app can spawn and manage OpenCode
const server = await createOpencodeServer()
// ... use it ...
server.close()
```

#### 2. **Directory Context** (`packages/sdk/js/src/client.ts:20-25`)

```typescript
if (config?.directory) {
  config.headers = {
    ...config.headers,
    "x-opencode-directory": config.directory,
  }
}
```

Every request automatically includes the working directory.

#### 3. **TUI Spawning** (`packages/sdk/js/src/server.ts:90-120`)

```typescript
export function createOpencodeTui(options?: TuiOptions) {
  const proc = spawn(`opencode`, args, {
    signal: options?.signal,
    stdio: "inherit",
  })

  return {
    close() {
      proc.kill()
    },
  }
}
```

You can launch the TUI programmatically from your application.

---

## Core Components

### 1. Server (`packages/opencode/src/server/`)

Built with **Hono** - a lightweight, fast web framework:

```typescript
import { Hono } from "hono"
import { cors } from "hono/cors"
import { streamSSE } from "hono/streaming"

const app = new Hono()
  .use(cors())
  .get("/config", async (c) => {
    return c.json(await Config.get())
  })
  .post("/session", async (c) => {
    const session = await Session.create(c.req.valid("json"))
    return c.json(session)
  })
```

**Key Features:**

- **OpenAPI Validation** - `hono-openapi` for automatic validation
- **Zod Schemas** - Type-safe validation with Zod
- **Error Handling** - Centralized error handling
- **Logging** - Structured logging throughout
- **CORS Enabled** - Cross-origin requests supported

### 2. Session Management (`packages/opencode/src/session/`)

Sessions are the core abstraction:

- **Persistent** - Stored to disk, survive restarts
- **Hierarchical** - Sessions can have children (forking)
- **Message History** - All messages stored
- **Compaction** - Summarize old messages to save tokens
- **Revert** - Undo changes, fork from any point

**Session Lifecycle:**

```
Create → Prompt → Agent processes → Stream events → Idle
           ↑                                         │
           └─────────── Continue ────────────────────┘
```

### 3. Agent System (`packages/opencode/src/agent/`)

Two built-in agents:

- **build** - Full access, default for development work
- **plan** - Read-only, asks permission for bash, ideal for exploration

Agents can be switched with `Tab` key in the TUI.

### 4. Tool Registry (`packages/opencode/src/tool/`)

Tools are the "hands" of the agent:

- **Bash** - Execute shell commands
- **Read/Write/Edit** - File operations
- **Glob/Grep** - Search files
- **LSP** - Language server queries
- **MCP** - Extended tools from MCP servers

Tools are registered dynamically and can be extended via MCP.

### 5. Provider System (`packages/opencode/src/provider/`)

Provider-agnostic LLM integration using Vercel AI SDK:

```typescript
import { generateText, streamText } from 'ai'

// Works with any provider
const result = await generateText({
  model: anthropic('claude-3-5-sonnet-20241022'),
  // or
  model: openai('gpt-4'),
  // or
  model: google('gemini-pro'),

  messages,
  tools,
})
```

**Supported Providers:**

- Anthropic (Claude)
- OpenAI (GPT-4, GPT-3.5)
- Google (Gemini, Vertex AI)
- Amazon Bedrock
- Azure OpenAI
- Local models (Ollama, etc.)

### 6. Protocol Integrations

#### ACP (Agent Client Protocol) (`packages/opencode/src/acp/`)

Standard protocol for editor integration:

- **Used by:** Zed, VS Code (via extension)
- **Protocol:** JSON-RPC over stdio
- **Purpose:** Editors can control OpenCode agents

#### MCP (Model Context Protocol) (`packages/opencode/src/mcp/`)

Extends agent capabilities with external tools:

- **Dynamic loading** - Add MCP servers at runtime
- **Tool discovery** - Automatic tool registration
- **Multi-server** - Use multiple MCP servers simultaneously

---

## Technology Stack

### Core Technologies

| Component | Technology | Why? |
|-----------|-----------|------|
| **Runtime** | Bun | Fast, batteries-included, TypeScript native |
| **Server Framework** | Hono | Lightweight, fast, edge-compatible |
| **Validation** | Zod | Type-safe schemas, runtime validation |
| **OpenAPI** | hono-openapi | Auto-generate docs & validation |
| **LLM Integration** | Vercel AI SDK | Provider-agnostic, streaming support |
| **TUI Framework** | OpenTUI + Solid.js | Reactive terminal UI |
| **Web Framework** | SolidStart | Full-stack web app |
| **Docs Site** | Astro + Starlight | Fast, beautiful docs |
| **Desktop** | Solid.js | Reactive desktop app |
| **Monorepo** | Turborepo | Fast, efficient builds |

### Key Dependencies

```json
{
  "dependencies": {
    // Server & API
    "hono": "^4.x",
    "hono-openapi": "^1.x",
    "zod": "^3.x",

    // LLM Integration
    "ai": "^3.x",
    "@ai-sdk/anthropic": "^x.x",
    "@ai-sdk/openai": "^x.x",

    // Protocols
    "@agentclientprotocol/sdk": "^0.5.x",
    "@modelcontextprotocol/sdk": "^1.x",

    // TUI
    "@opentui/core": "^0.1.x",
    "@opentui/solid": "^0.1.x",
    "solid-js": "^1.x",

    // Tools
    "tree-sitter": "^0.x",
    "vscode-jsonrpc": "^8.x",
    "@parcel/watcher": "^2.x"
  }
}
```

---

## Integration Patterns

### Pattern 1: Embedded AI Agent

**Use Case:** Add AI coding capabilities to your application

```typescript
import { createOpencodeServer, createOpencodeClient } from '@opencode-ai/sdk'

async function addAICapabilities() {
  // Spawn a server
  const server = await createOpencodeServer({ port: 4096 })

  // Create client
  const client = createOpencodeClient({
    baseUrl: server.url,
    directory: '/path/to/project'
  })

  // Create a session
  const session = await client.session.create()

  // Send prompts
  const response = await client.session.prompt({
    id: session.id,
    prompt: 'Analyze this codebase and suggest improvements'
  })

  // Stream events
  const events = await client.session.events(session.id)
  for await (const event of events) {
    if (event.type === 'message.part.updated') {
      console.log('Progress:', event.properties.text)
    }
  }

  return { server, client, session }
}
```

### Pattern 2: Remote Control

**Use Case:** Control OpenCode from a different device (e.g., mobile app)

```typescript
// On your computer: Start headless server
// $ opencode serve --hostname 0.0.0.0 --port 4096

// From mobile app / web app
const client = createOpencodeClient({
  baseUrl: 'http://your-computer-ip:4096',
  directory: '/home/user/project'
})

// Now you can control OpenCode from anywhere
const sessions = await client.session.list()
await client.session.prompt({
  id: sessions[0].id,
  prompt: 'Refactor the main function'
})
```

### Pattern 3: CI/CD Integration

**Use Case:** Use OpenCode in automated workflows

```bash
#!/bin/bash
# ci-review.sh

# Start server in background
opencode serve &
SERVER_PID=$!

# Wait for server to be ready
sleep 2

# Use curl to interact
SESSION_ID=$(curl -X POST http://localhost:4096/session | jq -r '.id')

curl -X POST http://localhost:4096/session/$SESSION_ID/message \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Review this PR and suggest improvements"}'

# Stream events until done
curl http://localhost:4096/session/$SESSION_ID/event

# Cleanup
kill $SERVER_PID
```

### Pattern 4: Editor Integration (ACP)

**Use Case:** Integrate with editors like Zed or VS Code

Editors use the **Agent Client Protocol** to communicate with OpenCode:

```typescript
import { ACPAgent } from '@agentclientprotocol/sdk'

// Editor creates an ACP agent
const agent = new ACPAgent({
  command: 'opencode',
  args: ['acp'],
})

// Send requests
const response = await agent.request('textDocument/codeAction', {
  textDocument: { uri: 'file:///path/to/file.ts' },
  range: { start: { line: 10, character: 0 }, end: { line: 15, character: 0 } },
})
```

### Pattern 5: MCP Tool Extension

**Use Case:** Add custom tools to OpenCode

```typescript
// Add an MCP server dynamically
await client.mcp.add({
  name: 'custom-tools',
  config: {
    command: 'node',
    args: ['custom-mcp-server.js']
  }
})

// Now agent can use tools from custom-mcp-server
await client.session.prompt({
  id: session.id,
  prompt: 'Use the custom tool to fetch data'
})
```

---

## Key Takeaways

### 1. Simplicity Wins

The decision to use **plain REST + SSE** instead of WebSockets or gRPC reduces friction massively:

- ✅ Works with curl
- ✅ Visible in browser
- ✅ Standard HTTP tooling
- ✅ Easy debugging
- ✅ Language agnostic
- ✅ Infrastructure friendly

### 2. OpenAPI-First Design

Auto-generating SDKs from OpenAPI spec is genius:

- ✅ Always in sync
- ✅ Type-safe
- ✅ Self-documenting
- ✅ Multi-language support
- ✅ Zero maintenance

### 3. Client-Server Separation

Having a headless server mode enables:

- ✅ Remote access
- ✅ Multiple frontends (TUI, web, mobile)
- ✅ Embedded usage
- ✅ CI/CD integration
- ✅ Distributed architectures

### 4. Provider Agnostic

Not being locked to one LLM provider is crucial:

- ✅ Use the best model for each task
- ✅ Avoid vendor lock-in
- ✅ Cost optimization
- ✅ Future-proof

### 5. Protocol Support

Supporting standard protocols (ACP, MCP) enables:

- ✅ Editor integration
- ✅ Tool extensibility
- ✅ Ecosystem growth
- ✅ Community contributions

### 6. Instance Isolation

Per-directory context isolation is clever:

- ✅ Multiple projects simultaneously
- ✅ Clean separation
- ✅ No cross-contamination
- ✅ Better resource management

---

## Comparison with Other Approaches

### OpenCode vs. WebSocket-Based Systems

| Aspect | OpenCode (REST + SSE) | WebSocket-Based |
|--------|----------------------|-----------------|
| **Testing** | `curl` commands | Special client needed |
| **Debugging** | Browser DevTools | Special debugging tools |
| **Load Balancing** | Standard HTTP LB | Special WebSocket LB config |
| **Caching** | Standard HTTP caching | N/A |
| **Firewalls** | Always works | Often blocked |
| **Simplicity** | Very simple | More complex |
| **Bidirectional** | No (SSE is unidirectional) | Yes |

**Verdict:** For this use case (server → client events), SSE is perfect. No need for client → server streaming.

### OpenCode vs. gRPC Systems

| Aspect | OpenCode (REST) | gRPC |
|--------|----------------|------|
| **Browser Support** | Native | Requires grpc-web |
| **Curl Support** | Yes | No |
| **Human Readable** | JSON | Binary (Protobuf) |
| **Debugging** | Easy | Harder |
| **Performance** | Good enough | Slightly faster |
| **Adoption** | Lower friction | Higher friction |

**Verdict:** The performance gains of gRPC don't justify the added complexity for this use case.

---

## Example: Building a Custom Client

Here's how you'd build a minimal client in Python (without using the SDK):

```python
import requests
import json

class OpenCodeClient:
    def __init__(self, base_url="http://localhost:4096", directory=None):
        self.base_url = base_url
        self.headers = {}
        if directory:
            self.headers["x-opencode-directory"] = directory

    def create_session(self):
        """Create a new session"""
        response = requests.post(
            f"{self.base_url}/session",
            headers=self.headers
        )
        return response.json()

    def send_prompt(self, session_id, prompt):
        """Send a prompt to a session"""
        response = requests.post(
            f"{self.base_url}/session/{session_id}/message",
            headers={**self.headers, "Content-Type": "application/json"},
            json={"prompt": prompt}
        )
        return response.json()

    def stream_events(self, session_id):
        """Stream events from a session"""
        response = requests.get(
            f"{self.base_url}/session/{session_id}/event",
            headers=self.headers,
            stream=True
        )

        for line in response.iter_lines():
            if line.startswith(b"data: "):
                data = json.loads(line[6:])
                yield data

# Usage
client = OpenCodeClient(directory="/path/to/project")

# Create session
session = client.create_session()
print(f"Created session: {session['id']}")

# Send prompt
client.send_prompt(session['id'], "Hello, analyze this codebase")

# Stream events
for event in client.stream_events(session['id']):
    if event['type'] == 'message.part.updated':
        print(event['properties']['text'], end='', flush=True)
    elif event['type'] == 'session.idle':
        print("\nSession complete!")
        break
```

**That's it!** ~50 lines of code for a functional client. No complex SDKs, no protocol buffers, no WebSocket handshakes.

---

## Conclusion

OpenCode's architecture is a masterclass in **developer-first design**. By choosing:

1. **Simple REST over complex protocols**
2. **OpenAPI-first SDK generation**
3. **Client-server separation**
4. **Provider agnosticism**
5. **Standard protocol support**

The team has created a system that is:

- **Easy to understand** - curl commands work
- **Easy to extend** - standard HTTP, MCP, ACP
- **Easy to deploy** - runs anywhere
- **Easy to integrate** - multiple SDKs, protocols
- **Future-proof** - not locked to any vendor

The philosophy of "obsessively reducing friction" is evident in every design decision. This is what makes OpenCode not just usable, but **delightful** to build on.

### For SDK Users

If you're considering using the OpenCode SDK:

1. **Start simple** - Try curl commands first to understand the API
2. **Use the SDK** - Then graduate to the SDK for your language
3. **Experiment** - The headless server mode makes it easy to experiment
4. **Extend** - Use MCP to add custom tools
5. **Contribute** - The architecture is easy to understand and extend

### For Architects

If you're building a similar system:

1. **Question complexity** - Do you really need WebSockets? gRPC?
2. **Developer experience** - Can users test it with curl?
3. **Discoverability** - Is the API self-documenting?
4. **Separation** - Can the server run without the UI?
5. **Standards** - Are you using standard protocols where possible?

OpenCode proves that **simplicity at scale is possible** and that **reducing friction drives adoption**.

---

## References

- **GitHub:** https://github.com/sst/opencode
- **Documentation:** https://opencode.ai/docs
- **Discord:** https://opencode.ai/discord
- **Server Implementation:** `packages/opencode/src/server/server.ts`
- **SDK Implementation:** `packages/sdk/js/src/`
- **OpenAPI Spec:** `GET http://localhost:4096/doc`

---

## Appendix: Complete API Reference

### Session Management

```bash
# Create session
curl -X POST http://localhost:4096/session

# List sessions
curl http://localhost:4096/session

# Get session
curl http://localhost:4096/session/{id}

# Update session
curl -X PATCH http://localhost:4096/session/{id} \
  -H "Content-Type: application/json" \
  -d '{"title": "New Title"}'

# Delete session
curl -X DELETE http://localhost:4096/session/{id}

# Fork session
curl -X POST http://localhost:4096/session/{id}/fork \
  -H "Content-Type: application/json" \
  -d '{"messageID": "msg123"}'

# Abort session
curl -X POST http://localhost:4096/session/{id}/abort
```

### Messaging

```bash
# Send prompt
curl -X POST http://localhost:4096/session/{id}/message \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Hello, world!"}'

# Send command
curl -X POST http://localhost:4096/session/{id}/command \
  -H "Content-Type: application/json" \
  -d '{"command": "/help"}'

# List messages
curl http://localhost:4096/session/{id}/message

# Get specific message
curl http://localhost:4096/session/{id}/message/{messageID}
```

### Events

```bash
# Stream session events
curl http://localhost:4096/session/{id}/event

# Stream all events
curl http://localhost:4096/event

# Stream global events
curl http://localhost:4096/global/event
```

### Configuration

```bash
# Get config
curl http://localhost:4096/config

# Update config
curl -X PATCH http://localhost:4096/config \
  -H "Content-Type: application/json" \
  -d '{"theme": "dark"}'

# List providers
curl http://localhost:4096/config/providers
```

### File Operations

```bash
# List files
curl "http://localhost:4096/file?path=/src"

# Read file
curl "http://localhost:4096/file/content?path=/src/index.ts"

# File status (git)
curl http://localhost:4096/file/status
```

### Search

```bash
# Search text
curl "http://localhost:4096/find?pattern=function"

# Find files
curl "http://localhost:4096/find/file?query=index"

# Find symbols
curl "http://localhost:4096/find/symbol?query=MyClass"
```

### Tools & Extensions

```bash
# List tool IDs
curl http://localhost:4096/experimental/tool/ids

# List tools with schemas
curl "http://localhost:4096/experimental/tool?provider=anthropic&model=claude-3-5-sonnet-20241022"

# LSP status
curl http://localhost:4096/lsp

# MCP status
curl http://localhost:4096/mcp

# Add MCP server
curl -X POST http://localhost:4096/mcp \
  -H "Content-Type: application/json" \
  -d '{"name": "my-mcp", "config": {"command": "node", "args": ["server.js"]}}'
```

### Documentation

```bash
# Get OpenAPI spec
curl http://localhost:4096/doc

# Save to file
curl http://localhost:4096/doc > openapi.json

# View in Swagger UI
open https://editor.swagger.io/?url=http://localhost:4096/doc
```

---

**Last Updated:** 2025-11-19
**OpenCode Version:** 1.0.78
