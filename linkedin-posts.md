
# Extended Thinking - 18 March 9am AEST
There's so much discourse over AI right now, my social feeds are full of opposing views. Plenty of "AI is going to replace everything" takes. Plenty of "AI is garbage" takes. Plenty of clever memes. Plenty of slop. I want to see more from people actually building things, and since I've been building with this stuff for quite some time now I thought I would also start contributing. I'm building, I'm making mistakes, I'm learning just like everyone else.

So I started a 10-part series breaking down Anthropic Claude API features I use in production. Just what works, what doesn't, and code you can run and learn from, minus the hype and blah-blah. Part 1 is Extended Thinking, the feature that gives Claude space to reason before it responds. 

I walk through how it works, when to use it, and include TypeScript examples you can learn from.

Link in comments...


https://robcost.com/blog/claude-extended-thinking/

---

# Tool Use - 21 March 9am AEST
Part 2 of my Claude API series is up. This one's about Tool Use,  the feature that turns Claude from a conversation partner into something that can actually do things. Call your APIs, query databases, search the web, all within a single interaction.

This is where AI stops giving advice and starts taking action. I walk through client tools vs server tools, the agentic loop pattern, and how to combine tools with thinking for genuinely useful multi-step workflows.

TypeScript examples included, as always.

Link in first comment...


https://robcost.com/blog/claude-tool-use/

---

# Structured Outputs - 25 March 9am AEST
Part 3. This one matters more than people realise.

Ask Claude to return JSON and it usually will. "Usually" is fine for a demo. It's not fine when that JSON feeds into your CRM, your billing system, or anything a human depends on. One malformed response and your pipeline breaks at 3am.

Structured Outputs guarantees the output matches your schema. Not "tries to." Guarantees. I break down how constrained decoding works, when to use JSON mode vs strict tool use, and the Zod integration that makes it feel native in TypeScript.

This is the feature that closes the gap between prototype and production.

Link in first comment...


https://robcost.com/blog/claude-structured-outputs/

---

# Vision & PDF - 28 March 9am AEST
Part 4. We're giving Claude eyes.

So much business information lives in formats that were never designed for machines,  scanned contracts, exported dashboards, photographed whiteboards, PDFs nobody has time to read. Vision lets Claude actually understand that stuff, not just the text in it, but the charts, layouts, and visual context.

I cover how to send images via the API, process PDFs, combine vision with structured outputs for data extraction, and where it genuinely falls short (because it does).

Link in first comment...


https://robcost.com/blog/claude-vision/

---

# Citations - 1 April 9am AEST
Part 5. The trust problem.

Every conversation I have about putting AI into production hits the same question: "how do I know it's not making things up?" Fair question. Confident, articulate, entirely wrong answers are a real thing with language models.

Citations is how Claude shows its working. When you enable it, responses come back with pointers to exact passages in your source documents. Not free-form references it generated,  actual verifiable links to specific text, page numbers, and character ranges.

I walk through how it works with plain text, PDFs, and search results. This is the feature that makes document-heavy AI workflows viable.

Link in first comment...


https://robcost.com/blog/claude-citations/

---

# Context Windows - 4 April 9am AEST
Part 6. The one about memory.

Every feature in this series,  thinking, tools, documents, citations,  they all consume tokens. And tokens live inside the context window. Understanding how it works is the difference between a demo and a production system.

Claude now has a 1 million token context window. That's roughly 8 novels. But "big" doesn't mean "infinite," and knowing what to put in context (and what to leave out) matters more than most people think.

I cover how the window fills up, what compaction does when it overflows, and practical strategies for managing it.

Link in first comment...


https://robcost.com/blog/claude-context-windows/

---

# Batch Processing & Prompt Caching - 8 April 9am AEST
Part 7 of the Claude API series. The unsexy one.

Nobody's going to demo batch processing on stage. But it's the feature that makes everything else economically viable. 50% off all token costs, same models, same quality,  you just give up immediate responses.

Prompt caching is the other half: up to 90% savings when you're reusing the same context across requests. I break down how both work, when to use them, what invalidates the cache, and include a real cost comparison that shows 70%+ savings on a document processing workload.

Not glamorous. Genuinely useful.

Link in first comment...


https://robcost.com/blog/claude-batch-caching/

---

# Agent Skills - 11 April 9am AEST
Part 8. Teaching Claude actual expertise.

Tools let Claude do things. Skills let Claude do things *well*. The difference is like giving someone a hammer vs teaching them carpentry.

A Skill is basically a folder with instructions and reference materials that Claude loads on demand. Instead of cramming thousands of tokens of domain knowledge into every system prompt, you package it up and the agent grabs what it needs for the specific task.

I use examples from GameForge, a multi-agent game creation platform I've been building to learn the ins and outs of the Claude API. Skills turned out to be one of the most impactful features in the whole system.

Link in first comment...


https://robcost.com/blog/claude-agent-skills/

---

# Agent SDK - 15 April 9am AEST
Part 9. This is where it clicks.

If you've been following this series, you've been building with the raw Messages API,  managing tool loops, handling stop reasons, routing tool results. It works, but the plumbing adds up.

The Agent SDK eliminates that. It's the same engine that powers Claude Code, exposed as a library. You get 15+ built-in tools, context management, automatic compaction, and streaming,  out of the box. Your job becomes defining what your agent does, not how the agent works.

I walk through query(), built-in tools, custom MCP servers, and when to use the SDK vs the raw API (because it's not always the right choice).

Link in first comment...


https://robcost.com/blog/claude-agent-sdk/

---

# MCP (Model Context Protocol) - 18 April 9am AEST
Part 10. The final one.

If Tool Use taught Claude how to call functions, MCP standardises what those functions look like. It's an open protocol for connecting AI to external services, adopted by OpenAI, Google, Microsoft, and thousands of developers since Anthropic introduced it. It might not be the forward, direct API calls and CLI usage from LLMs is getting better, but it has filled a gap and might still be part of the longer term picture.

Before MCP, connecting 10 AI apps to 20 tools meant up to 200 custom integrations. MCP turns that into 30. Same pattern that made USB and HTTP successful, standardise the interface and the ecosystem compounds.

This wraps up the series. Ten features that form a complete stack for building AI systems that can think, act, see, prove their work, scale affordably, and connect to everything.

Thanks for following along. If any of it was useful, I'd love to hear about it.

Link in first comment...


https://robcost.com/blog/claude-mcp/

