---
title: What Happens When You Let AI Think Before It Speaks?
date: 2026-03-16
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - Anthropic
  - Claude
  - Thinking
excludeSearch: false
images:
  - /images/extended_thinking_modes.svg
---

*Part 1 of a series on Claude API features for builders*

---

We're all inundated with AI-related content these days, but for me I seem to see more *AI=bad* than anything else online. It's all jailbreak-this, benchmark-fail-that, incoming apocalypse, and risk risk risk! But some of it **will** be good. So I'm planning to write some *AI=good* content, and hopefully give some of you good reasons to explore bringing AI technologies into what you're building. I've been working with the [Anthropic Claude APIs](https://platform.claude.com/docs/en/build-with-claude/overview) a lot lately, so I'm going to focus on some of its features that I think are genuinely useful, interesting, and worth understanding properly.

This first post covers Extended Thinking, what it is, why it matters, and how to actually use it in code. By the end you'll have runnable TypeScript examples and a sense of when it's the right tool for the job.

---

## The problem with fast answers

I spent a very long time in enterprise tech, watching organisations buy software and hardware they barely understood or needed, based on promises made by people who'd learned to say "digital transformation" with a serious face. Wave after wave of new technologies has come and gone, but I'm a believer that this AI wave is fundamentally different, and we're seeing it evolve more rapidly than any other wave before.

Anthropic, and others (Open Source included), have been leading the charge in advancing capabilities at a rapid rate. But there's a fundamental limitation in how most people interact with language models today that's worth understanding before we talk about the solution.

Most interactions people are used to having with language models are essentially reactive: you send a message (the prompt), and the model generates a response token by token, left to right, committing to each word as it goes. It's like asking someone to solve a crossword by filling in every square in order, top-left to bottom-right, with no erasing. Sure you could get it right, but having a little space to think could give you a better chance of success.

For simple tasks like drafting an email, summarising a document, or answering a factual question, that works brilliantly, and those are the types of features we saw integrated into our day-to-day apps first.

But if you throw a genuinely complex question at a language model, a business decision that has multiple variables, a financial model with competing constraints, a legal clause in a contract that interacts with three other clauses, this is where you start to see the cracks. The model commits early. It goes down a path and can't back up. It gives you an answer that *sounds* authoritative but hasn't actually been "thought" through.

## So what is Extended Thinking?

Extended Thinking is Anthropic's way of giving Claude a digital scratchpad. Others in the AI world might refer to it as **Reasoning** or **Test Time Compute (TTC),** the idea that you can make an AI respond better not just by training it on more data (which happens once, upfront, and costs billions), but by giving it more compute *at the moment it's answering your question*. Anthropic themselves describe it as Claude benefiting from ["serial test-time compute"](https://www.anthropic.com/news/visible-extended-thinking), using multiple sequential reasoning steps before producing its final output.

I don't want to anthropomorphise, yes it's not like a human thinking, but it does allow AI systems to consider and move in hopefully more accurate directions. When you enable it through the API, Claude doesn't just start generating its response immediately. Instead, it first works through the problem internally, reasoning step by step, considering alternatives, checking its own logic, before committing to a final answer. It's still generating tokens the same way, but we're allowing it to "think" before giving its final answer.

In practical terms, the API response comes back in two parts: a `thinking` block (where Claude shows its working) and then the actual `text` response. You get to see *how* it arrived at the answer, not just what the answer is. Here's what the response structure looks like:

```json
{
  "content": [
    {
      "type": "thinking",
      "thinking": "Let me analyse this step by step...",
      "signature": "WaUjzkypQ2mUEVM36O2..."
    },
    {
      "type": "text",
      "text": "Based on my analysis..."
    }
  ]
}
```

The `signature` field is worth a quick mention. In Claude 4 models, the thinking you see is actually a *summary* of Claude's full internal reasoning (Anthropic calls this "summarised thinking"). The full thinking is encrypted in that signature field, and you're charged for the full thinking tokens, not the summary. This is a safety measure, but it means the billed output token count won't exactly match what you see in the response. Something to be aware of when you're tracking costs.

The mental model I and others align it to is the difference between System 1 and System 2 thinking, Daniel Kahneman's framework from *Thinking, Fast and Slow*. Standard Claude is System 1: fast, intuitive, generating an answer in one pass. Extended Thinking is System 2: deliberate, analytical, slower, but much better when the stakes are high or the problem is genuinely hard.

![Standard response vs Extended Thinking](/images/extended_thinking_modes.svg)

## The three modes of thinking

Here's where it gets interesting. Anthropic doesn't just offer one flavour of thinking. There are actually three modes you can use, and choosing the right one matters.

### Mode 1: Manual Extended Thinking

This is the original implementation. You explicitly enable thinking and set a token budget, the maximum number of tokens Claude can use for its internal reasoning. Claude uses as much of that budget as it needs, sometimes a few hundred tokens for a moderately tricky question, sometimes most of the budget for something genuinely hard. It doesn't always use everything you allocate, which is a nice touch.

Here's what it looks like in TypeScript:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 16000,
  thinking: {
    type: "enabled",
    budget_tokens: 10000,
  },
  messages: [
    {
      role: "user",
      content:
        "A company has $2M to invest across three projects. Project A has 40% ROI but requires at least $500K. Project B has 25% ROI with no minimum. Project C has 60% ROI but has a 30% chance of total failure. How should they allocate?",
    },
  ],
});

// Process the response
for (const block of response.content) {
  if (block.type === "thinking") {
    console.log("=== Claude's Reasoning ===");
    console.log(block.thinking);
  }
  if (block.type === "text") {
    console.log("\n=== Final Answer ===");
    console.log(block.text);
  }
}
```

A couple of things to note. The `budget_tokens` must be less than `max_tokens`. The minimum budget is 1,024 tokens. And for thinking budgets above 32K tokens, Anthropic recommends using [batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing) to avoid network timeout issues, as these requests can take a while.

**When to use manual mode:** When you need precise control over thinking token spend, for example if you're building a system where cost predictability per request matters. Also the only option if you're using older models (Sonnet 4.5, Opus 4.5, etc.).

### Mode 2: Adaptive Thinking

This is the newer, recommended approach for the latest models (Opus 4.6 and Sonnet 4.6). Instead of you setting a fixed budget, Claude decides for itself whether the question warrants deep thinking or whether a quick response will do. For simple questions it just answers directly. For complex ones, it spins up the reasoning engine.

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 16000,
  thinking: {
    type: "adaptive",
  },
  messages: [
    {
      role: "user",
      content:
        "A company has $2M to invest across three projects. Project A has 40% ROI but requires at least $500K. Project B has 25% ROI with no minimum. Project C has 60% ROI but has a 30% chance of total failure. How should they allocate?",
    },
  ],
});
```

The API shape is almost identical, you just swap `"enabled"` for `"adaptive"` and drop the `budget_tokens`. But the behaviour difference is meaningful. Adaptive thinking also automatically enables *interleaved thinking*, which means Claude can think between tool calls in agentic workflows, not just at the start. That's a big deal if you're building anything that chains multiple steps together.

You can also combine adaptive thinking with the `effort` parameter to give Claude a general sense of how hard to think:

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 16000,
  thinking: {
    type: "adaptive",
  },
  output_config: {
    effort: "medium", // "low" | "medium" | "high" | "max"
  },
  messages: [
    {
      role: "user",
      content: "What is the capital of France?",
    },
  ],
});
```

At `high` effort (the default), Claude will almost always think. At `low`, it'll skip thinking for simple tasks where speed matters most. At `max` (Opus 4.6 only), it thinks with no constraints on depth.

This is a meaningful design choice. It means you can leave thinking enabled across your entire application without worrying that every "what time is it in Tokyo?" query is burning through a massive reasoning session. (Timezones are my Achilles heel, even after working in a global role for many years! I'd burn through 10,000 braincells doing the calculation 😂)

**When to use adaptive mode:** For most new projects on the latest models. It's simpler, often performs better, and handles the "should Claude think about this?" question for you. Particularly good for agentic workflows where thinking between tool calls matters.

### Mode 3: Thinking Disabled

This is the default if you don't specify a `thinking` parameter at all. Standard Claude behaviour, fast and intuitive. No reasoning trace, no thinking tokens, lowest latency and cost.

**When to use disabled mode:** High-volume, low-complexity tasks. Classification, simple extraction, conversational responses where speed is everything. You don't need your Claude-based AI system to engage in deep reasoning about whether to classify an email as "urgent" or "normal".

### The comparison at a glance

| Mode | Config | Best For | Model Support |
|------|--------|----------|---------------|
| **Adaptive** | `thinking: { type: "adaptive" }` | Most use cases on latest models. Agentic workflows. | Opus 4.6, Sonnet 4.6 |
| **Manual** | `thinking: { type: "enabled", budget_tokens: N }` | Precise cost control. Older models. | All thinking-capable models |
| **Disabled** | Omit `thinking` parameter | Speed-critical, simple tasks | All models |

## A worked example: investment analysis

Let me show you a more complete example that demonstrates the practical difference. We'll ask Claude to analyse an investment decision with and without thinking, and compare the outputs.

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const investmentPrompt = `
You are a financial analyst. A mid-size manufacturing company needs to decide 
between three strategic options:

1. Invest $5M in automation (reduces labour costs by 30%, 18-month payback, 
   but requires laying off 200 workers in a tight labour market)
2. Invest $3M in a new product line (projected 45% margin, but entering a 
   market with 3 established competitors and requires hiring 50 specialists)
3. Invest $2M in sustainability upgrades (15% energy cost reduction, qualifies 
   for $1M government grant, improves ESG rating, but 36-month payback)

They can choose one or combine options within a $6M total budget.
What should they do and why?
`;

// Without thinking
const standardResponse = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 4096,
  messages: [{ role: "user", content: investmentPrompt }],
});

console.log("=== Without Extended Thinking ===");
for (const block of standardResponse.content) {
  if (block.type === "text") console.log(block.text);
}

console.log("\n\n");

// With adaptive thinking
const thinkingResponse = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 16000,
  thinking: {
    type: "adaptive",
  },
  messages: [{ role: "user", content: investmentPrompt }],
});

console.log("=== With Extended Thinking ===");
for (const block of thinkingResponse.content) {
  if (block.type === "thinking") {
    console.log("--- Reasoning Trace ---");
    console.log(block.thinking);
    console.log("--- End Reasoning ---\n");
  }
  if (block.type === "text") {
    console.log(block.text);
  }
}

// Compare token usage
console.log("\n=== Token Usage ===");
console.log("Standard:", JSON.stringify(standardResponse.usage));
console.log("Thinking:", JSON.stringify(thinkingResponse.usage));
```

When I run something like this, the difference is striking. The standard response tends to give a reasonable-sounding recommendation, usually picking the "best" single option. The thinking response works through the interactions between options, considers the budget constraint combinatorics, weighs second-order effects (like how the ESG rating improvement might affect the company's ability to hire those 50 specialists), and often arrives at a combined strategy that the standard mode doesn't even consider.

The reasoning trace is the real gold here. You can see Claude weighing up options, catching its own initial assumptions, and course-correcting. For any application where you need to trust or audit the AI's reasoning, that visibility is invaluable.

## Streaming thinking responses

For real applications, you'll probably want to stream the response rather than waiting for the entire thing to complete, especially since thinking can take a while on complex prompts. Here's how:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const stream = await client.messages.stream({
  model: "claude-sonnet-4-6",
  max_tokens: 16000,
  thinking: {
    type: "adaptive",
  },
  messages: [
    {
      role: "user",
      content: "What are the second-order effects of a universal basic income?",
    },
  ],
});

for await (const event of stream) {
  if (event.type === "content_block_start") {
    if (event.content_block.type === "thinking") {
      process.stdout.write("\n🧠 Thinking: ");
    } else if (event.content_block.type === "text") {
      process.stdout.write("\n💬 Response: ");
    }
  } else if (event.type === "content_block_delta") {
    if (event.delta.type === "thinking_delta") {
      process.stdout.write(event.delta.thinking);
    } else if (event.delta.type === "text_delta") {
      process.stdout.write(event.delta.text);
    }
  }
}

const finalMessage = await stream.finalMessage();
console.log("\n\nTokens used:", finalMessage.usage);
```

One thing to be aware of with streaming: the thinking content can arrive in larger, "chunkier" batches rather than the smooth token-by-token delivery you get with regular text responses.

## Why should you care?

You're reading this and thinking: *Interesting technically, Rob, but why does this actually matter?*

Three reasons.

**First, it dramatically improves accuracy on complex tasks.** Anthropic's own benchmarks tell the story clearly. When Extended Thinking was first introduced with [Claude 3.7 Sonnet](https://www.anthropic.com/news/claude-3-7-sonnet), performance on competitive maths problems (AIME 2024) jumped from 61.3% to 80.0%. Graduate-level science reasoning (GPQA Diamond) went from 78.2% to 84.8%. Coding task accuracy on SWE-bench rose from 62.3% to 70.3%. Anthropic's own research showed that [maths accuracy improves logarithmically](https://www.anthropic.com/news/visible-extended-thinking) with the number of thinking tokens allocated, meaning more thinking budget equals measurably better answers on hard problems. But the benchmarks almost undersell it. Where I've seen the biggest difference is in tasks that require *holding multiple constraints in mind simultaneously*, exactly the kind of work that matters in business.

Think: analysing a contract where clause 4.2 interacts with the indemnification in section 7 and the termination rights in section 12. Or evaluating a market entry strategy where regulatory, competitive, and operational factors all pull in different directions. These are problems where a fast, intuitive answer is often a wrong answer.

**Second, the thinking is visible, and that changes trust dynamics.** One of the biggest objections I hear about using AI for anything consequential is: "I don't know how it got there, it's a magic black box spitting out probably next words." Extended Thinking gives you a reasoning trace. You can see Claude consider option A, weigh it against option B, identify a flaw in its initial reasoning, and course-correct. Anthropic themselves [cite trust as a primary motivation](https://www.anthropic.com/news/visible-extended-thinking) for making the thinking visible: being able to observe how Claude reasons makes it easier to understand and verify its answers. If it gets things wrong anyway, you know where its logic misstepped and can provide corrective context.

That's not just a nice-to-have. In regulated industries like financial services, healthcare, and legal, being able to demonstrate *how* a recommendation was reached isn't optional. It's a compliance requirement. Extended Thinking doesn't solve the whole explainability problem, but it moves the needle meaningfully.

**Third, it enables a new category of AI application.** Without extended thinking, most AI applications are essentially sophisticated autocomplete: summarisation, classification, extraction, generation. All valuable, but all relatively straightforward cognitive tasks. Retrieval Augmented Generation (RAG) will help, but just adding more data to the context window won't allow the model to reason fully about the data you're giving it. Combine RAG with Extended Thinking, and you're super-charging the model's ability to respond thoughtfully.

With extended thinking, you can start to build applications that do genuine *analysis*. Decision support systems that actually weigh trade-offs. Quality assurance workflows that reason about edge cases. Planning tools that consider second-order effects. That's a different class of product entirely.

## Where I've seen it make a real difference

I'll give you a concrete example from my own work. I'm building an AI tutoring platform for kids ([myEdi.ai](https://myedi.ai)), and one of the hardest problems is *adaptive difficulty*. When a 12-year-old gets a maths question wrong, the system needs to figure out *why*, was it a careless arithmetic error, a conceptual misunderstanding, or did they just not know the relevant formula?

Without extended thinking, the model tends to pattern-match to the most common error type and move on. With it, I can watch Claude reason through the student's specific answer, trace back the likely misconception, and generate a follow-up question that targets the actual gap. The difference in pedagogical quality is significant.

Scale that pattern up to enterprise use cases, diagnosing why a supply chain disruption cascaded the way it did, figuring out why a customer churn model is underperforming in one segment, or working through the implications of a regulatory change across multiple business units, and you start to see the leverage.

## The trade-offs

Extended Thinking uses more tokens and takes more time, meaning it costs more money per request. Thinking tokens are charged at the same rate as output tokens, so on Sonnet 4.6 that's $15 per million tokens for the thinking, on top of whatever the actual response costs. For high-volume, low-complexity tasks, it's overkill.

There's also a latency consideration. If you're building a real-time chat interface, adding several seconds (or even minutes for complex problems with large thinking budgets) of thinking time changes the user experience. That might be perfectly acceptable for a financial analysis tool (where users expect to wait for quality) but problematic for a customer support bot (where speed is everything). You don't want it to stop and think for 30 seconds about how to answer a user's question on your company website's chatbot.

And remember the summarised thinking point from earlier: on Claude 4 models, you're charged for the *full* thinking tokens, not the summary you see in the response. So your billing will be higher than what the visible token count suggests. Keep an eye on the `usage` object in the API response to track actual costs.

The art is in knowing *when* to turn it on. And Adaptive Thinking helps here, it lets you keep the capability available for when it's really needed, without paying the cost on every interaction.

## Practical tips from actual usage

A few things I've learned from building with this:

**Start with adaptive thinking on the latest model.** Unless you have a specific reason to control the budget precisely, let Claude decide when to think. Anthropic's own [internal evaluations](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking) show adaptive thinking reliably drives better performance than manual extended thinking for most workloads.

**Use the effort parameter liberally.** If you have a mix of simple and complex tasks hitting the same endpoint, set effort to `medium` and let Claude sort it out. For your hardest problems, crank it to `high` or `max`.

**Don't forget about streaming.** Thinking can take a while. If you're showing results to a user, stream the response so they see progress rather than staring at a spinner.

**Watch your token usage.** It's easy to burn through tokens when thinking is enabled. The `usage` object in the response tells you exactly what you used. Build monitoring around it early.

**Prompting matters differently with thinking enabled.** Anthropic's [prompting tips](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/extended-thinking-tips) note that extended thinking often works better with a general instruction to "think deeply" rather than step-by-step instructions. If you over-constrain the reasoning, you can actually reduce quality. Let the thinking mode do its thing.

## What's next

If you're a product leader or a technical decision-maker evaluating AI capabilities, here's the takeaway: Extended Thinking expands the boundary of what's worth automating or augmenting with AI. Tasks you previously ruled out because they were "too complex" or "too nuanced" for a language model are worth revisiting. Not all of them will work, but more of them will than you might expect.

It also raises the bar on what "good enough" looks like. Once your competitors' AI tools can actually reason through complex problems instead of pattern-matching their way to plausible-sounding answers, "we have an AI chatbot" stops being a differentiator.

And perhaps most importantly, it shifts the conversation from "can AI do this?" to "should we let AI do this?", which is a much more interesting (and honestly, more important) question.

---

*Next in this series: I'll explore Tool Use, how Claude can reach out into the real world, call APIs, search the web, and execute code, turning it from a conversational partner into something that can actually take action.*

*If you're building with this stuff or just curious about it, I'd love to hear what you think. You can find me at [robcost.com](https://robcost.com) or reach out on [LinkedIn](https://www.linkedin.com/in/robcost/).*

---

### References & further reading

- [Anthropic: Claude's Extended Thinking](https://www.anthropic.com/news/visible-extended-thinking), Anthropic's research blog on the thinking capability, including the test-time compute framing and benchmark analysis
- [Anthropic: Claude 3.7 Sonnet announcement](https://www.anthropic.com/news/claude-3-7-sonnet), The original announcement where extended thinking was introduced
- [Extended Thinking API docs](https://platform.claude.com/docs/en/build-with-claude/extended-thinking), Full API reference with examples in multiple languages
- [Adaptive Thinking API docs](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking), Documentation for the newer adaptive thinking mode
- [Extended Thinking prompting tips](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/extended-thinking-tips), Anthropic's guide to getting the best results with thinking enabled
- [Kahneman, D. (2011). *Thinking, Fast and Slow*](https://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow), The System 1 / System 2 framework referenced in this post
