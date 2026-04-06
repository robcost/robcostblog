---
title: On AI Roles
date: 2026-04-08
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - AI
  - Careers
  - Hiring
excludeSearch: false
images:
  - /images/ai_roles_the_line.svg
---

I've been looking at the AI job market recently to get a sense of whats out there, so I've spent more time than I'd like scrolling through job posts trying to figure out what companies actually want when they post an AI role. It's been... educational.

In the last few months I've looked at roles titled Solutions Architect, Applied AI Engineer, AI Engineer, Forward Deployed Engineer, ML Engineer, AI Evangelist, and various manager-level versions of all of the above. Some of these are clearly different jobs. Some of them are the same job with different names. And some of them are different jobs with the same name, which is the most dangerous combination if you're either applying for one or trying to hire for one.

The AI role market right now is a mess. Not because people aren't trying to get it right, but because the industry is moving faster than the job titles can keep up. Companies are inventing roles, borrowing titles from other companies without borrowing the definitions, and often not being clear about what they actually need. If you're a technologist trying to figure out where you fit, or a manager trying to build a team, this post is my attempt to draw some lines that I think will hold up for a while.

## The line

The simplest way I've found to think about AI roles is to draw a horizontal line. Above the line is where you build *with* AI. Below the line is where you build AI.

<img src="/images/ai_roles_the_line.svg" alt="Build with AI vs Build AI" style="background: #1a1a2e; border-radius: 0.5rem; padding: 1rem;" />

Below the line is model development. Training, fine-tuning, optimisation, inference infrastructure, the maths-heavy work of making models better. This is ML Engineering territory, and it has been for years. The people working below the line are typically strong in linear algebra, statistics, PyTorch, and training pipelines. They work at companies like NVIDIA, Meta, Google DeepMind, and Anthropic's research team. The title "ML Engineer" has been around long enough that it means roughly the same thing everywhere, which is more than you can say for most AI titles right now.

Above the line is where things get confusing. This is where you take existing models, the ones the below-the-line people built, and use them to build products, solve business problems, and help customers. The work involves API integration, prompt engineering, evaluation, RAG architectures, agentic workflows, and a lot of system design that has nothing to do with how the model itself works. You don't need to understand backpropagation to do this work well. You need to understand how to build reliable software and how to make AI do something useful in a specific context.

The problem is that "above the line" work gets called a dozen different things depending on who's hiring.

## The title zoo

![Same title, different job](/images/ai_roles_company_context.svg)

**AI Engineer** is the one causing the most confusion. At a company that builds AI, like a model provider, an "AI Engineer" might be working on inference infrastructure or model serving, which is below-the-line work. At a company that uses AI, like a SaaS platform or a startup building on Claude or GPT, an "AI Engineer" is almost certainly working above the line, integrating models into products. Same title, different job, different skills required.

Despite all the buzz around "AI Engineer" being the hot new role, the data tells a different story. The vast majority of people actually doing AI/ML work still have "ML" in their title. "AI Engineer" as a formal corporate title barely exists at the big labs. They use domain-specific titles instead: inference engineer, alignment researcher, safety engineer. The "AI Engineer" title lives mostly at companies building on top of models, not companies building the models.

**Applied AI Engineer** is the clearer version. The "Applied" modifier is doing real work here, it signals above-the-line activity explicitly. When Anthropic, Google, or Salesforce hire an "Applied AI Engineer", they mean someone building applications and tooling on top of existing models. I think "Applied" will become the standard corporate modifier, even if "AI Engineer" stays as the broad community term.

**Solutions Architect** is a title I know well from eight years at AWS, and it's one that varies wildly across companies. At AWS it was relatively clear: you were a pre-sales technical advisor, either a generalist or a specialist. At other companies, "Solutions Architect" can mean delivery, consulting, pre-sales, or some combination. In the AI space, it's typically pre-sales and advisory, helping customers understand what's possible and designing architectures for their use cases. Compared to engineering roles, SAs write less production code and spend more time on discovery, demos, and technical strategy.

**Forward Deployed Engineer** is the newest addition to the zoo. Palantir invented the role over a decade ago, but it's exploded in the last year. OpenAI, Anthropic, Salesforce, Scale AI, and others all have FDE teams now. The key distinction from a Solutions Architect is that FDEs write production code on customer infrastructure. They're embedded with the customer, building real systems, not just advising. At OpenAI, the FDE role was explicitly created as something different from Solutions Architect, more hands-on, more ambiguity, and more involved in feeding insights back to the product team. If you're a hiring manager, the FDE versus SA distinction matters a lot. Hiring an FDE when you need an SA means you're paying for engineering time you're using for advisory work. Hiring an SA when you need an FDE means you're putting someone in a delivery role without delivery expectations.

**Evangelist**, **Developer Advocate**, **AI Champion**, these are the roles that sit at the intersection of technical depth and community building. They write content, run workshops, speak at conferences, and build developer mindshare. They're technical but the output is influence and adoption, not shipped code.

## Why this matters if you're hiring

If you're a manager building an AI team, getting the titles right isn't just semantics. The title you post determines who applies, and the wrong title attracts the wrong people.

Post "AI Engineer" when you mean "Applied AI Engineer" and you'll get a mix of people who want to build models and people who want to build with models. Half your applicant pool is wrong for the job before they've even interviewed. Post "Solutions Architect" when you actually need someone writing production code on customer systems, and you'll hire someone who's great at whiteboards and discovery calls but isn't set up to deliver what you need.

The harder question for managers is understanding what combination of roles you actually need. A startup building on Claude doesn't need an ML Engineer. They need someone above the line who can integrate the API, build reliable systems around it, and evaluate whether the AI is actually doing what it's supposed to do. A company building its own models needs people below the line, but they also need above-the-line people to turn those models into products that customers can use. Most companies need both sides of the line, but the ratio matters, and getting it wrong means either over-investing in model work when your problem is application quality, or under-investing in engineering when your product keeps breaking in production.

The other thing managers need to understand is that these roles have very different career trajectories. An ML Engineer who wants to grow will typically move toward ML research, ML infrastructure, or ML team leadership. An Applied AI Engineer will grow toward staff engineering, architecture, or product leadership. An SA will grow toward management, pre-sales leadership, or customer success. If you hire someone into a role that doesn't have a growth path they care about, you'll lose them within two years, no matter how well-matched they are to the current job.

## Where things are headed

I think the "Applied" modifier is going to win. It's the clearest signal that a role is above the line, and it avoids the confusion that bare "AI Engineer" creates. I also think Forward Deployed Engineer will keep growing as a title, because the work of embedding with customers and building real systems is genuinely different from advisory and pre-sales work, and companies are figuring out they need to name that difference.

"ML Engineer" will stay stable below the line. It's well-defined and widely understood. "Solutions Architect" will persist but continue to mean different things at different companies, which is a problem the industry hasn't solved in the 20 years I've been seeing the title.

The roles that don't exist yet are the ones I'm most curious about. AI evaluation is becoming its own discipline. AI safety in production contexts is different from AI safety research. AI-native product management is different from traditional PM work. These will get their own titles eventually, and when they do, we'll go through another round of confusion until the definitions settle.

## A rough map

If you're a technologist trying to orient yourself, here's how I'd think about it.

<img src="/images/ai_roles_career_paths.svg" alt="Career paths across AI roles" style="background: #1a1a2e; border-radius: 0.5rem; padding: 1rem;" />

If you want to train and optimise models, and you're strong on maths, statistics, and frameworks like PyTorch, you're looking below the line. ML Engineer, Research Engineer, or domain-specific roles at model companies.

If you want to build products and systems using AI, and you're strong on software engineering, API integration, and system design, you're looking above the line. Applied AI Engineer or AI Engineer at a company that uses AI.

If you want to be customer-facing and technical, helping organisations adopt AI through advisory and architecture work, you're looking at Solutions Architect or a similar pre-sales role. Less code, more strategy and communication.

If you want to be customer-facing and hands-on, embedding with customers to build real production systems, you're looking at Forward Deployed Engineer. More code, more ambiguity, more autonomy.

If you want to lead any of these teams, the path to management depends on which side of the line you're on. Managing an ML team requires deep model expertise. Managing an Applied AI or SA team requires understanding customers, go-to-market strategy, and how to hire across a range of technical profiles. They're different management jobs, and being great at one doesn't automatically prepare you for the other.

The titles will keep changing. The line won't. Figure out which side you belong on, and the rest gets a lot simpler.
