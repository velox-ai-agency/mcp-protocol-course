# Lesson 3.4: Prompts Server — Templating for AI Assistants

## 🎯 Learning Objectives
By the end of this lesson, you will:
- Understand how MCP prompts work
- Create dynamic prompt templates with arguments
- Build a prompts-only MCP server
- Know when to use prompts vs tools

---

## 📋 What Are MCP Prompts?

**Prompts** in MCP allow servers to provide **reusable prompt templates** that:
- Accept dynamic arguments
- Can pull in context/resources
- Guide the AI's responses

Think of prompts as "smart templates" that the AI can use for consistent, structured interactions.

---

## 🔄 Prompts vs Tools: When to Use What?

| Feature | Tools | Prompts |
|---------|-------|---------|
| **Purpose** | Execute actions | Provide templates |
| **Result** | Data/response to user | Sending to AI model |
| **Arguments** | Yes | Yes |
| **Side Effects** | Can modify state | Read-only |
| **Example** | `read_file`, `calculate` | `analyze_code`, `summarize_document` |

**Rule of Thumb:**
- Use **Tools** when you want to *do something*
- Use **Prompts** when you want to *guide how the AI responds*

---

## 💻 Building a Prompts Server

### Step 1: Project Setup

```bash
mkdir mcp-prompts-server
cd mcp-prompts-server
npm init -y
npm install @modelcontextprotocol/sdk zod
```

### Step 2: Create the Server

Create `prompts-server.ts`:

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  GetPromptRequestSchema,
  ListPromptsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "prompts-server", version: "1.0.0" },
  { 
    capabilities: { 
      prompts: {}  // Enable prompts capability
    } 
  }
);

// Define available prompts
server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [
      {
        name: "code_review",
        description: "Review code for best practices and bugs",
        arguments: [
          {
            name: "language",
            description: "Programming language (e.g., typescript, python)",
            required: true
          },
          {
            name: "code",
            description: "Code to review",
            required: true
          }
        ]
      },
      {
        name: "explain_concept",
        description: "Explain a technical concept simply",
        arguments: [
          {
            name: "concept",
            description: "Concept to explain",
            required: true
          },
          {
            name: "audience",
            description: "Target audience (beginner, intermediate, expert)",
            required: false
          }
        ]
      },
      {
        name: "summarize_meeting",
        description: "Generate a structured meeting summary",
        arguments: [
          {
            name: "transcript",
            description: "Meeting transcript text",
            required: true
          }
        ]
      },
      {
        name: "debug_error",
        description: "Analyze and debug an error message",
        arguments: [
          {
            name: "error",
            description: "Error message or stack trace",
            required: true
          },
          {
            name: "context",
            description: "Additional context about the error",
            required: false
          }
        ]
      }
    ]
  };
});

// Handle prompt requests
server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "code_review": {
      const language = args?.language || "unknown";
      const code = args?.code || "";
      
      return {
        description: `Code review for ${language}`,
        messages: [
          {
            role: "user",
            content: {
              type: "text",
              text: `Please review the following ${language} code for:

1. **Bugs and Logic Errors** - Identify any obvious bugs
2. **Best Practices** - Check against ${language} conventions
3. **Security Issues** - Look for vulnerabilities
4. **Performance** - Note any inefficiencies
5. **Readability** - Suggest improvements

Here's the code:

\`\`\`${language}
${code}
\`\`\`

Provide specific, actionable feedback.`
            }
          }
        ]
      };
    }

    case "explain_concept": {
      const concept = args?.concept || "";
      const audience = args?.audience || "beginner";
      
      const audienceGuidance: Record<string, string> = {
        beginner: "Explain this as if to someone just starting out. Use simple analogies.",
        intermediate: "Assume some technical knowledge, but explain nuances clearly.",
        expert: "Provide a detailed, technical explanation with edge cases and implementation details."
      };
      
      return {
        description: `Explain ${concept}`,
        messages: [
          {
            role: "user",
            content: {
              type: "text",
              text: `Please explain the concept: "${concept}"

${audienceGuidance[audience] || audienceGuidance.beginner}

Structure your explanation:
1. **Definition** - What is it?
2. **How it works** - Core mechanics
3. **Why it matters** - Practical importance
4. **Example** - A concrete use case`
            }
          }
        ]
      };
    }

    case "summarize_meeting": {
      const transcript = args?.transcript || "";
      
      return {
        description: "Meeting summary",
        messages: [
          {
            role: "user",
            content: {
              type: "text",
              text: `Please summarize this meeting transcript:

${transcript}

Provide a structured summary with:

## Participants
[List who was present]

## Key Topics Discussed
[Main topics in order]

## Decisions Made
[Any decisions or agreements]

## Action Items
[Tasks assigned, with who owns them if mentioned]

## Next Steps`
            }
          }
        ]
      };
    }

    case "debug_error": {
      const error = args?.error || "";
      const context = args?.context || "";
      
      return {
        description: "Error analysis",
        messages: [
          {
            role: "user",
            content: {
              type: "text",
              text: `I need help debugging this error:

\`\`\`
${error}
\`\`\`

${context ? `Additional context:\n${context}` : ""}

Please:
1. **Identify the error type** - What kind of error is this?
2. **Root cause** - Why is this happening?
3. **Solution** - How do I fix it?
4. **Prevention** - How to avoid this in the future?`
            }
          }
        ]
      };
    }

    default:
      throw new Error(`Unknown prompt: ${name}`);
  }
});

// Start the server
const transport = new StdioServerTransport();
server.connect(transport);

console.error("Prompts MCP Server running...");
```

---

## 🧪 Testing with MCP Inspector

```bash
npx @modelcontextprompt/mcp/inspector node dist/prompts-server.js
```

**List prompts:**
```json
{
  "method": "prompts/list"
}
```

**Request a prompt:**
```json
{
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "language": "typescript",
      "code": "function add(a, b) { return a + b; }"
    }
  }
}
```

---

## 🔌 Configuration

```json
{
  "mcpServers": {
    "prompts": {
      "command": "node",
      "args": ["/path/to/mcp-prompts-server/dist/prompts-server.js"]
    }
  }
}
```

---

## 💡 Advanced: Resource References in Prompts

Prompts can reference resources:

```typescript
server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  if (request.params.name === "review_file") {
    return {
      description: "Review a specific file",
      messages: [
        {
          role: "user",
          content: {
            type: "resource",
            resource: {
              uri: "file:///path/to/review.md",
              mimeType: "text/markdown"
            }
          }
        },
        {
          role: "user",
          content: {
            type: "text",
            text: "Please review this document..."
          }
        }
      ]
    };
  }
});
```

---

## 📚 Summary

| Aspect | Details |
|--------|---------|
| **Capability** | `{ prompts: {} }` |
| **Discovery** | `prompts/list` |
| **Retrieval** | `prompts/get` with arguments |
| **Response** | `messages` array with role and content |
| **Dynamic** | Can generate based on arguments |

---

## ✅ Practice Exercise

Complete [Exercise 3.4: Build Your Own Prompt](../exercises/exercise-03-4-custom-prompt.md)
