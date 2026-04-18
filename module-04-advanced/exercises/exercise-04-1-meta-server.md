# Exercise 4.1: Build a Meta-Server

## 🎯 Objective
Create a meta-server that aggregates tools from multiple MCP servers.

---

## 📋 Instructions

Build a server that:
1. Connects to at least 2 child servers
2. Exposes their tools with namespacing
3. Routes calls to the correct server

---

## 🎨 Requirements

### Server 1: Calculator Server
Create a simple calculator:

```typescript
// calculator-server.ts
// Tools: add, subtract, multiply, divide
```

### Server 2: String Server
Create string utilities:

```typescript
// string-server.ts
// Tools: uppercase, lowercase, reverse, count_words
```

### Meta-Server
Combine them:

```typescript
// meta-server.ts
// Exports tools:
// - calculator_add
// - calculator_subtract
// - string_uppercase
// - string_lowercase
// etc.
```

---

## 💻 Implementation Template

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// Define child servers
const CHILD_SERVERS = [
  { name: "calculator", command: "node", args: ["./calculator-server.js"] },
  { name: "string", command: "node", args: ["./string-server.js"] },
];

async function createMetaServer() {
  // TODO: Connect to all child servers
  // TODO: Create aggregating server
  // TODO: Implement listTools with prefixing
  // TODO: Implement callTool with routing
}
```

---

## ✅ Requirements Checklist

- [ ] Connects to ≥2 child servers
- [ ] Prefixes tools: `servername_toolname`
- [ ] Routes calls correctly
- [ ] Handles child server failures gracefully
- [ ] Includes error messages that indicate which server failed

---

## 🧪 Testing

```json
{
  "method": "tools/list"
}
// Expected: 8+ tools with prefixes

{
  "method": "tools/call",
  "params": {
    "name": "calculator_add",
    "arguments": { "a": 5, "b": 3 }
  }
}
// Expected: 8

{
  "method": "tools/call",
  "params": {
    "name": "string_uppercase",
    "arguments": { "text": "hello" }
  }
}
// Expected: "HELLO"
```

---

## 🏆 Bonus Challenges

- [ ] Add 3+ servers
- [ ] Add resource aggregation
- [ ] Add prompt aggregation
- [ ] Implement automatic retry for failed servers
- [ ] Add server health checks

---

## 💡 Tips

- Start child servers before meta-server
- Use `Promise.all()` for parallel connections
- Store client references in a Map by server name
- Parse tool name to extract server and tool

---

## 🔗 Next

Continue to [Lesson 4.2: Subscriptions & Real-Time Updates](../lesson-02-subscriptions.md)
