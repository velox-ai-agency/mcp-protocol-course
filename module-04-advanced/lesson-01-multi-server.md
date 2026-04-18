# Lesson 4.1: Multi-Server Connections

## 🎯 Learning Objectives
By the end of this lesson, you will:
- Understand how multiple MCP servers coexist
- Learn cross-server integration patterns
- Build a meta-server that aggregates multiple servers
- Handle server conflicts and routing

---

## 📋 Why Multiple Servers?

Real applications use **multiple MCP servers**:

```
┌─────────────────────────────────────────┐
│              Claude Desktop             │
├─────────────────────────────────────────┤
│  filesystem   │  calculator │  github │
│     server     │    server   │  server │
├───────────────┼─────────────┼──────────┤
│  database     │   prompts   │  slack   │
│     server     │    server   │  server │
└───────────────┴─────────────┴──────────┘
```

**Benefits:**
- Separation of concerns
- Different teams own different servers
- Mix languages (Python, TypeScript, etc.)
- Independent versioning

---

## 🔌 How Clients Handle Multiple Servers

### Tool Namespacing

When multiple servers have the same tool name, clients may prefix:

```json
{
  "tools": [
    { "name": "read_file" },         // filesystem
    { "name": "github_read_file" },  // github
    { "name": "s3_read_file" }       // aws-s3
  ]
}
```

Or by server selection:

```json
{
  "mcpServers": {
    "filesystem": { "command": "..." },
    "github": { "command": "..." }
  }
}
```

---

## 💻 Building a Meta-Server

A meta-server aggregates multiple MCP servers:

### Architecture

```
┌────────────────┐
│   MCP Client   │
└───────┬────────┘
        │
┌───────▼────────┐
│  Meta-Server   │ ◄── Aggregates multiple servers
├───────┬────────┤
│       │        │
   ┌────┼────┐
   │    │    │
┌──▼┐ ┌─▼┐ ┌▼─┐
│S1 │ │S2│ │S3│ ◄── Real MCP servers
└───┘ └──┘ └──┘
```

### Implementation

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

// Child server configurations
const CHILD_SERVERS = [
  { name: "calculator", command: "node", args: ["./calculator-server.js"] },
  { name: "filesystem", command: "node", args: ["./filesystem-server.js"] },
];

async function createMetaServer() {
  const clients: Map<string, Client> = new Map();
  
  // Connect to all child servers
  for (const config of CHILD_SERVERS) {
    const transport = new StdioClientTransport({
      command: config.command,
      args: config.args,
    });
    
    const client = new Client(
      { name: "meta-client", version: "1.0.0" },
      { capabilities: { tools: {}, resources: {}, prompts: {} } }
    );
    
    await client.connect(transport);
    clients.set(config.name, client);
    
    console.error(`Connected to ${config.name}`);
  }
  
  // Create aggregating server
  const server = new Server(
    { name: "meta-server", version: "1.0.0" },
    { 
      capabilities: { 
        tools: {}, 
        resources: { subscribe: true },
        prompts: {}
      } 
    }
  );
  
  // Aggregate tools from all clients
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const allTools = [];
    
    for (const [name, client] of clients) {
      const response = await client.listTools();
      const prefixed = response.tools.map(tool => ({
        ...tool,
        name: `${name}_${tool.name}`,  // Add namespace prefix
        description: `[${name}] ${tool.description}`
      }));
      allTools.push(...prefixed);
    }
    
    return { tools: allTools };
  });
  
  // Route tool calls to appropriate client
  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    
    // Parse prefix: "calculator_add" → server: calculator, tool: add
    const [serverName, toolName] = name.split("_", 2);
    const client = clients.get(serverName);
    
    if (!client) {
      throw new Error(`Unknown server: ${serverName}`);
    }
    
    return await client.callTool(toolName, args);
  });
  
  // Similar patterns for resources and prompts...
  
  const transport = new StdioServerTransport();
  await server.connect(transport);
  
  console.error("Meta-server ready");
}

createMetaServer().catch(console.error);
```

---

## ⚠️ Handling Conflicts

### Case 1: Same Tool Name

**Solutions:**
1. **Prefixing** — `calculator_add` vs `math_add`
2. **Priority** — Client picks first match
3. **User selection** — Prompt user to choose

### Case 2: Different Versions

```
calculator v1.0: add(a, b)
calculator v2.0: add(a, b, c?)
```

**Solutions:**
- **Semantic versioning** — Include version in tool name
- **Capabilities** — Check server capabilities first
- **Graceful degradation** — Handle missing arguments

### Case 3: Circular Dependencies

```
Server A → calls tool from → Server B
Server B → needs resource from → Server A
```

**Solutions:**
- **Timeout and retry**
- **Dependency graph** — Load in order
- **Async only** — No synchronous calls

---

## 🧪 Testing Multi-Server Setup

```json
{
  "mcpServers": {
    "meta": {
      "command": "node",
      "args": ["/path/to/meta-server.js"]
    }
  }
}
```

With Inspector:

```bash
# Start individual servers in separate terminals
calculator-server && filesystem-server

# Then test meta-server
npx @modelcontextprotocol/inspector node meta-server.js
```

---

## 🏢 Real-World Pattern: Team Setup

```
Engineering Team:
├── code-reviewer-server
├── deployment-server
├── test-runner-server

Data Team:
├── sql-query-server
├── data-pipeline-server
├── analytics-server

Operations Team:
├── monitoring-server
├── alerts-server
├── incident-response-server
```

Each team owns their servers, independently versioned.

---

## 📚 Summary

| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| Tool Prefixing | Avoid name conflicts | `${server}_${tool}` |
| Meta-Server | Unify multiple servers | Aggregate in proxy |
| Capability Checking | Version compatibility | Check before calling |
| Async Only | Prevent deadlocks | Never sync between servers |

---

## ✅ Practice Exercise

Complete [Exercise 4.1: Build a Meta-Server](../exercises/exercise-04-1-meta-server.md)
