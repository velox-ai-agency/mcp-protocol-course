# Exercise 3.4: Build Your Own Prompt

## 🎯 Objective
Create a custom MCP prompt template for a specific use case.

---

## 📋 Instructions

Build a prompts server with at least **2 custom prompts**.

---

## 🎨 Prompt Ideas

Choose or create your own:

### Option A: Code Documentation
```
Name: document_code
Arguments:
  - code: The code to document
  - style: Documentation style (jsdoc, python-doc, simple)
```

### Option B: Code Review
```
Name: review_code
Arguments:
  - code: Code to review
  - language: Programming language
  - focus: What to focus on (security, performance, readability)
```

### Option C: Data Analysis
```
Name: analyze_data
Arguments:
  - data: CSV or JSON data
  - question: Specific question to answer
  - format: Output format (summary, detailed, chart)
```

### Option D: Your Own Idea
Create something useful for your workflow!

---

## 💻 Implementation Template

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  GetPromptRequestSchema,
  ListPromptsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-prompts", version: "1.0.0" },
  { capabilities: { prompts: {} } }
);

// Prompt 1: [Your prompt name]
server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [
      {
        name: "your_prompt_name",
        description: "What this prompt does",
        arguments: [
          {
            name: "arg1",
            description: "What this argument is",
            required: true
          },
          {
            name: "arg2",
            description: "Optional argument",
            required: false
          }
        ]
      },
      // Add your second prompt here
    ]
  };
});

server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "your_prompt_name") {
    return {
      description: "Prompt description",
      messages: [
        {
          role: "user",
          content: {
            type: "text",
            text: `Your prompt template here...
                   
Use ${args?.arg1} to include arguments.

Make it detailed and helpful!`
          }
        }
      ]
    };
  }

  throw new Error(`Unknown prompt: ${name}`);
});

const transport = new StdioServerTransport();
server.connect(transport);
```

---

## ✅ Requirements

Your prompts must:

1. **Have clear names** — Self-descriptive
2. **Accept arguments** — At least 1 required argument
3. **Use arguments dynamically** — Include them in the template
4. **Provide useful guidance** — Help the AI do a good job
5. **Include usage example** — Show how to call it

---

## 🧪 Testing

Test with MCP Inspector:

```bash
npx @modelcontextprotocol/inspector node dist/your-server.js
```

1. List your prompts: `prompts/list`
2. Request a prompt: `prompts/get` with arguments
3. Verify the output is useful

---

## 💡 Tips

- **Be specific**: Instead of "review code", say "security review for TypeScript code"
- **Provide context**: Give the AI enough information to do a good job
- **Structure output**: Ask for bullet points, sections, or specific formats
- **Test edge cases**: What happens if arguments are missing?

---

## 📋 Example Solution

### Prompt: SQL Query Generator

```typescript
{
  name: "generate_sql",
  description: "Generate a SQL query from natural language",
  arguments: [
    { name: "description", required: true },
    { name: "tables", required: true },
    { name: "dialect", required: false }
  ]
}

// Generated prompt:
"Please write a SQL query for: [description]

Available tables:
[tables]

SQL dialect: [dialect || 'PostgreSQL']

Requirements:
- Use proper JOIN syntax
- Include appropriate WHERE clauses
- Add comments for complex logic

Return only the SQL query."
```

---

## 🏆 Bonus Challenges

- [ ] Create 3+ prompts
- [ ] Include a prompt that references other resources
- [ ] Add argument validation
- [ ] Create a prompt that generates multiple messages

---

## 🔗 Next

Continue to [Lesson 3.5: Resources Server](../lesson-05-resources-server.md)
