# Exercise 4.2: Real-Time Log Watcher

## 🎯 Objective
Build an MCP server that watches log files and notifies clients of updates.

---

## 📋 Instructions

Create a server that:
1. Allows clients to subscribe to log file resources
2. Watches files for changes
3. Notifies subscribers of updates
4. Cleans up watchers on unsubscribe

---

## 🎨 Requirements

### Resources
| URI | Description |
|-----|-------------|
| `file:///app.log` | Application logs |
| `file:///error.log` | Error logs |
| `logs:///latest` | Last 50 lines (simulated) |

### Capabilities
- `resources.subscribe: true`
- `resources.listChanged: true`

---

## 💻 Implementation Template

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
  SubscribeRequestSchema,
  UnsubscribeRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import * as fs from "fs";
import * as path from "path";

const LOG_DIR = "/var/log/app";
const watchers: Map<string, fs.FSWatcher> = new Map();
const subscribers: Map<string, Set<string>> = new Map();

// TODO: Implement resources/list
// TODO: Implement resources/read
// TODO: Implement resources/subscribe with file watcher
// TODO: Implement resources/unsubscribe with cleanup
// TODO: Send notifications/resources/updated on change
```

---

## 🔍 Implementation Hints

### Setting Up File Watcher

```typescript
const watcher = fs.watch(filePath, (eventType) => {
  if (eventType === "change") {
    // File changed, notify subscribers
    notifySubscribers(uri);
  }
});
```

### Sending Notifications

```typescript
await server.notification({
  method: "notifications/resources/updated",
  params: { uri }
});
```

### Cleanup

```typescript
// Close watcher
watcher.close();
watchers.delete(filePath);

// Clean up subscribers
subscribers.delete(uri);
```

---

## ✅ Requirements Checklist

- [ ] Implements `resources/list`
- [ ] Implements `resources/read`
- [ ] Implements `resources/subscribe`
- [ ] Implements `resources/unsubscribe`
- [ ] Uses `fs.watch()` for file monitoring
- [ ] Sends `notifications/resources/updated`
- [ ] Cleans up watchers on unsubscribe
- [ ] Handles multiple subscribers to same resource
- [ ] Graceful shutdown (close all watchers)

---

## 🧪 Testing

```bash
# Terminal 1: Start server
node log-watcher-server.js

# Terminal 2: Subscribe with MCP Inspector
# resources/subscribe {"uri": "file:///app.log"}

# Terminal 3: Modify log
mkdir -p /var/log/app
echo "New log entry" >> /var/log/app/app.log

# Should see notification in Terminal 2
```

---

## 🏆 Bonus Challenges

- [ ] Add log rotation support
- [ ] Implement "tail -f" behavior with streaming
- [ ] Add filtering (only notify on error lines)
- [ ] Implement polling fallback for non-watchable files
- [ ] Add line count resource (e.g., `logs:///app.log/lines/100`)

---

## 💡 Tips

- Use `path.resolve()` for safe path handling
- Close watchers on process exit (`process.on("SIGINT", ...)`)
- Consider using `chokidar` for cross-platform watching
- Test with both file modifications and deletions

---

## 🔗 Next

Continue to [Lesson 4.3: Production Deployment](../lesson-03-production.md)
