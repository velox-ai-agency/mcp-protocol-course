# Lesson 3.5: Resources Server — Exposing Data Sources

## 🎯 Learning Objectives
By the end of this lesson, you will:
- Understand MCP Resources and how they differ from tools
- Build a resource server that exposes files and data
- Implement resource subscriptions for real-time updates
- Learn URI patterns and MIME type handling

---

## 📋 What Are MCP Resources?

**Resources** provide read-only access to data sources:
- Files (text, JSON, images, etc.)
- Database query results
- API responses
- Configuration files

Unlike **Tools** (actions), Resources are **data** that the AI can read.

### Tool vs Resource

```typescript
// Tool: DO something (action)
client.callTool("read_file", { path: "config.json" })

// Resource: GET something (data)
client.readResource("file:///config.json")
```

---

## 🔗 URI Schemes

Resources use URIs to identify data:

```
file:///home/user/project/readme.md
config:///app/settings.json
db:///queries/active-users
api:///weather/current?city=london
```

**Features:**
- Standardized identification
- MIME type support
- Subscriptions for updates

---

## 💻 Building a Resources Server

### Step 1: Create the Server

Create `resources-server.ts`:

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ReadResourceRequestSchema,
  ListResourcesRequestSchema,
  SubscribeRequestSchema,
  UnsubscribeRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import * as fs from "fs/promises";
import * as path from "path";

const BASE_DIR = "/home/user/project";

// Resource content cache for subscriptions
const resourceCache = new Map<string, string>();
const subscribers = new Set<string>();

const server = new Server(
  { name: "resources-server", version: "1.0.0" },
  { 
    capabilities: { 
      resources: {
        subscribe: true,  // Enable subscriptions
        listChanged: true // Notify on resource changes
      }
    } 
  }
);

// List available resources (static)
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: "file:///readme.md",
        name: "Project README",
        mimeType: "text/markdown",
        description: "Main documentation file"
      },
      {
        uri: "file:///package.json",
        name: "Package Configuration",
        mimeType: "application/json",
        description: "NPM package configuration"
      },
      {
        uri: "config:///settings.json",
        name: "App Settings",
        mimeType: "application/json",
        description: "Application configuration"
      },
      {
        uri: "logs:///latest",
        name: "Latest Logs",
        mimeType: "text/plain",
        description: "Most recent application logs"
      },
      {
        uri: "data:///stats",
        name: "Project Statistics",
        mimeType: "application/json",
        description: "Code statistics and metrics"
      }
    ]
  };
});

// Read resource content
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;
  const url = new URL(uri);
  
  let content: string;
  let mimeType: string;
  
  try {
    switch (url.protocol) {
      case "file:": {
        // File resources
        const filePath = path.join(BASE_DIR, url.pathname);
        content = await fs.readFile(filePath, "utf-8");
        
        // Determine MIME type from extension
        const ext = path.extname(filePath).toLowerCase();
        const mimeTypes: Record<string, string> = {
          ".md": "text/markdown",
          ".json": "application/json",
          ".txt": "text/plain",
          ".ts": "text/plain",
          ".js": "text/plain",
          ".py": "text/plain"
        };
        mimeType = mimeTypes[ext] || "text/plain";
        break;
      }
      
      case "config:": {
        // Simulated config resource
        if (url.pathname === "/settings.json") {
          content = JSON.stringify({
            theme: "dark",
            autoSave: true,
            version: "1.0.0",
            features: {
              beta: false,
              notifications: true
            }
          }, null, 2);
          mimeType = "application/json";
        } else {
          throw new Error(`Unknown config: ${url.pathname}`);
        }
        break;
      }
      
      case "logs:": {
        // Simulated log resource
        if (url.pathname === "/latest") {
          content = `[2024-01-15 10:30:00] INFO: Application started
[2024-01-15 10:30:01] INFO: Connected to database
[2024-01-15 10:30:02] WARN: Slow query detected (245ms)
[2024-01-15 10:30:05] INFO: Cache warmed up`;
          mimeType = "text/plain";
        } else {
          throw new Error(`Unknown log path: ${url.pathname}`);
        }
        break;
      }
      
      case "data:": {
        // Simulated data resource
        if (url.pathname === "/stats") {
          content = JSON.stringify({
            files: 42,
            linesOfCode: 3847,
            functions: 156,
            tests: 89,
            coverage: "87%",
            lastCommit: "2024-01-14T15:30:00Z"
          }, null, 2);
          mimeType = "application/json";
        } else {
          throw new Error(`Unknown data path: ${url.pathname}`);
        }
        break;
      }
      
      default:
        throw new Error(`Unsupported protocol: ${url.protocol}`);
    }
    
    // Cache for subscription comparisons
    resourceCache.set(uri, content);
    
    return {
      contents: [{
        uri,
        mimeType,
        text: content
      }]
    };
    
  } catch (error) {
    throw new Error(`Failed to read resource: ${(error as Error).message}`);
  }
});

// Handle subscriptions (simulated)
server.setRequestHandler(SubscribeRequestSchema, async (request) => {
  const { uri } = request.params;
  subscribers.add(uri);
  console.error(`Subscribed to: ${uri}`);
  
  // In a real implementation, you'd set up file watchers or polling
  return {};
});

server.setRequestHandler(UnsubscribeRequestSchema, async (request) => {
  const { uri } = request.params;
  subscribers.delete(uri);
  console.error(`Unsubscribed from: ${uri}`);
  return {};
});

// Start server
const transport = new StdioServerTransport();
server.connect(transport);

console.error("Resources MCP Server running...");
console.error("Serving resources from:", BASE_DIR);
```

---

## 🧪 Testing Resources

### List Resources
```json
{
  "method": "resources/list"
}
```

### Read a Resource
```json
{
  "method": "resources/read",
  "params": {
    "uri": "config:///settings.json"
  }
}
```

### Subscribe to Updates
```json
{
  "method": "resources/subscribe",
  "params": {
    "uri": "file:///readme.md"
  }
}
```

---

## 🔧 Configuration

```json
{
  "mcpServers": {
    "resources": {
      "command": "node",
      "args": ["/path/to/resources-server.js"],
      "env": {
        "BASE_DIR": "/home/user/project"
      }
    }
  }
}
```

---

## 💡 Real-World Example: Database Resources

For a real database integration:

```typescript
case "db:": {
  // Connect to your database
  const query = url.searchParams.get("q");
  const result = await db.query(query);
  content = JSON.stringify(result, null, 2);
  mimeType = "application/json";
  break;
}
```

URI example: `db:///query?q=SELECT * FROM users WHERE active=true`

---

## 📊 Resource vs Tool Decision Matrix

| Use Case | Resource | Tool | Reason |
|----------|----------|------|--------|
| Read a file | ✅ | ❌ | Read-only data |
| Write a file | ❌ | ✅ | Changes state |
| Query database | ✅ | ❌ | Read-only |
| Modify database | ❌ | ✅ | Changes state |
| Get configuration | ✅ | ❌ | Static data |
| Update configuration | ❌ | ✅ | Changes state |
| Read logs | ✅ | ❌ | Read-only |
| Trigger deployment | ❌ | ✅ | Action |

---

## 📚 Summary

| Aspect | Details |
|--------|---------|
| **Protocol** | `resources://`, `file://`, custom schemes |
| **Operations** | list, read, subscribe, unsubscribe |
| **Content** | Binary or text with MIME type |
| **Subscriptions** | Real-time change notifications |
| **Caching** | Client-side caching with ETags |

---

## ✅ Practice Exercise

Complete [Exercise 3.5: Resource Server Challenge](../exercises/exercise-03-5-resource-challenge.md)

Then move on to [Module 4: Advanced Patterns](../../module-04-advanced/)
