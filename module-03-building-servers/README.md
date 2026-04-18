# Module 3: Building Your First MCP Server 💻

> **Hands-on development: From zero to working MCP server.**
>
> This module guides you through building real MCP servers using official SDKs.

---

## 📚 Module Overview

| Item | Details |
|------|---------|
| **Duration** | ~90 minutes (coding + testing) |
| **Level** | Intermediate — Completion of Modules 1-2 recommended |
| **Prerequisites** | Node.js 18+ or Python 3.10+, Module 1-2 |
| **Outcome** | Build and test a complete MCP server |

---

## 🎯 Learning Objectives

By the end of this module, you will:

1. ✅ **Set up** the MCP SDK development environment
2. ✅ **Build** a simple MCP server from scratch
3. ✅ **Add** tools, resources, and prompts to your server
4. ✅ **Test** your server using MCP Inspector
5. ✅ **Debug** common MCP development issues
6. ✅ **Package** your server for distribution

---

## 🛠️ Prerequisites Setup

### Option A: TypeScript/Node.js

```bash
# Check Node.js version (needs 18+)
node --version

# Create project directory
mkdir my-first-mcp-server
cd my-first-mcp-server

# Initialize project
npm init -y

# Install MCP SDK
npm install @modelcontextprotocol/sdk

# Install TypeScript (optional but recommended)
npm install -D typescript @types/node
npx tsc --init

# Create source directory
mkdir src
```

### Option B: Python

```bash
# Check Python version (needs 3.10+)
python3 --version

# Create project directory
mkdir my-first-mcp-server
cd my-first-mcp-server

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate  # Windows

# Install MCP SDK
pip install mcp

# Create source directory
mkdir src
```

---

## 📖 Lesson Structure

### Lesson 3.1: Hello World Server
**Duration:** 15 minutes

#### The Simplest MCP Server

Let's build the most basic MCP server that just says "Hello".

##### TypeScript Version

**`src/hello-server.ts`**

```typescript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// Create server instance
const server = new Server(
  {
    name: "hello-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Define available tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "hello",
        description: "Say hello to someone",
        inputSchema: {
          type: "object",
          properties: {
            name: {
              type: "string",
              description: "Name of the person to greet",
            },
          },
          required: ["name"],
        },
      },
    ],
  };
});

// Handle tool execution
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "hello") {
    const name = request.params.arguments?.name;
    return {
      content: [
        {
          type: "text",
          text: `Hello, ${name}! Welcome to MCP.`,
        },
      ],
    };
  }
  throw new Error(`Unknown tool: ${request.params.name}`);
});

// Start server using stdio transport
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Hello server running on stdio");
}

main().catch(console.error);
```

**`package.json`**

```json
{
  "name": "hello-mcp-server",
  "version": "1.0.0",
  "description": "My first MCP server",
  "type": "module",
  "bin": {
    "hello-server": "./dist/hello-server.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/hello-server.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

**Build and run:**

```bash
npm run build
node dist/hello-server.js
```

##### Python Version

**`src/hello_server.py`**

```python
#!/usr/bin/env python3
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# Create server
app = Server("hello-server")


@app.list_tools()
async def list_tools() -> list[Tool]:
    """List available tools."""
    return [
        Tool(
            name="hello",
            description="Say hello to someone",
            input_schema={
                "type": "object",
                "properties": {
                    "name": {
                        "type": "string",
                        "description": "Name of the person to greet"
                    }
                },
                "required": ["name"]
            }
        )
    ]


@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Execute a tool."""
    if name == "hello":
        person_name = arguments.get("name", "World")
        return [
            TextContent(
                type="text",
                text=f"Hello, {person_name}! Welcome to MCP."
            )
        ]
    raise ValueError(f"Unknown tool: {name}")


async def main():
    """Run the server."""
    async with stdio_server() as (read_stream, write_stream):
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options()
        )


if __name__ == "__main__":
    asyncio.run(main())
```

**`pyproject.toml`**

```toml
[project]
name = "hello-mcp-server"
version = "1.0.0"
description = "My first MCP server"
requires-python = ">=3.10"
dependencies = [
    "mcp>=1.0.0",
]

[project.scripts]
hello-server = "src.hello_server:main"
```

**Run:**

```bash
python src/hello_server.py
```

#### Test with MCP Inspector

```bash
# Install MCP Inspector (global, one-time)
npm install -g @modelcontextprotocol/inspector

# Test your server TypeScript:
mcp-inspector node dist/hello-server.js

# Test your server Python:
mcp-inspector python src/hello_server.py
```

**Inspector UI:**
- Navigate to `http://localhost:5173`
- Click "List Tools" to see available tools
- Try "Call hello tool" with: `{ "name": "MCP Student" }`

---

### Lesson 3.2: Calculator Server
**Duration:** 20 minutes

Let's build a more useful server — a calculator with multiple operations.

##### TypeScript Version

**`src/calc-server.ts`**

```typescript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "calc-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// Define all calculator tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "add",
        description: "Add two numbers",
        inputSchema: {
          type: "object",
          properties: {
            a: { type: "number", description: "First number" },
            b: { type: "number", description: "Second number" },
          },
          required: ["a", "b"],
        },
      },
      {
        name: "subtract",
        description: "Subtract second number from first",
        inputSchema: {
          type: "object",
          properties: {
            a: { type: "number", description: "First number" },
            b: { type: "number", description: "Second number" },
          },
          required: ["a", "b"],
        },
      },
      {
        name: "multiply",
        description: "Multiply two numbers",
        inputSchema: {
          type: "object",
          properties: {
            a: { type: "number", description: "First number" },
            b: { type: "number", description: "Second number" },
          },
          required: ["a", "b"],
        },
      },
      {
        name: "divide",
        description: "Divide first number by second",
        inputSchema: {
          type: "object",
          properties: {
            a: { type: "number", description: "Numerator" },
            b: { type: "number", description: "Denominator" },
          },
          required: ["a", "b"],
        },
      },
      {
        name: "power",
        description: "Raise first number to the power of second",
        inputSchema: {
          type: "object",
          properties: {
            base: { type: "number", description: "Base number" },
            exponent: { type: "number", description: "Exponent" },
          },
          required: ["base", "exponent"],
        },
      },
    ],
  };
});

// Handle tool execution
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  let result: number;

  switch (name) {
    case "add":
      result = (args?.a as number) + (args?.b as number);
      break;
    case "subtract":
      result = (args?.a as number) - (args?.b as number);
      break;
    case "multiply":
      result = (args?.a as number) * (args?.b as number);
      break;
    case "divide":
      if ((args?.b as number) === 0) {
        throw new Error("Cannot divide by zero");
      }
      result = (args?.a as number) / (args?.b as number);
      break;
    case "power":
      result = Math.pow(args?.base as number, args?.exponent as number);
      break;
    default:
      throw new Error(`Unknown tool: ${name}`);
  }

  return {
    content: [
      {
        type: "text",
        text: String(result),
      },
    ],
  };
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Calculator server running on stdio");
}

main().catch(console.error);
```

##### Test It

```bash
# Build
npm run build

# Test with inspector
mcp-inspector node dist/calc-server.js
```

**Expected behavior:**
- AI can now say: "Add 5 and 3" → MCP calls `add(5, 3)` → Returns 8
- AI can now say: "What's 10 to the power of 2?" → MCP calls `power(10, 2)` → Returns 100

##### Error Handling

Notice how we handle divide-by-zero:

```typescript
case "divide":
  if ((args?.b as number) === 0) {
    throw new Error("Cannot divide by zero");
  }
  // ...
```

This throws an error that the MCP host can catch and display to the user.

---

### Lesson 3.3: File System Server with Resources
**Duration:** 25 minutes

Now let's add **Resources** — the ability to read files.

##### TypeScript Version

**`src/files-server.ts`** (simplified version)

```typescript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import * as fs from "fs/promises";
import * as path from "path";

// Get allowed directory from command line
const allowedDir = process.argv[2] || process.cwd();

const server = new Server(
  { name: "files-server", version: "1.0.0" },
  {
    capabilities: {
      tools: {},
      resources: {},
    },
  }
);

// === TOOLS ===

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "list_directory",
        description: "List files in a directory",
        inputSchema: {
          type: "object",
          properties: {
            dirPath: {
              type: "string",
              description: "Directory path (relative to allowed directory)",
            },
          },
        },
      },
      {
        name: "read_file",
        description: "Read contents of a file",
        inputSchema: {
          type: "object",
          properties: {
            filePath: {
              type: "string",
              description: "File path (relative to allowed directory)",
            },
          },
          required: ["filePath"],
        },
      },
    ],
  };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  // Security: Resolve path within allowed directory
  const resolvePath = (inputPath: string) => {
    const resolved = path.resolve(allowedDir, inputPath);
    if (!resolved.startsWith(allowedDir)) {
      throw new Error("Access denied: Path outside allowed directory");
    }
    return resolved;
  };

  switch (name) {
    case "list_directory": {
      const dirPath = resolvePath((args?.dirPath as string) || ".");
      const files = await fs.readdir(dirPath, { withFileTypes: true });
      const result = files.map((f) => ({
        name: f.name,
        type: f.isDirectory() ? "directory" : "file",
      }));
      return {
        content: [{ type: "text", text: JSON.stringify(result, null, 2) }],
      };
    }

    case "read_file": {
      const filePath = resolvePath(args?.filePath as string);
      const content = await fs.readFile(filePath, "utf-8");
      return {
        content: [{ type: "text", text: content }],
      };
    }

    default:
      throw new Error(`Unknown tool: ${name}`);
  }
});

// === RESOURCES ===

// List available resources (files in allowed directory)
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  const files = await fs.readdir(allowedDir);
  const txtFiles = files.filter((f) => f.endsWith(".txt") || f.endsWith(".md"));

  return {
    resources: txtFiles.map((file) => ({
      uri: `file:///${path.join(allowedDir, file)}`,
      name: file,
      mimeType: file.endsWith(".md")
        ? "text/markdown"
        : "text/plain",
    })),
  };
});

// Read a specific resource
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const uri = request.params.uri;
  const filePath = uri.replace("file:///", "");

  // Security check
  const resolved = path.resolve(filePath);
  if (!resolved.startsWith(path.resolve(allowedDir))) {
    throw new Error("Access denied");
  }

  const content = await fs.readFile(filePath, "utf-8");

  return {
    contents: [
      {
        uri,
        mimeType: filePath.endsWith(".md") ? "text/markdown" : "text/plain",
        text: content,
      },
    ],
  };
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error(`Files server running on stdio (allowed: ${allowedDir})`);
}

main().catch(console.error);
```

##### Key Concepts

**1. Path Security**:
```typescript
const resolvePath = (inputPath: string) => {
  const resolved = path.resolve(allowedDir, inputPath);
  if (!resolved.startsWith(allowedDir)) {
    throw new Error("Access denied: Path outside allowed directory");
  }
  return resolved;
};
```
Always validate paths to prevent directory traversal attacks!

**2. Resource Listing**:
```typescript
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      { uri: "file:///path/to/file.txt", name: "file.txt", mimeType: "text/plain" }
    ]
  };
});
```

**3. Resource Reading**:
```typescript
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const content = await fs.readFile(request.params.uri);
  return { contents: [{ uri, mimeType, text: content }] };
});
```

##### Configuration

Add to your MCP host config:

```json
{
  "mcpServers": {
    "files": {
      "command": "node",
      "args": ["/path/to/dist/files-server.js", "/home/user/documents"]
    }
  }
}
```

---

### Lesson 3.4: Adding Prompts
**Duration:** 15 minutes

**Prompts** are pre-defined templates that help the AI generate consistent outputs.

##### TypeScript Version

Add to your server:

```typescript
import {
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// Update capabilities
const server = new Server(
  { name: "advanced-server", version: "1.0.0" },
  {
    capabilities: {
      tools: {},
      resources: {},
      prompts: {},  // ← Enable prompts
    },
  }
);

// List available prompts
server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [
      {
        name: "explain_code",
        description: "Explain code in simple terms",
        arguments: [
          {
            name: "language",
            description: "Programming language",
            required: true,
          },
          {
            name: "code",
            description: "Code to explain",
            required: true,
          },
        ],
      },
      {
        name: "code_review",
        description: "Review code for best practices",
        arguments: [
          {
            name: "code",
            description: "Code to review",
            required: true,
          },
        ],
      },
    ],
  };
});

// Return prompt content
server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "explain_code") {
    return {
      description: "Code explanation prompt",
      messages: [
        {
          role: "user",
          content: {
            type: "text",
            text: `Please explain the following ${args?.language} code in simple terms:

\`\`\`${args?.language}
${args?.code}
\`\`\`

Break down:
1. What the code does
2. Key concepts used
3. Potential improvements`,
          },
        },
      ],
    };
  }

  if (name === "code_review") {
    return {
      description: "Code review prompt",
      messages: [
        {
          role: "user",
          content: {
            type: "text",
            text: `Please review this code for:

1. Best practices
2. Potential bugs
3. Performance issues
4. Security concerns
5. Code style

\`\`\`
${args?.code}
\`\`\``,
          },
        },
      ],
    };
  }

  throw new Error(`Unknown prompt: ${name}`);
});
```

---

### Lesson 3.5: Testing & Debugging
**Duration:** 15 minutes

#### MCP Inspector

The official testing tool:

```bash
# Install once
npm install -g @modelcontextprotocol/inspector

# Usage
mcp-inspector <command> [args...]

# Examples:
mcp-inspector node dist/my-server.js
mcp-inspector python my_server.py
mcp-inspector npx -y @modelcontextprotocol/server-filesystem /path
```

**Features:**
- 🎨 UI at `http://localhost:5173`
- 📋 List and test all tools
- 📄 View resources
- 📝 Test prompts
- 🔍 View raw JSON-RPC messages

#### Logging

Use `console.error` for server logs (stdout is for MCP protocol):

```typescript
console.error("[Server] Starting up...");
console.error("[Server] Connected to transport");
console.error("[Server] Received request:", request.params.name);
```

#### Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Connection refused` | Server crashed | Check stderr for errors |
| `Unknown tool` | Tool name mismatch | Check tool registration |
| `Invalid schema` | JSON Schema errors | Validate schema syntax |
| `Access denied` | Security check failed | Check path permissions |
| `Timeout` | Tool takes too long | Add async handling |

#### Debugging Tips

1. **Start with stdio** — Easier to debug than HTTP
2. **Use Inspector first** — Before testing with Claude
3. **Check stderr** — All logs go there
4. **Test incrementally** — Add one tool at a time
5. **Validate schemas** — Use JSON Schema validators

---

## ✏️ Exercises

### Exercise 3.1: Temperature Converter

Build an MCP server with a tool `convert_temperature` that:
- Takes: value (number), from_unit ("celsius" | "fahrenheit" | "kelvin"), to_unit (same)
- Returns: converted temperature

**Hint:** Formulas:
- C to F: `(C × 9/5) + 32`
- F to C: `(F - 32) × 5/9`
- K = C + 273.15

<details>
<summary>✅ Solution</summary>

```typescript
{
  name: "convert_temperature",
  description: "Convert between temperature units",
  inputSchema: {
    type: "object",
    properties: {
      value: { type: "number" },
      from_unit: { enum: ["celsius", "fahrenheit", "kelvin"] },
      to_unit: { enum: ["celsius", "fahrenheit", "kelvin"] }
    },
    required: ["value", "from_unit", "to_unit"]
  }
}

// Handler:
case "convert_temperature": {
  const { value, from_unit, to_unit } = args as any;
  
  // Convert to celsius first
  let celsius = value;
  if (from_unit === "fahrenheit") celsius = (value - 32) * 5/9;
  if (from_unit === "kelvin") celsius = value - 273.15;
  
  // Convert from celsius to target
  let result = celsius;
  if (to_unit === "fahrenheit") result = (celsius * 9/5) + 32;
  if (to_unit === "kelvin") result = celsius + 273.15;
  
  return { content: [{ type: "text", text: String(result) }] };
}
```

</details>

---

### Exercise 3.2: To-Do List Server

Build a server with CRUD operations for a todo list:
- `add_todo`: Add a new task
- `list_todos`: List all tasks
- `complete_todo`: Mark a task done
- `delete_todo`: Remove a task

Store todos in memory (use a Map or array).

---

### Exercise 3.3: Weather Resource

Add a resource `weather://current` to your server that:
- Returns current "weather" (can be mock data)
- Updates every call

Hint: Use `ListResourcesRequestSchema` and `ReadResourceRequestSchema`.

---

### Exercise 3.4: Bug Hunt

This code has a bug. Find and fix it:

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const args = request.params.arguments;
  
  if (request.params.name === "divide") {
    // BUG: What if b is 0?
    const result = args.a / args.b;
    return { content: [{ type: "text", text: String(result) }] };
  }
});
```

<details>
<summary>✅ Solution</summary>

```typescript
if (request.params.name === "divide") {
  if (args.b === 0) {
    throw new Error("Cannot divide by zero");
  }
  const result = args.a / args.b;
  return { content: [{ type: "text", text: String(result) }] };
}
```

</details>

---

### Exercise 3.5: Package Your Server

Prepare your server for distribution:

1. **Create proper `package.json`**:
```json
{
  "name": "@yourname/mcp-todo-server",
  "version": "1.0.0",
  "description": "MCP server for managing todos",
  "type": "module",
  "bin": {
    "mcp-todo-server": "./dist/index.js"
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc",
    "prepare": "npm run build"
  },
  "keywords": ["mcp", "model-context-protocol"],
  "license": "MIT"
}
```

2. **Add license and README**
3. **Test installation**: `npm install -g .`
4. **Publish to npm** (optional): `npm publish --access public`

---

## 📋 Summary

### Key Takeaways

1. ✅ **Setup:** Install SDK (`@modelcontextprotocol/sdk` or `mcp`)
2. ✅ **Basic server:** Create instance, set handlers, connect transport
3. ✅ **Tools:** Implement `list_tools` and `call_tool` handlers
4. ✅ **Resources:** Implement `list_resources` and `read_resource` handlers
5. ✅ **Prompts:** Implement `list_prompts` and `get_prompt` handlers
6. ✅ **Testing:** Use MCP Inspector via `npx @modelcontextprotocol/inspector`
7. ✅ **Security:** Always validate paths and sanitize inputs

### Project Structure

```
my-mcp-server/
├── src/
│   └── index.ts          # Server code
├── dist/
│   └── index.js          # Compiled output
├── package.json
├── tsconfig.json
└── README.md
```

### Next Steps

➡️ **Module 4:** Advanced patterns — multi-server, subscriptions, production deployment

---

## 📚 Additional Resources

- 🔗 [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- 🔗 [Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- 🔗 [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
- 📝 [Server Development Guide](https://modelcontextprotocol.io/docs/concepts/servers)

---

*Module 3 of Mastering MCP*  
*© Velox AI Agency*