# Exercise 4.3: Production-Ready Server

## 🎯 Objective
Take an MCP server and make it production-ready.

---

## 📋 Instructions

Choose any server from previous lessons (or create a new one) and add:

1. **Security** — Input validation, secrets management
2. **Observability** — Structured logging, metrics, health checks
3. **Reliability** — Error handling, graceful shutdown
4. **Deployment** — Docker container, CI/CD ready

---

## 🎨 Requirements

### 1. Security (4 points)

- [ ] Secrets from environment variables (never hardcoded)
- [ ] Input validation with schema (Zod)
- [ ] Path traversal protection (if filesystem access)
- [ ] Rate limiting (optional but recommended)

**Example:**
```typescript
// ❌ Bad
const API_KEY = "sk-abc123";

// ✅ Good
const API_KEY = process.env.API_KEY;
if (!API_KEY) {
  throw new Error("API_KEY required");
}

// ✅ Validation
const args = Schema.parse(request.params.arguments);
```

### 2. Observability (4 points)

- [ ] Structured JSON logging
- [ ] Request ID tracing
- [ ] Metrics collection (optional)
- [ ] Health check endpoint

**Example:**
```typescript
import winston from "winston";

const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [new winston.transports.Console()]
});

// Usage
const requestId = randomUUID();
logger.info("Request started", { requestId, tool: name });
```

### 3. Reliability (4 points)

- [ ] Graceful shutdown handler
- [ ] Proper error responses
- [ ] Timeout handling
- [ ] Retry logic for external calls

**Example:**
```typescript
// Graceful shutdown
process.on("SIGINT", async () => {
  logger.info("Shutting down...");
  await server.close();
  process.exit(0);
});

// Error handling
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    // ... logic ...
  } catch (error) {
    logger.error("Tool failed", { error: error.message });
    return {
      content: [{ type: "text", text: `Error: ${error.message}` }],
      isError: true
    };
  }
});
```

### 4. Deployment (4 points)

- [ ] Dockerfile
- [ ] docker-compose.yml
- [ ] Health check script
- [ ] README with deployment instructions

**Dockerfile:**
```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json .
RUN npm ci --only=production

COPY . .
RUN npm run build

USER node
EXPOSE 3000

HEALTHCHECK --interval=30s CMD node healthcheck.js

CMD ["node", "dist/server.js"]
```

---

## 💻 Assessment Rubric

| Aspect | Points | Criteria |
|--------|--------|----------|
| Security | 4 | Env vars, validation, path safety, rate limit |
| Observability | 4 | JSON logs, tracing, metrics, health check |
| Reliability | 4 | Shutdown, errors, timeout, retry |
| Deployment | 4 | Docker, compose, health script, docs |
| **Total** | **16** | **12+ to pass** |

---

## 📁 Submission Structure

```
my-production-server/
├── src/
│   ├── server.ts          # Main server
│   ├── validation.ts      # Input schemas
│   └── logging.ts         # Logger config
├── dist/                  # Compiled output
├── Dockerfile
├── docker-compose.yml
├── healthcheck.js         # Health check script
├── .env.example           # Example environment
├── security-checklist.md  # Security audit
└── README.md              # Deployment guide
```

---

## 🏆 Bonus Challenges

- [ ] Add Prometheus metrics endpoint
- [ ] Implement circuit breaker pattern
- [ ] Add distributed tracing (OpenTelemetry)
- [ ] Create Kubernetes manifests
- [ ] Add security headers
- [ ] Implement least-privilege container

---

## 💡 Production Checklist

### Before Deploying
- [ ] All secrets in environment variables
- [ ] No `console.log`, use structured logging
- [ ] All inputs validated
- [ ] Error messages don't leak sensitive data
- [ ] Health checks implemented
- [ ] Graceful shutdown tested
- [ ] Resource limits configured
- [ ] Security audit completed

### Documentation
- [ ] README with setup instructions
- [ ] Environment variables documented
- [ ] Deployment guide
- [ ] Troubleshooting guide
- [ ] Monitoring/alerting runbook

---

## 🎓 Congratulations!

You now have a **production-ready MCP server** that you can confidently deploy.

This exercise completes the **Mastering MCP Protocol** course. 🎉

---

## 🔗 Course Complete!

Return to [Course Home](../../) or explore [Additional Resources](../../resources/).
