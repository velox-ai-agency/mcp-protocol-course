# Module 2: MCP Architecture Deep Dive 🏗️

> **Dive deep into how MCP works under the hood.**
>
> This module covers transport layers, the protocol lifecycle, security, and implementation details.

---

## 📚 Module Overview

| Item | Details |
|------|---------|
| **Duration** | ~60 minutes (reading + hands-on) |
| **Level** | Intermediate — Completion of Module 1 recommended |
| **Prerequisites** | Module 1, basic understanding of JSON-RPC |
| **Outcome** | Understand MCP internals deeply enough to debug and optimize |

---

## 🎯 Learning Objectives

By the end of this module, you will:

1. ✅ **Explain** MCP transport mechanisms (stdio, HTTP, SSE)
2. ✅ **Trace** the full lifecycle of an MCP connection
3. ✅ **Understand** JSON-RPC 2.0 message format
4. ✅ **Describe** capability negotiation between Host and Server
5. ✅ **Analyze** the MCP security model
6. ✅ **Debug** MCP communication issues

---

## 📖 Lesson Structure

### Lesson 2.1: Transport Layers
**Duration:** 15 minutes

#### What is a Transport?

The transport layer defines **how** the Host and Server communicate:

```
┌────────────┐     Transport      ┌────────────┐
│   Host     │←══════════════════→│   Server   │
│            │    (stdio/HTTP)   │            │
└────────────┘                    └────────────┘
```

#### Transport Options

##### Option 1: Standard IO (stdio) 🖥️

**How it works:**
- Server runs as a child process of the Host
- Messages sent via stdin/stdout
- Stderr used for logging

**Pros:**
- ✅ Simple — No network required
- ✅ Secure — Process isolation
- ✅ Portable — Works everywhere

**Cons:**
- ❌ One server per process (can't share)
- ❌ Harder to debug
- ❌ Server lifecycle tied to Host

**Use case:** Desktop apps, local integrations

```json
// claude_desktop_config.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/data"]
    }
  }
}
```

**Communication flow:**
```
┌────────────────────────────────────────────────────────┐
│                        Host                            │
│  ┌─────────────────┐                                   │
│  │   MCP Client    │                                   │
│  │                 │◄──stdin───┐                       │
│  │   - stdin write  │          │                       │
│  │   - stdout read  │──stdout──┘                       │
│  └─────────────────┘                                   │
│          │                                            │
│          │ fork()                                      │
└──────────┼─────────────────────────────────────────────┘
           │
┌──────────┴─────────────────────────────────────────────┐
│                   Server Process                       │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │   MCP Server                                     │ │
│  │   - stdin read  ←── receives JSON-RPC           │ │
│  │   - stdout write ──→ sends JSON-RPC             │ │
│  │   - stderr write ──→ logging                     │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

##### Option 2: HTTP with SSE 📡

**How itworks:**
- Server runs as a web service
- Host connects via HTTP
- Server-sent events for stream

**Pros:**
- ✅ Multiple clients can connect
- ✅ Can share servers
- ✅ Easier to monitor/debug
- ✅ Remote servers

**Cons:**
- ❌ Network-dependent
- ❌ Security concerns
- ❌ More complex setup

**Use case:** Cloud services, remote tools

```json
// Configuration
{
  "server": {
    "url": "http://localhost:3000/sse",
    "type": "sse"
  }
}
```

**Communication flow:**
```
┌──────────────┐         HTTP POST        ┌──────────────┐
│              │  ─────────────────────►  │              │
│    Host      │                          │    Server    │
│   (Client)   │ ◀──────────────────────  │   (HTTP)     │
│              │       SSE (Event Stream) │              │
└──────────────┘                          └──────────────┘

Request: POST /mcp { method: "tools/call", ... }
Response: SSE stream with results
```

##### Option 3: WebSocket 💬

**How it works:**
- Bidirectional persistent connection
- Real-time communication

**Pros:**
- ✅ True bidirectional
- ✅ Low latency
- ✅ Real-time updates

**Cons:**
- ❌ More complex
- ❌ WebSocket support required

**Use case:** Real-time applications

---

### Lesson 2.2: JSON-RPC Protocol
**Duration:** 15 minutes

#### What is JSON-RPC?

**JSON-RPC** is a lightweight remote procedure call (RPC) protocol.

**Key characteristics:**
- Transport-agnostic
- Simple request-response format
- Supports notifications (no response)
- Supports batch requests

**Why MCP uses JSON-RPC:**
- 🔹 Standardized — Industry proven
- 🔹 Language-agnostic — Works in any language
- 🔹 Simple — Easy to implement
- 🔹 Flexible — Supports streaming

#### Message Types

##### 1️⃣ Request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": {
      "query": "SELECT * FROM users"
    }
  }
}
```

**Fields:**
- `jsonrpc`: Must be "2.0"
- `id`: Unique identifier for the request
- `method`: The method to call
- `params`: Method arguments

##### 2️⃣ Response (Success)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "[{\"id\": 1, \"name\": \"Alice\"}]"
      }
    ],
    "isError": false
  }
}
```

**Fields:**
- `id`: Matches the request id
- `result`: The return value

##### 3️⃣ Response (Error)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": "Missing required parameter: query"
  }
}
```

**Common error codes:**

| Code | Meaning | Description |
|------|---------|-------------|
| -32700 | Parse error | Invalid JSON |
| -32600 | Invalid Request | JSON not a valid request object |
| -32601 | Method not found | Method doesn't exist |
| -32602 | Invalid params | Invalid method params |
| -32603 | Internal error | Server error |
| -32000 to -32099 | Server errors | Reserved for servers |

##### 4️⃣ Notification

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

**Notice:** No `id` field — no response expected

#### MCP-Specific Methods

##### Initialization

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": { "subscribe": true }
    },
    "clientInfo": {
      "name": "claude-ai",
      "version": "1.0.0"
    }
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {},
      "resources": {},
      "prompts": {}
    },
    "serverInfo": {
      "name": "postgres-server",
      "version": "1.0.0"
    }
  }
}
```

##### Tool Operations

```json
// List available tools
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}

// Call a tool
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": {
      "query": "SELECT COUNT(*) FROM users"
    }
  }
}
```

##### Resource Operations

```json
// List available resources
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/list",
  "params": {}
}

// Read a resource
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/read",
  "params": {
    "uri": "file:///home/user/README.md"
  }
}
```

---

### Lesson 2.3: MCP Lifecycle
**Duration:** 15 minutes

#### Connection Establishment

##### Step 1: Initialization

```
┌────────┐                              ┌────────┐
│  Host  │                              │ Server │
└────┬───┘                              └───┬────┘
     │                                      │
     │ ════════════════════════════════════│
     │         CONNECTION OPENED            │
     │ ════════════════════════════════════│
     │                                      │
     │  {                                   │
     │    "method": "initialize",           │
     │    "params": {                       │
     │      "protocolVersion": "2024-11-05",│
     │      "capabilities": {...},          │
     │      "clientInfo": {...}            │
     │    }                                 │
     │  }                                   │
     │ ──────────────────────────────────► │
     │                                      │
     │  {                                   │
     │    "result": {                       │
     │      "protocolVersion": "...",        │
     │      "capabilities": {...},          │
     │      "serverInfo": {...}            │
     │    }                                 │
     │  }                                   │
     │ ◄────────────────────────────────── │
     │                                      │
     │                                      │
     │ ════════════════════════════════════│
     │          CAPABILITIES EXCHANGED       │
     │ ════════════════════════════════════│
     │                                      │
```

##### Step 2: Capability Negotiation

**Flow:**
1. **Host** declares what it supports (e.g., "I can use tools")
2. **Server** declares what it offers (e.g., "I have 5 tools, 3 resources")
3. Both agree on the intersection

**Example:**

```
Host says: "I support tool listing and tool calls"
     ↓
Server says: "I offer query_database and send_email tools"
     ↓
Host knows: "I can use query_database and send_email"
```

##### Step 3: Ready State

```json
// Host sends
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}

// Connection is now ready for use!
```

#### Normal Operation

```
┌────────┐                              ┌────────┐
│  Host  │                              │ Server │
└────┬───┘                              └───┬────┘
     │                                      │
     │  ╔════════════════════════════════╗  │
     │  ║     NORMAL OPERATION CYCLE     ║  │
     │  ╚════════════════════════════════╝  │
     │                                      │
     │  { "method": "tools/call", ... }    │
     │ ──────────────────────────────────► │
     │     [Server executes query]         │
     │  { "result": {...} }                │
     │ ◄────────────────────────────────── │
     │                                      │
     │  { "method": "resources/read", ...} │
     │ ──────────────────────────────────► │
     │     [Server reads file]             │
     │  { "result": {...} }                │
     │ ◄────────────────────────────────── │
     │                                      │
```

#### Teardown

```
┌────────┐                              ┌────────┐
│  Host  │                              │ Server │
└────┬───┘                              └───┬────┘
     │                                      │
     │  { "method": "notifications/cancel" } │
     │ ──────────────────────────────────► │
     │                                      │
     │         [OR Host closes stdin]       │
     │                                      │
     │ ════════════════════════════════════│
     │       SERVER PROCESS TERMINATES      │
     │ ════════════════════════════════════│
```

---

### Lesson 2.4: Security Model
**Duration:** 15 minutes

#### Security Principles

MCP follows the principle of **least privilege**:

1. **Explicit consent** — User approves which servers to connect
2. **Capability-based** — Only requested capabilities are available
3. **Isolated execution** — Servers run in separate processes
4. **No ambient authority** — Auth tokens not shared automatically

#### Security Layers

```
┌─────────────────────────────────────────────────────┐
│ Layer 4: User Consent                                 │
│         ↓ User grants permission to connect server    │
├─────────────────────────────────────────────────────┤
│ Layer 3: Host Policy                                  │
│         ↓ Host decides which capabilities to allow  │
├─────────────────────────────────────────────────────┤
│ Layer 2: Capability Negotiation                       │
│         ↓ Server exposes only what it supports        │
├─────────────────────────────────────────────────────┤
│ Layer 1: Transport Security                           │
│         ↓ stdin/stdout isolation OR TLS for HTTP    │
└─────────────────────────────────────────────────────┘
```

#### Best Practices

**For Host Developers:**
- ✅ Sandbox server processes
- ✅ Validate JSON-RPC messages
- ✅ Set resource limits (CPU, memory, time)
- ✅ Log all tool invocations
- ✅ Allow users to approve sensitive operations

**For Server Developers:**
- ✅ Never hardcode secrets
- ✅ Validate all inputs
- ✅ Rate limit expensive operations
- ✅ Minimize required permissions
- ✅ Clear error messages (but not too verbose)

#### Common Attack Vectors

| Vector | Mitigation |
|--------|------------|
| Malicious server code | Process isolation, limited permissions |
| Data exfiltration | Access controls, audit logging |
| Prompt injection | Input validation, sanitization |
| Resource exhaustion | Timeouts, memory limits |
| Man-in-the-middle | TLS for transports, stdio trust model |

---

## ✏️ Exercises

### Exercise 2.1: Transport Selection

Choose the best transport for each scenario:

1. A developer tool that analyzes local code files
2. A shared company tool accessed by multiple developers
3. A real-time collaboration feature needing instant updates
4. An AI assistant installed on a user's desktop

<details>
<summary>✅ Answer Key</summary>

1. **stdio** — Local access, simple setup, no network needed
2. **HTTP/SSE** — Shared resource, multiple clients
3. **WebSocket** — Real-time, bidirectional communication
4. **stdio** — Desktop app, process isolation, no network dependency

</details>

---

### Exercise 2.2: JSON-RPC Analysis

Analyze this MCP message:

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "tools/call",
  "params": {
    "name": "delete_database",
    "arguments": {
      "database_name": "production"
    }
  }
}
```

**Questions:**
1. Is this a request, response, or notification?
2. What tool is being called?
3. Should this be allowed? Why or why not?

<details>
<summary>✅ Answer Key</summary>

1. **Request** — Has `id`, `method`, and `params`
2. **delete_database** — Very dangerous operation!
3. **Probably not allowed** — Deleting production databases is extremely risky. A good MCP host would either:
   - Reject this tool entirely
   - Require explicit user confirmation
   - Only allow in specific contexts (e.g., with --dev-mode flag)

</details>

---

### Exercise 2.3: Debug the Connection

You're debugging an MCP connection that's not working. The server logs show:

```
Received: {"jsonrpc": "2.0", "method": "initialize", "params": {}}
Error: Invalid request - missing 'id' field
```

**What's wrong and how do you fix it?**

<details>
<summary>✅ Answer Key</summary>

**Problem:** The `initialize` request is missing the `id` field. MCP requires `id` for initialization (it's not a notification).

**Fix:** Add a unique id:

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {}
  }
}
```

</details>

---

### Exercise 2.4: Security Design

You're designing an MCP server for a cryptocurrency wallet. List 5 security measures you would implement.

**Example Answer:**

1. **Require explicit transaction confirmation** — Never auto-sign transactions
2. **Rate limiting** — Prevent rapid-fire transaction attempts
3. **Amount caps** — Limit maximum transaction values without MFA
4. **Audit logging** — Log every transaction attempt
5. **Network validation** — Verify network (mainnet vs testnet) before signing
6. **Read-only by default** — Require explicit permission for write operations
7. **Timeout on approvals** — Transaction requests expire after 5 minutes

---

### Exercise 2.5: Protocol Flow Diagram

Draw or describe the complete flow when:
1. A user opens Claude Desktop
2. Claude connects to a PostgreSQL MCP server
3. User asks "How many users are in the database?"
4. Claude queries the database and responds

**Steps to include:**
- Process spawning (stdio)
- Initialization
- Capability exchange
- Tool list
- Tool call
- Response

<details>
<summary>✅ Answer Flow</summary>

```
1. User opens Claude Desktop
2. Claude spawns PostgreSQL server process (stdio)
3. Claude sends initialize request
4. Server responds with capabilities (tools, resources)
5. Claude sends initialized notification
6. Claude requests tools/list
7. Server returns available tools including "query_database"
8. User: "How many users?"
9. Claude decides to use "query_database" tool
10. Claude sends tools/call with query "SELECT COUNT(*) FROM users"
11. Server executes query
12. Server returns result
13. Claude displays "There are 1,234 users in the database"
```

</details>

---

## 📋 Summary

### Key Takeaways

1. ✅ **Three transports:** stdio (local), HTTP/SSE (remote), WebSocket (real-time)
2. ✅ **JSON-RPC 2.0** is the protocol — requests, responses, and notifications
3. ✅ **Lifecycle stages:** Initialization → Capability negotiation → Ready → Operation → Teardown
4. ✅ **Security layers:** User consent → Host policy → Capability negotiation → Transport security
5. ✅ **Error handling** via JSON-RPC error codes and custom error messages

### Next Steps

➡️ **Module 3:** Building Your First MCP Server — hands-on development!

---

## 📚 Additional Resources

- 🔗 [MCP Specification](https://spec.modelcontextprotocol.io)
- 🔗 [JSON-RPC 2.0 Spec](https://www.jsonrpc.org/specification)
- 📝 [MCP Security Best Practices](https://modelcontextprotocol.io/docs/concepts/security)

---

*Module 2 of Mastering MCP Protocol*  
*© Velox AI Agency*