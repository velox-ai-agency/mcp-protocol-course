# Module 4: Advanced MCP Patterns 🚀

> **Take your MCP skills to production level.**
>
> Multi-server orchestration, subscriptions, security hardening, and real-world patterns.

---

## 📚 Module Overview

| Item | Details |
|------|---------|
| **Duration** | ~60 minutes |
| **Level** | Advanced — Completion of Modules 1-3 required |
| **Prerequisites** | All previous modules, production experience recommended |
| **Outcome** | Design and deploy production-ready MCP servers |

---

## 🎯 Learning Objectives

By the end of this module, you will:

1. ✅ **Orchestrate** multiple MCP servers
2. ✅ **Implement** resource subscriptions and real-time updates
3. ✅ **Secure** MCP servers for production
4. ✅ **Optimize** performance and scalability
5. ✅ **Deploy** servers to cloud environments
6. ✅ **Integrate** with popular AI frameworks

---

## 📖 Lesson Structure

### Lesson 4.1: Multi-Server Orchestration
**Duration:** 15 minutes

#### The Multi-Server Pattern

Real applications often need **multiple MCP servers**:

```
                    ┌──────────────┐
                    │    Host      │
                    │   (Claude)   │
                    └──────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  PostgreSQL  │ │   Filesystem │ │    GitHub    │
    │    Server    │ │    Server    │ │    Server    │
    └──────────────┘ └──────────────┘ └──────────────┘
```

**Use Cases:**
- 🔹 Query database → Fetch related files → Create GitHub issues
- 🔹 Search web → Analyze results → Save to database
- 🔹 Read documentation → Generate code → Commit to repo

#### Server Dependencies

Some servers may depend on others:

```
Workflow: Analyze sales data

1. PostgreSQL Server → Query sales data
2. Pandas Server → Process/analyze data
3. Visualization Server → Create charts
4. Email Server → Send report to team
```

#### Best Practices

| Practice | Why |
|----------|-----|
| **Declarative config** | JSON configs easy to version and share |
| **Environment variables** | Secrets outside of config files |
| **Health checks** | Monitor server availability |
| **Time isolation** | Kill stuck servers |
| **Circuit breakers** | Fail gracefully when servers error |

**Sample Configuration:**

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "${DATABASE_URL}" },
      "timeout": 30000
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "timeout": 10000
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" },
      "timeout": 30000
    }
  }
}
```

---

### Lesson 4.2: Resource Subscriptions
**Duration:** 15 minutes

#### What Are Subscriptions?

**Subscriptions** allow the server to **push updates** to the host:

```
Traditional (Polling):
┌──────┐  "What's the weather?"  ┌────────┐
│ Host │ ───────────────────────→│ Server │
└──────┘                         └────────┘
                                     ↓
┌──────┐  "Still 72°F"          ┌────────┐
│ Host │ ←───────────────────────│ Server │
└──────┘   (asked again)        └────────┘

Subscription (Push):
┌──────┐  "Subscribe to weather"  ┌────────┐
│ Host │  ───────────────────────→│ Server │
└──────┘                         └────────┘
                                      ↓
┌──────┐  "🌡️ Temperature: 72°F"   ┌────────┐
│ Host │ ←─────────────────────────│ Server │
└──────┘   (pushed automatically) └────────┘
                                      ↓
┌──────┐  "🌡️ Temperature: 73°F"   ┌────────┐
│ Host │ ←─────────────────────────│ Server │
└──────┘   (pushed automatically) └────────┘
```

#### Implementing Subscriptions

##### Server-Side (TypeScript)

```typescript
import {
  SubscribeRequestSchema,
  UnsubscribeRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// Track active subscriptions
const subscriptions = new Map<string, Set<string>>(); // uri → connectionIds

// Handle subscription requests
server.setRequestHandler(SubscribeRequestSchema, async (request) => {
  const { uri } = request.params;
  const connectionId = getConnectionId(); // implementation-specific
  
  if (!subscriptions.has(uri)) {
    subscriptions.set(uri, new Set());
    startWatching(uri); // Start file watcher, DB listener, etc.
  }
  
  subscriptions.get(uri)!.add(connectionId);
  return {}; // Success
});

// Handle unsubscription
server.setRequestHandler(UnsubscribeRequestSchema, async (request) => {
  const { uri } = request.params;
  const connectionId = getConnectionId();
  
  subscriptions.get(uri)?.delete(connectionId);
  if (subscriptions.get(uri)?.size === 0) {
    stopWatching(uri);
    subscriptions.delete(uri);
  }
  return {};
});

// Push updates when resource changes
async function notifySubscribers(uri: string, newContent: string) {
  const connectionIds = subscriptions.get(uri);
  if (!connectionIds) return;
  
  for (const connectionId of connectionIds) {
    await sendNotification(connectionId, {
      method: "notifications/resources/updated",
      params: { uri },
    });
  }
}
```

##### Use Cases

| Scenario | Implementation |
|----------|----------------|
| **File watching** | Watch filesystem, notify on changes |
| **Database changes** | Listen to DB triggers, push updates |
| **API monitoring** | Poll external API, notify on changes |
| **Real-time data** | Stock prices, weather, sports scores |
| **Build status** | CI/CD pipeline updates |

---

### Lesson 4.3: Production Security
**Duration:** 20 minutes

#### Security Checklist

##### 1. Input Validation

```typescript
// ❌ Bad: Trust user input
const query = args.sql;
db.execute(query); // SQL injection!

// ✅ Good: Validate and sanitize
if (!isValidSQL(query)) {
  throw new Error("Invalid SQL query");
}
if (isDestructiveOperation(query)) {
  throw new Error("Destructive operations not allowed");
}
db.execute(query);
```

##### 2. Path Traversal Prevention

```typescript
// ❌ Bad: Direct path usage
const fullPath = path.join(baseDir, userPath);
fs.readFile(fullPath);

// ✅ Good: Resolve and validate
const resolved = path.resolve(baseDir, userPath);
if (!resolved.startsWith(baseDir)) {
  throw new Error("Access denied: Path outside allowed directory");
}
fs.readFile(resolved);
```

##### 3. Secrets Management

**Environment Variables:**
```bash
# ❌ Hardcoded
const apiKey = "sk-1234567890abcdef";

# ✅ From environment
const apiKey = process.env.API_KEY;
if (!apiKey) throw new Error("API_KEY not set");
```

**Secret Managers:**
```typescript
// Using AWS Secrets Manager
import { SecretsManager } from "@aws-sdk/client-secrets-manager";

const secrets = new SecretsManager();
const secret = await secrets.getSecretValue({ SecretId: "mcp/api-key" });
const apiKey = secret.SecretString;
```

##### 4. Rate Limiting

```typescript
import { RateLimiter } from "limiter";

const limiter = new RateLimiter({
  tokensPerInterval: 10,
  interval: "minute",
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (!await limiter.tryRemoveTokens(1)) {
    throw new Error("Rate limit exceeded. Try again later.");
  }
  // ... process request
});
```

##### 5. Request Timeout

```typescript
const TIMEOUT = 30000; // 30 seconds

async function withTimeout<T>(
  promise: Promise<T>,
  timeout: number
): Promise<T> {
  return Promise.race([
    promise,
    new Promise<never>((_, reject) => 
      setTimeout(() => reject(new Error("Request timeout")), timeout)
    )
  ]);
}

// Usage
const result = await withTimeout(
  expensiveOperation(),
  TIMEOUT
);
```

##### 6. Sandboxing

**Docker Container:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
USER node  # Run as non-root
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Run with limits:**
```bash
docker run --rm \
  --memory="512m" \
  --cpus="1.0" \
  --read-only \
  --tmpfs /tmp \
  -e API_KEY="${API_KEY}" \
  my-mcp-server
```

---

### Lesson 4.4: Performance & Scalability
**Duration:** 10 minutes

#### Performance Tips

| Technique | Benefit |
|-----------|---------|
| **Connection pooling** | Reuse DB connections |
| **Caching results** | Avoid repeated expensive operations |
| **Streaming responses** | Handle large data without buffering |
| **Async loading** | Don't block on slow resources |
| **Lazy initialization** | Only connect when needed |

**Caching Example:**

```typescript
import NodeCache from "node-cache";

const cache = new NodeCache({ stdTTL: 300 }); // 5 min TTL

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const cacheKey = JSON.stringify(request.params);
  
  // Check cache first
  const cached = cache.get(cacheKey);
  if (cached) {
    return { content: [{ type: "text", text: cached }] };
  }
  
  // Execute and cache
  const result = await expensiveOperation(request.params);
  cache.set(cacheKey, result);
  
  return { content: [{ type: "text", text: result }] };
});
```

---

## ✏️ Exercises

### Exercise 4.1: Design a Multi-Server System

Design an architecture for a **Sales Analytics Dashboard** using MCP:

**Requirements:**
1. Fetch sales data from PostgreSQL
2. Query web for competitor pricing
3. Generate charts/visualizations
4. Save reports to S3
5. Email reports to stakeholders
6. Schedule daily reports

**Questions:**
- How many MCP servers do you need?
- What are the dependencies between them?
- What capabilities does each server expose?

<details>
<summary>✅ Solution</summary>

**Servers:**
1. **PostgreSQL Server** — Database queries
2. **Web Search Server** (Tavily/Brave) — Competitor research
3. **Chart Server** — Data visualization
4. **S3 Server** — File storage
5. **Email Server** — Notifications
6. **Scheduler Server** — Cron-like scheduling

**Workflow:**
```
Scheduler (6) triggers daily
  ↓
PostgreSQL (1) → Sales data
  ↓
Web Search (2) → Competitor prices
  ↓
Chart Server (3) → Generate dashboard
  ↓
S3 Server (4) → Save report
  ↓
Email Server (5) → Notify stakeholders
```

</details>

---

### Exercise 4.2: Secure File Server

Review this code and identify security issues:

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "read_file") {
    const path = request.params.arguments.path;
    const content = await fs.readFile(path);
    return { content: [{ type: "text", text: content }] };
  }
});
```

<details>
<summary>✅ Issues</summary>

1. **No path validation** — Can read any file on filesystem
2. **No error handling** — Sensitive info might leak in errors
3. **No size limits** — Could crash reading huge files
4. **No rate limiting** — Could be used for DoS

**Fixed version:**
```typescript
const ALLOWED_DIR = "/safe/directory";
const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "read_file") {
    const userPath = request.params.arguments.path;
    
    // Validate path
    const fullPath = path.resolve(ALLOWED_DIR, userPath);
    if (!fullPath.startsWith(ALLOWED_DIR)) {
      throw new Error("Access denied");
    }
    
    // Check file size
    const stats = await fs.stat(fullPath);
    if (stats.size > MAX_FILE_SIZE) {
      throw new Error("File too large");
    }
    
    // Read with error handling
    try {
      const content = await fs.readFile(fullPath, "utf-8");
      return { content: [{ type: "text", text: content }] };
    } catch (err) {
      throw new Error("Failed to read file");
    }
  }
});
```

</details>

---

### Exercise 4.3: Create a Subscription

Implement a file watcher that:
1. Allows hosts to subscribe to file changes
2. Watches a directory for changes
3. Notifies subscribers when files change
4. Properly handles unsubscribes

**Hint:** Use `fs.watch()` or `chokidar` package.

---

## 📋 Summary

### Advanced Patterns

| Pattern | Use Case |
|---------|----------|
| **Multi-server** | Complex workflows across tools |
| **Subscriptions** | Real-time updates, monitoring |
| **Caching** | Performance optimization |
| **Rate limiting** | Prevent abuse |
| **Sandboxing** | Security isolation |
| **Streaming** | Large data handling |

### Production Checklist

- ✅ Input validation and sanitization
- ✅ Path traversal prevention
- ✅ Secrets in environment variables
- ✅ Rate limiting implemented
- ✅ Request timeouts configured
- ✅ Sandboxed deployment
- ✅ Health checks and monitoring
- ✅ Logging and audit trails

### Next Steps

🎓 You now have the skills to:
1. Build custom MCP servers
2. Design multi-server architectures
3. Deploy to production securely
4. Optimize for performance

➡️ **Connect with the community:**
- 🔗 [MCP Discord](https://discord.gg/mcp)
- 🔗 [GitHub Discussions](https://github.com/modelcontextprotocol/specification/discussions)
- 🔗 [Share your servers](https://github.com/modelcontextprotocol/servers#contributing)

---

## 📚 Additional Resources

### Official
- 🔗 [MCP Specification](https://spec.modelcontextprotocol.io)
- 🔗 [Official Servers](https://github.com/modelcontextprotocol/servers)
- 🔗 [Release Notes](https://github.com/modelcontextprotocol/specification/releases)

### Community
- 🔗 [Awesome MCP](https://github.com/punkpeye/awesome-mcp-servers)
- 🔗 [FastMCP (Python)](https://github.com/jlowin/fastmcp)
- 🔗 [MCP Proxy](https://github.com/sparfenyuk/mcp-proxy)

### Deployment
- 🔗 [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- 🔗 [AWS Lambda MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/aws-lambda)

---

*Module 4 of Mastering MCP*  
*© Velox AI Agency*

**Congratulations! You've completed the MCP Protocol Course! 🎉**