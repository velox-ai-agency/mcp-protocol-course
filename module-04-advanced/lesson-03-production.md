# Lesson 4.3: Production Deployment

## 🎯 Learning Objectives
By the end of this lesson, you will:
- Understand MCP server deployment patterns
- Learn security best practices for production
- Implement monitoring and logging
- Build a production-ready MCP server

---

## 📋 Production Considerations

Running MCP servers in production requires:

| Concern | Development | Production |
|--------|-------------|------------|
| Secrets | Hardcoded | Environment variables / secrets manager |
| Logging | `console.log` | Structured logging (JSON) |
| Monitoring | Manual | Automated alerts |
| Security | Relaxed | Strict validation |
| Scaling | Single instance | Multiple instances |
| Updates | Anytime | Zero-downtime deployments |

---

## 🔐 Security Best Practices

### 1. Secrets Management

**❌ Never do this:**
```typescript
const API_KEY = "sk-abc123..."; // Hardcoded!
```

**✅ Do this instead:**
```typescript
const API_KEY = process.env.API_KEY;
if (!API_KEY) {
  throw new Error("API_KEY environment variable required");
}
```

**Using Docker secrets:**
```yaml
# docker-compose.yml
version: "3.8"
services:
  mcp-server:
    image: my-mcp-server
    secrets:
      - api_key
    environment:
      - API_KEY_FILE=/run/secrets/api_key

secrets:
  api_key:
    file: ./secrets/api_key.txt
```

### 2. Input Validation

```typescript
import { z } from "zod";

const ToolSchema = z.object({
  path: z.string().refine(val => !val.includes(".."), {
    message: "Path traversal detected"
  }),
  content: z.string().max(100000, "Content too large")
});

// Validate before processing
try {
  const validated = ToolSchema.parse(args);
} catch (error) {
  return { 
    content: [{ type: "text", text: "Validation error" }],
    isError: true 
  };
}
```

### 3. Rate Limiting

```typescript
import Bottleneck from "bottleneck";

const limiter = new Bottleneck({
  minTime: 100,    // Max 10 requests per second
  maxConcurrent: 5
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  return limiter.schedule(async () => {
    // Your tool logic here
  });
});
```

### 4. Sandboxing

Run servers in isolated environments:

```dockerfile
FROM node:20-alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S mcp -u 1001

WORKDIR /app
COPY --chown=mcp:nodejs . .

# Read-only filesystem
RUN chmod -R 555 /app

# Drop to non-root
USER mcp

CMD ["node", "server.js"]
```

---

## 📊 Structured Logging

```typescript
import winston from "winston";

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: "mcp-server" },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: "error.log", level: "error" }),
    new winston.transports.File({ filename: "combined.log" })
  ]
});

// Replace console.error
logger.info("Server starting", { version: "1.0.0" });
logger.error("Tool execution failed", { 
  tool: name, 
  error: error.message,
  duration: Date.now() - start
});
```

---

## 📈 Monitoring & Metrics

### Health Check Endpoint

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "__health") {
    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          status: "healthy",
          uptime: process.uptime(),
          memory: process.memoryUsage(),
          timestamp: new Date().toISOString()
        })
      }]
    };
  }
  // ...
});
```

### Prometheus Metrics

```typescript
import { collectDefaultMetrics, register, Counter, Histogram } from "prom-client";

collectDefaultMetrics();

const toolCalls = new Counter({
  name: "mcp_tool_calls_total",
  help: "Total tool calls",
  labelNames: ["tool", "status"]
});

const toolDuration = new Histogram({
  name: "mcp_tool_duration_seconds",
  help: "Tool execution duration",
  labelNames: ["tool"],
  buckets: [0.1, 0.5, 1, 2, 5]
});

// Usage
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const start = Date.now();
  
  try {
    // ... tool logic ...
    toolCalls.inc({ tool: request.params.name, status: "success" });
  } catch (error) {
    toolCalls.inc({ tool: request.params.name, status: "error" });
    throw error;
  } finally {
    toolDuration.observe(
      { tool: request.params.name },
      (Date.now() - start) / 1000
    );
  }
});
```

---

## 🚀 Deployment Patterns

### Pattern 1: Docker Container

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json .
RUN npm ci --only=production

COPY . .
RUN npm run build

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

USER node
CMD ["node", "dist/server.js"]
```

### Pattern 2: systemd Service

```ini
# /etc/systemd/system/mcp-server.service
[Unit]
Description=MCP Server
After=network.target

[Service]
Type=simple
User=mcp
Group=mcp
WorkingDirectory=/opt/mcp-server
Environment=NODE_ENV=production
EnvironmentFile=/etc/mcp-server/env
ExecStart=/usr/bin/node dist/server.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Pattern 3: Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mcp-server
  template:
    metadata:
      labels:
        app: mcp-server
    spec:
      containers:
      - name: mcp-server
        image: my-registry/mcp-server:v1.0.0
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: mcp-secrets
              key: api-key
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - node
            - healthcheck.js
          initialDelaySeconds: 30
          periodSeconds: 10
```

---

## 🔄 Zero-Downtime Updates

### Blue-Green Deployment

```
┌─────────────┐     ┌─────────────┐
│   Blue      │ ──▶ │   Green     │
│  (v1.0)     │     │  (v2.0)     │
└─────────────┘     └─────────────┘
       ▲                    │
       └──── Switch ─────────┘
```

### Rolling Updates (Kubernetes)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

---

## 🔍 Debugging in Production

### Structured Error Responses

```typescript
class MCPError extends Error {
  constructor(
    message: string,
    public code: string,
    public details?: Record<string, unknown>
  ) {
    super(message);
  }
}

// Usage
if (!isValid) {
  throw new MCPError(
    "Invalid input",
    "VALIDATION_ERROR",
    { field: "path", received: inputPath }
  );
}
```

### Request ID Tracing

```typescript
import { randomUUID } from "crypto";

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const requestId = randomUUID();
  
  logger.info("Tool request", { 
    requestId,
    tool: request.params.name,
    args: request.params.arguments 
  });
  
  try {
    // ... execute tool ...
  } catch (error) {
    logger.error("Tool failed", { 
      requestId,
      error: error.message 
    });
    throw error;
  }
});
```

---

## 📚 Production Checklist

### Security
- [ ] Secrets in environment variables
- [ ] Input validation with Zod
- [ ] Path traversal protection
- [ ] Rate limiting
- [ ] Non-root container user
- [ ] Read-only filesystem
- [ ] Network policies

### Reliability
- [ ] Health checks
- [ ] Graceful shutdown handlers
- [ ] Retry logic for external calls
- [ ] Circuit breakers
- [ ] Timeout configuration

### Observability
- [ ] Structured logging (JSON)
- [ ] Metrics collection
- [ ] Distributed tracing
- [ ] Error alerting

### Operations
- [ ] Automated deployments
- [ ] Rollback strategy
- [ ] Resource limits
- [ ] Backup procedures

---

## ✅ Practice Exercise

Complete [Exercise 4.3: Production-Ready Server](../exercises/exercise-04-3-production-server.md)

---

## 🎓 Course Complete!

Congratulations on finishing **Mastering the Model Context Protocol**!

You've learned:
1. ✅ MCP fundamentals and architecture
2. ✅ Building servers with tools, prompts, and resources
3. ✅ Advanced patterns: multi-server, subscriptions, production

**Next steps:**
- 🚀 Deploy your first MCP server
- 🌟 Contribute open-source servers
- 📖 Join the [MCP community](https://github.com/modelcontextprotocol)
