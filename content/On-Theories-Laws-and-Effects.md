---
title: On Theories, Laws and Effects
date: 2025-09-16
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - Theories Laws and Effects
  - Professional Development
  - Corprate
  - Culture
excludeSearch: false
---
When I started at AWS in 2016, I didn't know it at the time but I was about to start learning about various theories, laws and effects. My manager, Simon, introduced me to the first one that caught my attention (for purely technical reasons), but soon after I started taking more and more notice of these seemingly universal and well-accepted statements about situations and behaviours.

Now I've had time to absorb them and see how they apply in the real world, I wanted to call out some of my favourites and talk about my experiences with them. Overall, I don't think you can pigeonhole any one person or situation with these, but by leveraging them as a collection of mental models you can start to understand the reasons behind someone's thoughts or actions, even your own.

## Technical Truths: When Systems Meet Reality

**CAP Theorem**

The CAP theorem states that distributed systems can only guarantee two out of three properties: Consistency, Availability, and Partition tolerance. When Simon first explained this to me, it felt like learning that you can't have your cake, eat it, and also keep it forever. The funny thing is, Google's Spanner paper claimed to have "beaten" CAP by using synchronized clocks, which is a bit like saying you've beaten the speed of light by redefining what "speed" means. Not many companies or people really need to push this theory to the limits, not everyone gets to build truely globally distributed and extremely fault tolerant systems, but it is a very helpful reminder that there are always trade-offs in any system you might build… and in anything you do in life. Choose wisely.

[https://en.wikipedia.org/wiki/CAP_theorem](https://en.wikipedia.org/wiki/CAP_theorem)

**Conway's Law**

"Any organisation that designs a system will produce a design whose structure is a copy of the organisation's communication structure." I've seen this play out so many times amongst my employers and my customers. That micro-services architecture that perfectly mirrors your org chart? That's Conway's Law winking at you. The three different APIs that do almost the same thing because three teams don't talk to each other? Conway strikes again. Most visibly though is a “system” in the non-technical sense, like how org’s constrain their communications or their activities based on business structures: like when Sales and Product both want to own/control customer relationships, so they both avoid talking to each other, making a royal mess for customers to deal with. 

[https://www.melconway.com/Home/Conways_Law.html](https://www.melconway.com/Home/Conways_Law.html)

**Two is Better Than None**

This principle acknowledges that in large organisations like AWS, you'll often see multiple teams building similar solutions. It's the organisational equivalent of survival of the fittest, let different approaches compete and see what survives. I've watched competing teams at big tech companies build nearly identical services, and while it seems wasteful, sometimes the redundancy leads to breakthrough innovations. Other times, well, you just get two mediocre solutions instead of one good one.

[https://pedrodelgallego.github.io/blog/amazon/tolerating-eliminating-duplication/](https://pedrodelgallego.github.io/blog/amazon/tolerating-eliminating-duplication/)

## The Human Factor: Cognitive Quirks and Biases

**Dunning-Kruger Effect**

The less people know about something, the more confident they tend to be about their knowledge. I've sat in countless meetings where the person who just learned about “distributed systems”/”serverless”/”Artificial Intelligence” yesterday is absolutely certain about the "right" architecture. Meanwhile, the principal tech with 20+ years of experience is hedging every statement with "it depends" - I know I said this an awful lot over the years, sorry.  Peak confidence often signals minimum competence. Stay humble and accept that you’ll never know everything, stay in learning mode as much as you can, you’ll end up better off in the long run, so will your companies and customers.

[https://www.verywellmind.com/an-overview-of-the-dunning-kruger-effect-4160740](https://www.verywellmind.com/an-overview-of-the-dunning-kruger-effect-4160740)

**Gell-Mann Amnesia Effect**

You read an article about your area of expertise and spot ten errors. You turn the page and read about economics or politics, forgetting that the same publication probably got that wrong too. Named after physicist Murray Gell-Mann, this effect is everywhere in tech journalism. I'll rage about how wrongly AI is portrayed when it's about my field (and it is portrayed wrong a lot!), then nod along to articles about quantum computing as if they're gospel.

[https://www.epsilontheory.com/gell-mann-amnesia/](https://www.epsilontheory.com/gell-mann-amnesia/)

**Hanlon's Razor**

"Never attribute to malice that which is adequately explained by stupidity", or in corporate speak, "incompetence or misaligned incentives." This one's my personal mantra, especially in big tech where people's motivations are often self-interest-driven. When that other team blocks your project, they're probably not evil; they just have different OKRs. Remembering this has saved my sanity more times than I can count. I try to believe that true malice is exceptionally rare, especially in the professional world, so give those stupid people a break 😉.

[https://en.wikipedia.org/wiki/Hanlon%27s_razor](https://en.wikipedia.org/wiki/Hanlon%27s_razor)

**Occam's Razor**

The simplest explanation is usually correct. When your service goes down, it's probably not a coordinated DDoS attack from state actors (sorry InfoSec peeps, it’s just the truth), someone likely just pushed bad config. I brought down ~~many~~ one-or-two systems in prod by doing dumb shit, we all do, thats how we learn. When a project fails, it's rarely due to sabotage; usually, the requirements were just unclear from the start. When a system crashes, don’t dive deep on a stack trace and recompile your kernel, check what changed last night first.

[https://en.wikipedia.org/wiki/Occam%27s_razor](https://en.wikipedia.org/wiki/Occam%27s_razor)

## Organisational Dynamics: How Work Really Works

**Parkinson's Law**

"Work expands to fill the time available for its completion." Leading large programs for public sector customers, I watched this law in action constantly. Give a team two weeks for a task that takes two days, and somehow it'll take exactly two weeks. The worst part? They'll genuinely feel busy the whole time. Public sector customers were masters at this, pushing out dates rather than committing to hard deadlines, watching projects balloon to fill whatever timeline was allowed. Sometimes there are valid reasons, but sometimes it’s humans lazy nature kicking in, making us avoid the hard work as much as possible.

[https://en.wikipedia.org/wiki/Parkinson's_law](https://en.wikipedia.org/wiki/Parkinson%27s_law)

**Peter Principle**

People rise to their level of incompetence. That brilliant engineer who got promoted to manager and now runs meetings like they’re diving deep into a compiler error? That's the Peter Principle. That CIO/CTO that can’t make a decision until everyone in the business has put their two-cents in, then ends up hedging their bets to avoid failure, thereby making failure more likely? That’s the Peter Principle. That skip-level manager that is always busy doing something else, not able to contribute/communicate with teams? That’s the Peter Principle. The frustrating part is that organisations keep promoting their best individual contributors into leadership roles they're not suited for, losing great ICs and gaining mediocre leaders/managers. It’s a good thing to put yourself in situations where you’re out-of-depth sometimes, it forces you to learn. But in the rampaging scurry up the ever-lengthening corporate-ladder that we do sometimes, we should be wary of becoming a Peter, especially if it means that life becomes harder for others. 

[https://en.wikipedia.org/wiki/Peter_principle](https://en.wikipedia.org/wiki/Peter_principle)

**Gregor's Law**

"Excessive complexity is nature's punishment for organisations that are unable to make decisions." I've seen codebases that look like archaeological sites, with layers upon layers of different architectural decisions because no one could commit to a direction. Every "let's support both approaches for now" decision adds another sedimentary layer to your technical debt. I’ve done it, “lets create an abstraction so we can have multiple providers underneath and don’t have to commit now” always seems like such a logical thing to do, but it hurts later.

[https://architectelevator.com/gregors-law/](https://architectelevator.com/gregors-law/)

**Hofstadter's Law**

"It always takes longer than you expect, even when you take into account Hofstadter's Law." Building my own startup has made this painfully real. I'll estimate a feature will take a week, double it to be safe, and somehow still be debugging edge cases three or four weeks later. It's recursively true, even knowing about the law doesn't help you avoid it.

[https://en.wikipedia.org/wiki/Hofstadter's_law](https://en.wikipedia.org/wiki/Hofstadter%27s_law)

**Jevons Paradox**

When technological progress increases efficiency, consumption often increases rather than decreases. Make cloud computing cheaper and easier? Companies don't save money, they spin up more resources. Create faster deployment pipelines? Teams don't work less, they deploy more frequently. I see this everywhere in tech: every efficiency gain gets immediately consumed by new demands.

[https://en.wikipedia.org/wiki/Jevons_paradox](https://en.wikipedia.org/wiki/Jevons_paradox)

## Human Scale and Performance

**Dunbar's Number**

Humans can maintain about 150 stable social relationships. This explains why startups feel different once they grow beyond a certain size, why large organisations split into divisions, and why that 500-person all-hands meeting feels so pointless. Once your organisation exceeds Dunbar's number, you need formal processes to replace informal communication, and thats when that organisation feels like less of a group of humans, and more a collection of resources. Don’t rush to grow your company too huge too quick, it’s vibe will change dramatically as you grow.

[https://en.wikipedia.org/wiki/Dunbar%27s_number](https://en.wikipedia.org/wiki/Dunbar%27s_number)

**Yerkes-Dodson Law**

Performance increases with arousal/stress up to a point, then decreases. A little deadline pressure can spark creativity; too much and people make sloppy mistakes. I've seen teams deliver their best work under moderate pressure and their worst during both dead periods and crisis mode. Having 100 days to deliver a critical new system that would normally take 200 days, puts you in go-mode and forces you to focus. Having to deliver a major presentation in 2 days that would normally take weeks to prepare, you’re in a world of hurt and you’re brain is not going to cope.

[https://en.wikipedia.org/wiki/Yerkes%E2%80%93Dodson_law](https://en.wikipedia.org/wiki/Yerkes%E2%80%93Dodson_law)

## Looking Forward

**Amara's Law**

"We tend to overestimate the effect of a technology in the short run and underestimate the effect in the long run." Remember when everyone thought smartphones were just expensive toys? Or when cloud computing was "just someone else's computer"? Every major technology goes through this cycle, initial hype, disappointment, then gradual world-changing adoption that we barely notice. We’re in the middle of this one right now, AI is very hyped at the moment and probably not having the sweeping immediate impacts that some estimated, but it will absolutely have massive impacts to the way the world works over the coming years and decades.

[https://en.wikipedia.org/wiki/Roy_Amara](https://en.wikipedia.org/wiki/Roy_Amara)

## The Meta-Law

Here's the thing about all these laws and effects: knowing about them doesn't make you immune to them. I still fall victim to Dunning-Kruger when I venture outside my expertise, “yeah NextJS is TypeScript just like React, it’ll be easy”. I still underestimate timelines despite knowing Hofstadter's Law by heart, I’m 9 months into a 3 month build right now! But having these mental models helps me recognise what's happening, take a step back, and maybe, just maybe, make slightly better decisions.

These theories, laws, and effects aren't prescriptive rules or universal truths. They're patterns that tend to emerge in complex systems full of humans trying to build things together. Use them as lenses to understand the chaos around you, not as hammers to hit every nail. 

So remember: *The fact that humans have so many "theories, laws and effects" says more about our need to find patterns in chaos than about any fundamental truths of the universe.*

Maybe that's just another law waiting to be named, for now lets call it Costellos Law.