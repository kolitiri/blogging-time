---
title: A Personal Take on the AI Hype
description: >-
  My two cents
date: 2026-03-14 12:00:00 +0100
categories: [General Engineering]
tags: []
tok: true
---

---
Over the past couple of years, I’ve spent some time experimenting with AI tools. My main motivation wasn’t to jump on the hype train, but simply to stay current and relevant in the rapidly evolving world of software engineering.

I wouldn’t consider myself a power user of AI. In fact, only recently did I start using these tools more systematically. Once I did, however, a few interesting observations started to emerge.

This post isn’t meant to be a technical deep dive into AI. Instead, it’s a personal reflection on AI assisted development and where I believe the hype diverges from reality.


## Caught in the hype
I’ll start with a confession: I’m not the biggest AI enthusiast.

And I definitely don’t buy into the increasingly common narrative that *“AI will replace us all”*.

The idea that AI will reach some kind of **god mod** intelligence anytime soon sounds absurd to me from an engineering standpoint. Most of the people making such bold claims seem to fall into one of two categories:
- Those who profit from selling AI products and derivatives, and
- Those who are either too excited, or simply too ignorant

Right now, we appear to be in the phase of the hype cycle where FOMO has taken over.

Suddenly, everything needs to involve AI.

AI "experts" are sprouting up everywhere, often peddling illusions which couln't be further away from what the technology can realistically achieve.

Managers push engineers to integrate with AI simply so they can present something impressive to executives.

In some cases, organisations are even firing employees throwing away years worth of domain knowlege, driven by baseless assumptions they can cut down labor expenses.

And it really baffles me that most often than not, those decisions are made without the slightest clue of what the long term implications could be.

In addition, several important questions remain largely unanswered.

- What are the long term costs of integrating AI into everything?
- What happens if pricing models change in the future?
- How much dependency are we introducing into our systems?
- How maintainable will AI driven solutions be in the long run?
- Who is going to actually maintain them if all the junior, future senior, engineers have disappeared?

These are not small questions, yet they rarely appear in the day to day discussions about AI adoption.


## The real problem
When we step outside the hype cycle, the picture becomes a bit clearer.

As much impressive as the power of AI might seem at the moment, the catch is that it still works best under certain conditions that, in reality, are rarely met in most organisations.

Well structured codebases, thoughtful architectures, AI friendly processes and carefully designed failsafes can pave the way for AI to thrive.

But the reality is that organisations are messy. Their systems are layered with years, if not decades, of technical debt and it feels like impulsively integrating them with AI could only exacerbate the situation.

The foundations are just not there yet and it often feels like we’re trying to skip several evolutionary steps.

Meanwhile, the far less glamorous work to properly analyse and redesign existing systems rarely happens.

Rebuilding these systems from the ground up might not always be technically difficult, but it is almost always expensive.

And expense is often the ultimate blocker.

If there’s already a system in place that works and generates profit, convincing leadership to invest large sums of money to rebuild it rarely succeeds.

This isn’t a technological limitation. It’s a human one.

Even if AI was capable of independently taking over large parts of software development, our own organisational structures and decision making processes would still prevent us from fully unlocking that potential.

So we're stuck in a situation where we are blindly driven by the hype, patching up with shoddy solutions, hoping that we won't f**k it up even more and that everything is going to be alright.

Of course, there are exceptions which are usually either companies large enough to invest heavily in research and development, or small startups agile enough to restructure their systems without much friction.

What worries me however is the large middle ground, the majority of companies, which are foolishly trying to follow these examples without ever having put the effort to build the foundations properly.


## My personal take
Now let me be clear. As I mentioned earlier, I might not be the greatest fan of AI, but that doesn't necessarily mean I am not excited about it.

I'm just a tiny bit more skeptical and possibly defensive towards radical and irrational changes.

In my opinion, where AI truly shines today is something much more grounded.

At its core, AI is extremely good at identifying patterns in existing data and applying them to new situations.

Does that qualify as cognitive intelligence? I don’t think so. Not yet at least.

Can it render human workers obsolete? I don’t think so either. Neither do I think we should be trying to do so.

I feel there is a middle ground, that can benefit both the employees and the organisations' capitalistic hunger for profit.

That being said, instead of jumping the gun and going all in, we should be trying to make the best possible use of AI we can at the moment, while we are still in the process of laying the foundations.


## How I actually use AI

My own usage of AI is surprisingly simple.

I mainly rely on two tools: ChatGPT for quick research and documentation lookups, and Cursor for writing and manipulating code.

That’s about it.

ChatGPT works well as a kind of “lazy search engine” since it's great for answering questions or pointing me towards relevant documentation.

Cursor, on the other hand, is quite effective for code related tasks such as auto completing, generating small pieces of functionality, or modifying existing code.

I’m fortunate to work in an environment where the codebase is very well structured. Well thought architectural patterns and proper use of SOLID principles make tools like Cursor work wonders.

With just a few AGENTS.md files in place, the agent can handle a large portion of routine tasks, such as

- Adding a new API endpoint
- Writing additional test cases
- Creating adapters for existing interfaces
- Integrating an external library

These aren’t complex engineering challenges.

They’re simply repetitive tasks that are much easier to review than they are to manually write from scratch.

What I’m not doing is letting AI run freely, making architectural decisions or building entire systems from scratch.

And interestingly enough, it’s precisely these smaller, repetitive tasks where, for me personally, AI provides the most value.

Looking at roughly the last 30 features I’ve shipped, tasks that would normally take about a day to implement can often be completed in under an hour with AI assistance (with strong emphasis in the word "assistance").

This aspect is often overlooked because people try to attribute far more capability to AI than it currently has, shifting the narrative into something that sounds more like aspiration than reality.

Now, as I mentioned earlier, chances are that the majority of engineers work on legacy systems that are far more complicated even for AI to deal with.

I'm sure you've already found yourself in a situation where a single prompt can cause havoc, making wild changes that aren't just hallucinations, but also impossible to review and reason about.

But this is precisely the moment where the point I am trying to make becomes obvious.

We shouldn't expect AI to fix up the mess at the first place. It's just too much to ask yet and it doesn't do it justice.

We should instead take a step back, evaluate our architecture, understand the limitations and lay the groundwork for true AI integration in the future.



## Conclusion
If I had to summarise my experience in a single sentence, it would be this:

*"AI doesn’t replace engineering expertise, nor does it remove the need for critical thinking."*

What it does extremely well is automate repetitive work. The kind of work engineers already know how to do but would rather not spend hours implementing manually, freeing up resources to focus on more important tasks and produce more output, faster.

And while that may not sound revolutionary to those who preach about AI day and night, it is already a significant and practical improvement in the daily workflow of many engineers, which can masively increase productivity with minimal risks to an organisation.

So my two cents would be the following:
- Don't get caught in the hype
- Don't throw away domain knowlege that took years to acquire
- Don't stop investing on junior employees. They will be the future seniors you will need
- Invest in proper education that can bring long term value rather than short term gains
- Don't worry, there are still ways to increase your profits without taking up too much risk..
