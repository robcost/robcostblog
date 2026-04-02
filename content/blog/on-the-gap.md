---
title: On the Gap
date: 2026-04-06
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - Startup
  - Builder
  - Professional Development
excludeSearch: false
---

There were two moments that sent me down the startup founder path, and neither happened at my desk.

The first one I was sitting on a tram going to work one day, watching tech Twitter posts, and one was a demo video from Google showing an early version of Gemini generating an app's UI on the fly. A user typed into a text box and the system generated Flutter code in real time, rendering cards, buttons, images, the whole interface, dynamically based on what the user asked for. It wasn't a chatbot. It wasn't a static screen. It was an application that built itself around the user's intent.

The second was a night in Geneva. I was sitting in my car outside my son's basketball practice, and I decided to try ChatGPT's voice mode, which had just been released. I ended up having a 20-minute conversation about quantum physics with an AI, a very sureal experience the first time you do it. Not reading text on a screen. Talking. It talked back. I sat in that car park for a while after it ended, thinking about what I'd just experienced.

![Woah](https://media3.giphy.com/media/v1.Y2lkPTc5MGI3NjExbDF4dTJ1NWxuMDU2amh0em1pNXZndmtjM2NwNm9vbGxmZ2h0dG12bSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/uPnKU86sFa2fm/giphy.gif)

Those two moments convinced me to start building an AI-native product. A few months into that journey, I realised something: every founder building with AI right now has had their own version of the car park moment. The moment where the technology blows your mind and you think "I need to build something with this." The demo that makes the future feel obvious.

That car park is a long way from a shipped product.

## The gap

Some days you feel ready to release. Then you're down a rabbit hole for a week fixing something you hadn't considered. You surface, feel ready again, and the cycle repeats.

The distance between "woah, this is amazing" and "this is reliable enough to ship" is much, much larger than it looks from the woah side. I've lived it myself building myEdi, an AI tutoring platform for kids, and I've watched other founders hit the same walls. The patterns are remarkably consistent.

## Founders fall in love with the model

This is the first trap, and almost everyone falls into it. The demo is so good that the model becomes the product in your head. You start thinking about model selection in terms of benchmarks and leaderboards, which one is "best", as if that's a meaningful question without context.

It's not. I learned this the hard way. I used one model that was genuinely good at teaching, it would ask questions instead of giving answers, deliver information in small chunks, check understanding before moving on. When I had to switch providers for compliance reasons, the replacement model was more fluent, more confident, technically more capable on paper. But it just wanted to talk. It would explain fractions to a 10 year old in great detail when what it should have been doing was helping the kid figure it out for themselves.

Neither model was "better." One was better for teaching. *Model selection isn't a leaderboard exercise, it's a product design decision.* Treating it any other way leads to something that demos well but doesn't actually do what you built it to do.

## The AI was never the expensive part

Every conversation about AI startups eventually turns to cost. Can you afford the inference? How do you manage token spend at scale? Will the unit economics work?

I spent a lot of time worrying about this early on. Turns out it was the wrong thing to worry about. From my initial cost model 18 months ago to now, the model calls are some of the smallest line items in the whole solution. The databases, the auth, the storage, the infrastructure that wraps around the AI, that's where the real money goes.

The conventional wisdom that "AI is expensive to run at scale" doesn't match what I've seen. Everything *around* the AI is the expensive bit. And more importantly, the non-technical costs are the ones that truly blindside you. Company setup. Legal structure. Terms of service. Privacy policies, especially if you're dealing with children or sensitive data. Tax obligations across multiple countries. Compliance frameworks you've never heard of until a lawyer tells you about them.

None of this appeared in any "how to build an AI startup" guide I've read. And none of it is natural for a technologist.

## Platform risk is a business problem, not a tech problem

I had my entire product built on one provider's API when they changed their terms of service and excluded under-18s from features I needed. Not a technical failure. Not a performance issue. A policy change that invalidated months of work overnight.

This isn't unique to me. Every startup building on a single model provider's API is one policy change, one pricing adjustment, or one deprecation notice away from the same situation. The founders who think about this early, who design for portability, who treat model providers as interchangeable components rather than foundations, are the ones who survive when the ground shifts. Most don't think about it until it happens.

## Evaluation is the unsolved problem

If there's one thing I'd tell any AI founder to think about earlier, it's this: how will you know your product is actually working?

Not in the demo sense. In the "real users depending on this every day" sense. Most evaluation tooling was built for a different kind of AI application. Evaluating whether a chatbot answered a support ticket correctly is relatively straightforward. You have an expected answer, you compare. But evaluating whether an AI had a good tutoring conversation? Whether it gave appropriate financial advice? Whether it correctly triaged a medical symptom? Those aren't comparisons against known answers. They're judgements about process, not just output.

I looked at the eval frameworks (DSPy / lang-graph etc). I tried to fine-tune system prompts based on conversation histories. It got complicated fast, and I still haven't cracked it fully. I don't think many people have. And I suspect this, more than inference costs or model capability, is going to be the defining challenge for AI-native products over the next few years. The companies that figure out how to reliably evaluate AI behaviour in context-dependent, open-ended interactions are the ones that will build products people actually trust.

## The gap is the job

I don't have a neat conclusion for this one, because I'm still in it. The gap between demo and production isn't something you cross once. It's something you live in.

But I've come to think the gap itself is where the interesting work happens. The model is the easy part. Building a real business around it, making it reliable for real users, knowing whether it's actually working, surviving the vendor changes and compliance requirements and platform decisions that compound over time, that's the job. It's the part that separates a demo from a product.

I sat in that car park in Geneva three years ago and thought, "this is how applications should work." I still believe that, but now I know a lot more now about what sits between that thought and something you can actually put in someone's hands.
