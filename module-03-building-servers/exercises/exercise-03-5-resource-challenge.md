# Exercise 3.5: Resource Server Challenge

## 🎯 Objective
Build a resource server that exposes dynamic data from multiple sources.

---

## 📋 Instructions

Create a resource server that exposes at least **3 different resource types**:

1. **File Resource** — Read project files
2. **Data Resource** — Expose application metrics/logs
3. **Custom Resource** — Your choice!

---

## 🎯 Requirements

### 1. Resource Discovery (resources/list)

Your server must implement `resources/list` that returns:

```typescript
{
  resources: [
    {
      uri: "file:///readme.md",
      name: "Project README",
      mimeType: "text/markdown",
      description: "..."
    },
    // at least 2 more resources
  ]
}
```

### 2. Resource Reading (resources/read)

Implement `resources/read` that handles:

| Resource | URI Pattern | Content | MIME Type |
|----------|-------------|---------|-----------|
| Files | `file:///[path]` | Text content | text/markdown, text/plain |
| Stats | `data:///stats` | JSON metrics | application/json |
| Custom | Your choice | Your choice | Appropriate |

### 3. Optional: Subscriptions

Implement `resources/subscribe` and `resources/unsubscribe`:

```typescript
server.setRequestHandler(SubscribeRequestSchema, async (request) => {
  const { uri } = request.params;
  // Set up watcher for uri
  return {};
});
```

---

## 💻 Starter Code

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ReadResourceRequestSchema,
  ListResourcesRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const BASE_DIR = "/path/to/project";

const server = new Server(
  { name: "resources-challenge", version: "1.0.0" },
  { 
    capabilities: { 
      resources: { subscribe: true, listChanged: true }
    } 
  }
);

// TODO: Implement ListResourcesRequestSchema

// TODO: Implement ReadResourceRequestSchema
//   - file:///* — read from BASE_DIR
//   - data:///stats — return project stats
//   - Your custom resource

const transport = new StdioServerTransport();
server.connect(transport);
```

---

## 🧪 Testing

Test each resource:

1. **List resources**:
   ```json
   { "method": "resources/list" }
   ```

2. **Read a file**:
   ```json
   {
     "method": "resources/read",
     "params": { "uri": "file:///readme.md" }
   }
   ```

3. **Read stats**:
   ```json
   {
     "method": "resources/read",
     "params": { "uri": "data:///stats" }
   }
   ```

4. **Your custom resource**:
   ```json
   {
     "method": "resources/read",
     "params": { "uri": "your:///custom" }
   }
   ```

---

## 💡 Ideas for Custom Resource

- **weather:///current** — Current weather data
- **news:///latest** — Latest news headlines
- **stocks:///quote?symbol=AAPL** — Stock price
- **joke:///random** — Random programming joke
- **time:///now** — Current time in different zones

---

## ✅ Validation Checklist

- [ ] `resources/list` returns all resources
- [ ] `resources/read` handles file:///
- [ ] `resources/read` handles data:///
- [ ] `resources/read` handles your custom scheme
- [ ] Proper MIME types returned
- [ ] Errors handled gracefully
- [ ] Subscriptions work (optional)

---

## 🏆 Bonus Challenges

- [ ] Add 5+ resources
- [ ] Implement file watching for subscriptions
- [ ] Add resource caching
- [ ] Support binary resources (images)
- [ ] Custom URI scheme with query parameters

---

## 📚 Example Solution: Weather Resource

```typescript
case "weather:": {
  if (url.pathname === "/current") {
    // Simulated weather data
    content = JSON.stringify({
      temperature: 22,
      unit: "celsius",
      condition: "partly cloudy",
      humidity: 65,
      updated: new Date().toISOString()
    }, null, 2);
    mimeType = "application/json";
  }
  break;
}
```

---

## 🔗 Next

Congratulations! 🎉 Move to [Module 4: Advanced Patterns](../../module-04-advanced/)
