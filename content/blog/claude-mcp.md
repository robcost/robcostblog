---
title: The Universal Connector for AI
date: 2026-04-17
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - Anthropic
  - Claude
  - MCP
excludeSearch: false
---

*Part 10 of a series on Claude API features for builders*

---

Throughout this series we've built up a picture of what's possible with the Claude API: deep reasoning, tool use, structured outputs, document understanding, citations, context management, cost optimisation, reusable Skills, and the Agent SDK. But there's one thread that's been running through several of these posts without getting its own spotlight: the protocol that lets AI connect to *everything else*.

The [Model Context Protocol](https://modelcontextprotocol.io) (MCP) is an open standard for connecting AI models to external tools, data sources, and services. It was introduced by Anthropic in November 2024, and in the roughly 18 months since, it's been adopted by OpenAI, Google DeepMind, Microsoft, and thousands of developers. It's now governed by the Linux Foundation under the Agentic AI Foundation. Jensen Huang called it a revolution. Sam Altman said people love it. And it's the plumbing underneath a lot of what we've already discussed.

If [Tool Use](/blog/claude-tool-use) taught Claude *how* to call functions, MCP standardises *what* those functions look like and how they're discovered, connected, and executed. It's the difference between building a custom cable for every device and using USB-C.

## The N×M problem

Before MCP, every AI application that needed to connect to an external service required a custom integration. If you had 10 AI applications and 20 tools, you potentially needed 200 different integrations. Each one had its own authentication flow, data format, error handling, and maintenance burden.

![The MCP architecture](/images/mcp_architecture.svg)

MCP turns this into an N+M problem. Each application implements the MCP client protocol once. Each tool implements the MCP server protocol once. Any client can connect to any server. Ten applications plus twenty servers equals thirty implementations, not two hundred.

This is the same pattern that made USB, HTTP, and the Language Server Protocol (LSP) successful. Standardise the interface, and the ecosystem compounds.

## How MCP works

The architecture has three components:

**MCP Hosts** are AI applications (like Claude Code, your custom agent, or an IDE plugin) that need to access external capabilities.

**MCP Clients** manage the connection between the host and MCP servers. In practice, the client is usually built into the host, like the SDK's built-in MCP support.

**MCP Servers** expose capabilities, primarily tools that the AI can call. A GitHub MCP server exposes tools like "create issue", "list PRs", and "search code". A database MCP server exposes "query" and "mutate". A Slack MCP server exposes "send message" and "read channel".

The protocol itself is JSON-RPC 2.0 over either stdio (for local servers) or HTTP with Server-Sent Events (for remote servers). The AI model discovers available tools by querying the server, then calls them when needed.

## MCP in the Claude API

You can connect to remote MCP servers directly from the Messages API using the [MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector), without building your own MCP client:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.beta.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 4096,
  betas: ["mcp-client-2025-11-20"],
  mcp_servers: [
    {
      type: "url",
      url: "https://mcp.notion.com/mcp",
      name: "notion",
    },
  ],
  messages: [
    {
      role: "user",
      content: "What are my open tasks in Notion?",
    },
  ],
});
```

Claude discovers the tools available on the Notion MCP server, calls the relevant ones, and incorporates the results into its response. You didn't need to define any tool schemas, build an execution loop, or implement the Notion API. The MCP server handled all of that.

## MCP in the Agent SDK

This is where MCP really shines, and where I've spent most of my time with it. In the Agent SDK, you create custom MCP servers for your domain-specific tools and pass them to the agent:

```typescript
import { createSdkMcpServer, tool } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const gameToolServer = createSdkMcpServer({
  name: "game-tools",
  tools: [
    tool(
      "get_design_document",
      "Read the current Game Design Document as JSON",
      { format: z.enum(["json", "summary"]).optional() },
      async ({ format }) => {
        const gdd = session.gdd;
        return {
          content: [{
            type: "text",
            text: format === "summary"
              ? summariseGDD(gdd)
              : JSON.stringify(gdd, null, 2),
          }],
        };
      }
    ),
    tool(
      "get_project_structure",
      "List the file tree of the game project",
      {},
      async () => ({
        content: [{
          type: "text",
          text: buildFileTree(session.projectPath),
        }],
      })
    ),
  ],
});
```

This is from [GameForge](https://github.com/robcost/gameforge). The game-tools MCP server gives agents the ability to read game design documents and inspect project structure. A separate playwright MCP server gives the QA agent the ability to navigate to the running game, take screenshots, press keys, and check console errors:

```typescript
const playwrightServer = createSdkMcpServer({
  name: "playwright",
  tools: [
    tool(
      "navigate_to_game",
      "Navigate the browser to the game preview URL",
      {},
      async () => { /* launch browser, go to game URL */ }
    ),
    tool(
      "take_screenshot",
      "Capture a screenshot of the current game state",
      { description: z.string().describe("What you expect to see") },
      async ({ description }) => { /* screenshot, return base64 */ }
    ),
    tool(
      "press_key",
      "Simulate a keyboard press in the game",
      { key: z.string(), duration: z.number().optional() },
      async ({ key, duration }) => { /* send key event */ }
    ),
  ],
});
```

Then you pass these servers to the agent:

```typescript
const agentQuery = query({
  prompt: "Test the game by playing through the first level",
  options: {
    mcpServers: {
      "game-tools": gameToolServer,
      "playwright": playwrightServer,
    },
    allowedTools: [
      "Read",
      "mcp__game-tools__get_design_document",
      "mcp__playwright__navigate_to_game",
      "mcp__playwright__take_screenshot",
      "mcp__playwright__press_key",
    ],
  },
});
```

The naming convention for MCP tools is `mcp__{server-name}__{tool-name}`. You can grant access to specific tools from specific servers through the `allowedTools` list, giving you fine-grained control over what each agent can do.

## The ecosystem

The real power of MCP isn't in building your own servers (though that's valuable). It's in the ecosystem that already exists. Companies like Notion, Slack, GitHub, Google, and many others have shipped MCP servers. The community has built thousands more. When you connect to an MCP server, you get instant access to a set of well-tested tools without writing integration code.

As of early 2026, there are over 5,800 MCP servers available, with SDKs in TypeScript, Python, C#, Java, and Ruby. The protocol is governed by the Linux Foundation, ensuring vendor neutrality.

## Where MCP fits in this series

Looking back across all ten posts, MCP is the connective tissue:

- **Tool Use** ([Part 2](/blog/claude-tool-use)): MCP standardises the tool interface that Claude calls
- **Agent Skills** ([Part 8](/blog/claude-agent-skills)): Skills and MCP complement each other, Skills for domain expertise, MCP for external service access
- **Agent SDK** ([Part 9](/blog/claude-agent-sdk)): The SDK uses MCP servers as its extension mechanism for custom tools
- **The broader ecosystem**: MCP means your AI application can connect to any MCP-compatible service without custom code

In GameForge, we use four custom MCP servers: game-tools (design document access), playwright (browser automation for QA), asset-tools (AI image generation), and music-tools (AI music generation). Each server encapsulates a domain, each agent gets access to only the servers it needs, and the Agent SDK handles all the protocol mechanics.

## When to build your own MCP server vs using existing ones

**Use existing servers when:** The service you need already has an MCP server (GitHub, Slack, Notion, Google Drive, databases). Check the [MCP server registry](https://modelcontextprotocol.io) and the community repositories.

**Build your own when:** You need domain-specific tools that don't exist yet. Your internal APIs, business logic, proprietary data sources, custom workflows. The `createSdkMcpServer()` helper in the Agent SDK and the `@modelcontextprotocol/sdk` package both make this straightforward.

**Consider both when:** You need a mix of standard integrations and custom logic. GameForge uses custom servers for game-specific tools but could easily add a GitHub MCP server for version control integration.

## Trade-offs

**Token cost of tool definitions.** Every MCP server you connect adds its tool definitions to the context. A five-server setup can easily consume 55,000+ tokens before Claude reads a single message. This is why [Tool Search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool) (from Part 2) and [Programmatic Tool Calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) exist, to manage this at scale.

**Security.** MCP servers can execute arbitrary code and access external services. The protocol doesn't enforce authentication or sandboxing at the protocol level, those are implementation concerns. Audit any server before connecting it to your agent, especially community-built ones.

**Maturity.** The protocol is evolving. The spec got major updates in November 2025 (async operations, server identity, statelessness), and the ecosystem is still standardising around authentication patterns. It works well today, but expect things to keep changing.

## The series, complete

This post wraps up the full series. Here's what we've covered:

1. [**Extended Thinking**](/blog/claude-extended-thinking): Deep reasoning for complex problems
2. [**Tool Use**](/blog/claude-tool-use): Calling functions and taking actions
3. [**Structured Outputs**](/blog/claude-structured-outputs): Guaranteed schema-conformant responses
4. [**Vision & PDF**](/blog/claude-vision): Understanding images and documents
5. [**Citations**](/blog/claude-citations): Verifiable source attribution
6. [**Context Windows**](/blog/claude-context-windows): Managing AI working memory at scale
7. [**Batch Processing & Prompt Caching**](/blog/claude-batch-caching): Cost and performance optimisation
8. [**Agent Skills**](/blog/claude-agent-skills): Reusable packages of domain expertise
9. [**Agent SDK**](/blog/claude-agent-sdk): The production agent runtime
10. **MCP** (this post): The universal protocol for AI integrations

Together, these ten features form a complete stack for building AI-powered systems that can think, act, see, prove their work, manage their resources, scale affordably, learn domain expertise, operate autonomously, and connect to everything.

The pace at which this stack has evolved is genuinely remarkable. Most of what's in this series didn't exist two years ago. Some of it didn't exist six months ago. If you're building with AI, there's never been a better time to start, and there's never been a more complete set of tools to work with.

---

*Thanks for following along through all ten posts. This started as a LinkedIn article idea and grew into something much bigger. If any of it helped you build something, or if you've got questions, I'd genuinely love to hear about it. You can find me at [robcost.com](https://robcost.com) or reach out on [LinkedIn](https://www.linkedin.com/in/robcostello/).*

---

### References & further reading

- [Model Context Protocol specification](https://modelcontextprotocol.io), The official protocol documentation, SDKs, and server registry
- [MCP connector, Claude API docs](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector), Connecting to remote MCP servers from the Messages API
- [Introducing MCP (Anthropic)](https://www.anthropic.com/news/model-context-protocol), Anthropic's original announcement post
- [Code execution with MCP (Anthropic engineering)](https://www.anthropic.com/engineering/code-execution-with-mcp), How code execution solves MCP's token scaling problem
- [A Year of MCP (Pento)](https://www.pento.ai/blog/a-year-of-mcp-2025-review), Excellent retrospective on MCP's first year with practical insights
- [GameForge source code](https://github.com/robcost/gameforge), Multi-agent platform using four custom MCP servers
- [MCP on Wikipedia](https://en.wikipedia.org/wiki/Model_Context_Protocol), Good overview of the protocol's history and adoption
