---
title: Creating Technical Content
date: 2026-04-10
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - Content
  - Pre-Sales
  - Leadership
excludeSearch: false
images:
  - /images/content_direction_of_flow.svg
---

I've been creating technical content in one form or another for over 20 years. Blog posts on the AWS and Azure blogs, a book on Microsoft automation, presentations at conferences, workshops delivered to customers in cities across three continents, and more recently a 10-part series on Claude's API for my own site. Some of it had real, measurable impact. Some of it disappeared into the void. The difference was almost never the quality of the technical material itself.

The content that worked had one thing in common: it started with the customer's problem, not the technology. The content that didn't work, and I've created plenty of that too, started with something I thought was interesting and hoped someone else would find useful. That distinction sounds obvious when you write it down. In practice, technical people get it wrong constantly, including me.

## The mistake

If you work in a technical pre-sales role, or any customer-facing technical role, you've seen this happen. An SA or engineer gets excited about a service or architecture pattern and builds a presentation around it. They demo SQS, or walk through a CQRS pattern, or show off a new model's benchmark results. The content is technically excellent. The delivery is confident. The customer sits politely, asks a few questions, and nothing changes.

The problem is that the content was built from the technology outward. "Here's what this thing can do, now let's find where it fits in your world." That's backwards. A CTO running a digital conference platform doesn't care about SQS. They care about the fact that their platform falls over when 50,000 people try to join a keynote simultaneously, and their team has been patching the same bottleneck for three years. If you can connect the technology to that specific pain, you have their attention. If you're pitching the technology and hoping they make the connection themselves, you've lost them.

This sounds like basic sales advice because it is. But technical people resist it, because we're trained to lead with the thing we know best. The tech is our comfort zone. Moving it to the back of the conversation feels unnatural, like showing up to a job interview and not mentioning your qualifications until the last five minutes.

<img src="/images/content_direction_of_flow.svg" alt="What most people do vs what actually works" style="background: #1a1a2e; border-radius: 0.5rem; padding: 1rem;" />

## Customer first, tech last

When I'm working with other SAs on how to structure a customer engagement, whether it's a workshop, a presentation, or a piece of written content, I don't tell them what to do. I ask questions. The kind of questions that seem obvious but that most people haven't actually answered before they start building slides.

"Who are we communicating to?"

"What's the pain point we're looking to solve?"

"Do we have any examples of success in a similar situation?"

"Do we understand the customer's existing preferences or biases?"

These aren't sophisticated questions. But if you can't answer them clearly before you start building slides, whatever you create won't land. Knowing the customer is the most important part. The technology comes last, and that's how I structure the conversation with SAs I'm mentoring, the same way the conversation should go with customers.

I learned this the slow way. Early in my time at AWS I built content the same way everyone else did: start with the service, explain what it does, show a demo, hope the customer sees the relevance. It works often enough that you think it's working well. It's not. You're just getting by on the strength of the technology itself and the fact that some customers will do the translation work for you. The best engagements I had were the ones where I'd spent enough time understanding the customer's world that I could frame the technology inside their problem, using their language, referencing their constraints. When you do that, the customer stops evaluating the tech and starts imagining it in their environment. That's the shift you're looking for.

## Formats and where they land

<img src="/images/content_format_tradeoffs.svg" alt="Content formats in a sales context" style="background: #1a1a2e; border-radius: 0.5rem; padding: 1rem;" />

Not all content is equal in a sales context, and the differences aren't what you'd expect.

Blog posts and written content have a slow, wide reach. A post on the AWS blog about building progressive web apps with Amplify and AppSync wasn't aimed at one customer. But it was connected to real work I was doing with the Australian Bureau of Statistics. They'd come off the back of the 2016 Census failure, I'd helped them deliver the digital component of the Same-Sex Marriage plebiscite, and that post was part of the broader technical groundwork that fed into them pushing ahead with a digital Census on AWS in 2021. The post itself didn't close a deal. But it was a visible artefact of the work we were doing together, and it gave other teams inside the ABS (and other government agencies) something to point at when they were building the case internally. Written content is a long game. It builds credibility across many potential customers over time. Individual impact is hard to measure, but cumulative impact is real.

Workshops are different. They're the highest-impact format I've found for moving customers forward, and it's not close. When I built the GenAI enablement program for the UN ecosystem, the workshops we delivered in agency offices across multiple countries triggered more new conversations and opportunities than anything else we did. More than blog posts, more than presentations, more than any amount of email follow-up.

But the reason workshops work isn't because the content is better than what you'd put in a blog post. It's because you're in the room. When you spend a week at a customer's office running sessions, you build relationships that wouldn't happen over a video call. You have hallway conversations. You learn about problems they haven't put in a brief. You meet people who weren't in the original meeting invite but who turn out to be the actual decision makers. The content is the vehicle for the proximity, and the proximity is where the real work happens.

Presentations and conference talks sit somewhere in between. They're good for visibility and credibility, terrible for depth. A 30-minute talk at a conference has never, in my experience, directly moved a customer engagement forward. What it does is make the next conversation easier, because the customer has already seen you demonstrate competence in public. It's a trust accelerator, not a sales tool.

## Building the enablement program

The GenAI workshops I ran across the UN system are probably the best example of how all of this comes together. Nobody asked me to build them. I saw a gap. Almost every customer conversation I was having was converging on the same topic, everyone wanted to understand what generative AI meant for their agency, but nobody had given them a structured way to think about it.

So I built it. I defined the topics and flow, the target audience level, the host agencies in each city. I led the initiative, but I needed help building and delivering it, so I got my team of SAs to own parts of the content development. I found people in each location to help with local delivery. I brought the sales team into the equation so they were visible as leaders to their customers, not just the technical people. I never asked permission or waited for someone above me to approve a plan. I just started building, and pulled in the people I needed along the way.

The material itself was solid, but it wasn't the material that made it work. It was the structure around it. The right people in the room. The sales team feeling ownership. Local hosts who understood their agency's context. The SAs who'd built parts of it feeling invested in the delivery. The content was the scaffold, but the building was made of relationships.

## What I'd tell someone building a technical content function

If I were setting up a technical content function inside a sales organisation today, here's what I'd focus on.

Start by listening, not creating. Spend the first few weeks in customer meetings, not at your desk writing. Understand what questions customers are actually asking, where the sales team is getting stuck, what the recurring objections are. The content backlog writes itself once you know the real problems.

Structure every piece of content around a customer problem, not a product feature. The technology is the answer. The content should be framed as the question.

Invest disproportionately in workshops and hands-on formats. Blog posts and presentations have their place, but if you want to move pipeline, get your technical people in the room with customers for extended engagements. The content justifies the time, the relationships justify the investment.

Bring sales into the process early. Not as an afterthought, not as a distribution channel. Make them part of the planning so they feel ownership over the outcomes. If the sales team doesn't see your content as their content, it won't get used.

And measure the right things. Blog post views and conference attendance are vanity metrics in a sales context. What matters is: did a piece of content open a conversation that wasn't happening before? Did a workshop create relationships that led to new work? Those are harder to measure, but they're the ones that actually tell you whether your content is working.
