# Exercise 3.3: Filesystem Security Audit

## 🎯 Objective
Review and fix security vulnerabilities in a filesystem server.

---

## 📋 Instructions

You're given a filesystem server with security vulnerabilities. Your task is to:
1. Identify the vulnerabilities
2. Explain why they're dangerous
3. Provide the fixed code

---

## 🔍 Vulnerable Code

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { CallToolRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import * as fs from "fs/promises";

const ALLOWED_DIR = "/home/user/projects";

const server = new Server(
  { name: "filesystem-bad", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "read_file") {
    // ❌ VULNERABLE: No path validation
    const content = await fs.readFile(args.path, "utf-8");
    return { content: [{ type: "text", text: content }] };
  }
  
  if (name === "write_file") {
    // ❌ VULNERABLE: No validation, creates any path
    await fs.writeFile(args.path, args.content);
    return { content: [{ type: "text", text: "Written" }] };
  }
  
  if (name === "delete_file") {
    // ❌ VULNERABLE: Can delete any file
    await fs.unlink(args.path);
    return { content: [{ type: "text", text: "Deleted" }] };
  }
});
```

---

## ❓ Vulnerability Analysis

### Issue 1: Path Traversal
**Problem:**
```
args.path = "../../../etc/passwd"
// Reads /etc/passwd! ❌
```

### Issue 2: Arbitrary File Creation
**Problem:**
```
args.path = "/home/other-user/secret.txt"
// Overwrites any file! ❌
```

### Issue 3: Arbitrary File Deletion
**Problem:**
```
args.path = "/etc/passwd"
// Deletes system file! ❌
```

---

## ✅ Solution

### Fixed validatePath Function

```typescript
import * as path from "path";

function validatePath(inputPath: string): string {
  // 1. Join with allowed directory
  const fullPath = path.join(ALLOWED_DIR, inputPath);
  
  // 2. Resolve to absolute path (removes ../ sequences)
  const resolved = path.resolve(fullPath);
  
  // 3. Ensure it starts with allowed directory
  if (!resolved.startsWith(path.resolve(ALLOWED_DIR))) {
    throw new Error("Access denied: Path outside allowed directory");
  }
  
  // 4. Return the safe path
  return resolved;
}
```

### Fixed Tool Handlers

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "read_file") {
    const safePath = validatePath(args.path);
    const content = await fs.readFile(safePath, "utf-8");
    return { content: [{ type: "text", text: content }] };
  }
  
  if (name === "write_file") {
    const safePath = validatePath(args.path);
    
    // Optional: Ensure parent directory exists
    await fs.mkdir(path.dirname(safePath), { recursive: true });
    
    await fs.writeFile(safePath, args.content);
    return { content: [{ type: "text", text: "Written" }] };
  }
  
  if (name === "delete_file") {
    const safePath = validatePath(args.path);
    await fs.unlink(safePath);
    return { content: [{ type: "text", text: "Deleted" }] };
  }
});
```

---

## 🧪 Test Cases

Your solution should pass these tests:

| Test | Input | Expected |
|------|-------|----------|
| Valid read | `"readme.md"` | ✅ Success |
| Subdirectory | `"src/app.ts"` | ✅ Success |
| Path traversal | `"../../../etc/passwd"` | ❌ Access denied |
| Absolute escape | `"/etc/passwd"` | ❌ Access denied |
| Double escape | `"foo/../../../etc/passwd"` | ❌ Access denied |

---

## 📚 Security Checklist

- [ ] Path validation using `path.resolve()`
- [ ] Whitelist checking with `startsWith()`
- [ ] Error messages don't leak file system structure
- [ ] Parent directory creation with `recursive: true`
- [ ] Only allowed operations (read/write/delete)

---

## 🔗 Next

Continue to [Lesson 3.4: Prompts Server](../lesson-04-prompts-server.md)
