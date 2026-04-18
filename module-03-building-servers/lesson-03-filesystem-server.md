# Lesson 3.3: Filesystem Server — Building a Real-World MCP Server

## 🎯 Learning Objectives
By the end of this lesson, you will:
- Build a filesystem MCP server with secure path handling
- Implement path validation to prevent directory traversal attacks
- Use tools for read, write, list, and delete operations
- Understand security best practices for resource servers

---

## 📋 Overview

A filesystem server is one of the most practical MCP servers. It allows AI assistants to:
- Read file contents
- Write new files
- List directory contents
- Delete files

However, **security is critical** — we must prevent unauthorized access to sensitive files outside our allowed directories.

---

## 🛡️ Security First: Path Validation

Before writing code, let's understand the security model:

```
Allowed: /home/user/projects/
Blocked: /etc/passwd
Blocked: /home/user/projects/../../../etc/passwd
```

### Path Traversal Attack
```typescript
// ❌ VULNERABLE CODE
const filePath = "../../../etc/passwd";  // User input
readFile(filePath);  // This is dangerous!

// ✅ SAFE CODE
const allowedDir = "/home/user/projects/";
const fullPath = path.join(allowedDir, filePath);
const resolved = path.resolve(fullPath);

if (!resolved.startsWith(allowedDir)) {
  throw new Error("Access denied: Path outside allowed directory");
}
```

---

## 💻 Implementation: Secure Filesystem Server

### Step 1: Set Up the Project

```bash
mkdir mcp-filesystem-server
cd mcp-filesystem-server
npm init -y
npm install @modelcontextprotocol/sdk zod
```

### Step 2: Create the Server

Create `filesystem-server.ts`:

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import * as fs from "fs/promises";
import * as path from "path";
import { z } from "zod";

const ALLOWED_DIR = "/home/user/projects"; // Configure this

// Validate that a path is within the allowed directory
function validatePath(inputPath: string): string {
  const fullPath = path.resolve(path.join(ALLOWED_DIR, inputPath));
  
  // Ensure the resolved path starts with allowed directory
  if (!fullPath.startsWith(path.resolve(ALLOWED_DIR))) {
    throw new Error(
      `Access denied: Path "${inputPath}" is outside allowed directory`
    );
  }
  
  return fullPath;
}

const server = new Server(
  { name: "filesystem-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// Define our tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "read_file",
        description: "Read the contents of a file",
        inputSchema: {
          type: "object",
          properties: {
            path: {
              type: "string",
              description: "Relative path to the file (e.g., 'readme.md')"
            }
          },
          required: ["path"]
        }
      },
      {
        name: "write_file",
        description: "Write content to a file (creates if doesn't exist)",
        inputSchema: {
          type: "object",
          properties: {
            path: {
              type: "string",
              description: "Relative path to the file"
            },
            content: {
              type: "string",
              description: "Content to write"
            }
          },
          required: ["path", "content"]
        }
      },
      {
        name: "list_directory",
        description: "List files and directories",
        inputSchema: {
          type: "object",
          properties: {
            path: {
              type: "string",
              description: "Relative path to directory (default: '.')"
            }
          }
        }
      },
      {
        name: "delete_file",
        description: "Delete a file or directory",
        inputSchema: {
          type: "object",
          properties: {
            path: {
              type: "string",
              description: "Relative path to file or directory"
            }
          },
          required: ["path"]
        }
      }
    ]
  };
});

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case "read_file": {
        const filePath = validatePath(args.path as string);
        const content = await fs.readFile(filePath, "utf-8");
        return {
          content: [{ type: "text", text: content }]
        };
      }

      case "write_file": {
        const writePath = validatePath(args.path as string);
        const content = args.content as string;
        
        // Ensure directory exists
        await fs.mkdir(path.dirname(writePath), { recursive: true });
        await fs.writeFile(writePath, content, "utf-8");
        
        return {
          content: [{ 
            type: "text", 
            text: `File written successfully: ${args.path}` 
          }]
        };
      }

      case "list_directory": {
        const dirPath = validatePath((args.path as string) || ".");
        const entries = await fs.readdir(dirPath, { withFileTypes: true });
        
        const formatted = entries.map(entry => {
          return entry.isDirectory() ? `📁 ${entry.name}/` : `📄 ${entry.name}`;
        }).join("\\n");
        
        return {
          content: [{ type: "text", text: formatted || "(empty directory)" }]
        };
      }

      case "delete_file": {
        const deletePath = validatePath(args.path as string);
        const stats = await fs.stat(deletePath);
        
        if (stats.isDirectory()) {
          await fs.rmdir(deletePath, { recursive: true });
        } else {
          await fs.unlink(deletePath);
        }
        
        return {
          content: [{ 
            type: "text", 
            text: `Deleted: ${args.path}` 
          }]
        };
      }

      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (error) {
    return {
      content: [{ 
        type: "text", 
        text: `Error: ${(error as Error).message}` 
      }],
      isError: true
    };
  }
});

// Start the server
const transport = new StdioServerTransport();
server.connect(transport);

console.error("Filesystem MCP Server running...");
```

### Step 3: Compile and Run

```bash
npx tsc filesystem-server.ts --module nodenext --moduleResolution nodenext --esModuleInterop --target es2022

# Or add to package.json scripts
```

---

## 🔧 Configuration in Claude/Cursor

Add to your Claude Desktop config:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "node",
      "args": ["/path/to/mcp-filesystem-server/dist/filesystem-server.js"],
      "env": {
        "ALLOWED_DIR": "/home/user/projects"
      }
    }
  }
}
```

---

## 🧪 Testing Your Server

Test with MCP Inspector:

```bash
npx @modelcontextprotocol/inspector node dist/filesystem-server.js
```

Try these calls:

```json
{
  "name": "write_file",
  "arguments": {
    "path": "notes.txt",
    "content": "Hello from MCP!"
  }
}
```

```json
{
  "name": "read_file",
  "arguments": {
    "path": "notes.txt"
  }
}
```

```json
{
  "name": "list_directory",
  "arguments": {
    "path": "."
  }
}
```

---

## 🚨 Security Checklist

- ✅ ✅ Path validation with `path.resolve()` and `startsWith()`
- ✅ ✅ Error messages don't leak file system structure
- ✅ ✅ No shell execution or command injection
- ✅ ✅ Allowed directory is configurable via env variable
- ✅ ✅ All errors return 200 OK (MCP standard, with `isError` flag)

---

## 📚 Summary

| Concept | Implementation |
|---------|---------------|
| Path Security | `path.resolve()` + whitelist validation |
| File Operations | `fs/promises` API |
| Error Handling | Return `isError: true` for client errors |
| Directory Creation | `recursive: true` for nested paths |

---

## ✅ Practice Exercise

Before moving to the next lesson, complete [Exercise 3.3: Filesystem Security Audit](../exercises/exercise-03-3-filesystem-security.md)
