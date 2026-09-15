---
title: A terrible idea with a happy ending
description: >-
  Software engineering is not always about principles and best practices. Sometimes it's just about delivering value on time.
date: 2026-09-15 08:00:00 +0100
categories: [General Engineering]
tok: true
---

---
A few weeks days ago I was having dinner with a dear friend of mine who is working for a company that creates various courses for online learning.

The company is a charitable organization with fairly limited resources and as expected, no in house software engineering team.

They use a CMS platform to manage their public website and although the structure is not complicated, they have a surprisingly large amount of content in multiple languages.

So while we were chatting over a glass of wine, she shared with me one of the latest challenges she was facing.

## The problem

A decision had been made to upgrade the CMS platform to the latest version that supports new features required for the business.

Sounds simple, right? Same product, newer version. It's a no-brainer.

However, she caught me off guard when she shared with me that they were asked to manually migrate the content to the new version.

I was baffled.. Either they were completely ignorant, or I was missing some important details.

It turns out, the new version of the CMS was so far ahead of the old one that would require significant effort to upgrade.

As a matter of fact, the options proposed by the vendor were to:
- Either proceed with the upgrade and all the effort and risk involved in it,
- Or start with a fresh CMS and manually migrate all the content.

So, naturally, given the lack of resources and experience to tackle this project, they decided to move on with the latter.

The only issue they did not take very seriously was that there were teams, like my friend's, which were responsible for hundreds of pages worth of content.

To make things worse, many of these pages had links to other pages or images, meaning that their hrefs had to also be mapped to their new location.

So the migration would be technically a nightmare and the risk of messing up pretty high. After all, how much copy-paste can a human do before making a mistake?

## I am an engineer, I'll fix it

As a software engineer, I couldn't accept the fact that there wasn't a better way to tackle this, so I started brainstorming in my head and took the task personally.

Next day I found myself reading the official documentation from the vendor, investigating available APIs or export/import capabilities and going through comments by people who tried to achieve the same goal.

But everything I found was pointing to the same conclusion: *Yes, it can be done, but does it worth the effort from a business perspective. And even if it does, who's gonna do it?*

These two versions were essentially two completely different products, which is precisely why the vendor was not able to provide a simpler solution (I should have taken their word for it).

And given that the company had no in house experience to tackle this at the first place, option two was justifiable. 

## A shameful idea

But the thing is.. I still couldn't let go. So I started thinking it over again.

These employees are not engineers, they can't write code, they can't glue together the different pieces, but.. they have **Claude**!

I remembered my friend mentioning that they had a corporate Claude account, meaning they had a decent amount of tokens, and were also allowed to use it for their content according to the company's policies.

So an idea just popped into my head.. Playwright MCP for Claude! Brute force it!

A simple installation of Playwright is all they need. No code, no programming, just a regular Claude chat and a handful of prayers to god Claude.

The plan was simple:
1. Install Playwright MCP their Claude Desktop
2. Ask Claude to extract the content from the old CMS in a structured format, with all the required metadata
3. Perform a sample check to verify the accuracy of the extracted content
4. Ask Claude to upload the content to the new CMS through the vendor's MCP that is officially supported in the new version
5. And finally, ask Claude to go through both CMS versions and verify that the content is identical. Flag it otherwise

YIKES! I can admit that as a software engineer I felt I should be ashamed of myself for even considering this.. But hey, I thought we could at least give it a shot.

## Let's make it work

After a few days I managed to sit down with my friend again, this time in front of her laptop.

We started by setting up Playwright MCP in Claude and playing around a bit with the tool so that she understood how it works.

We came up with a minimal prompt, and after some tinkering and quite a few trial and error attempts, we managed to export our first page from the old CMS and import it into the new one.

Not much of a surprise to me but the look on my friend's face was priceless. She was looking at me as if I was a magician of some sort.

***"Can we do a bulk migration?"***, she asked.

***"Oh hell yeah, we're deep into it now! Let's just hope Claude won't trash the whole CMS and you will get to keep your job"***, I replied.

In reality, that was not a great deal of concern. My friend's account was only scoped to a certain project, so Claude couldn't really mess anything up even if it wanted to.

So we tampered a bit with the prompt, pointed Claude to the right directory in the CMS and there you go.

The whole process must have taken around 7 hours to complete and almost 200 pages were migrated successfully.

Bonus points, she now had a complete backup in her local machine if anything went wrong.

In all honesty, I was actually quite amazed with the result knowing how much time it would have taken to do it manually.

Fun fact, my friend has now magically become the migration expert in the company and has to assist the rest of the teams do the same. **Oops!**

## A leason learned

I don't consider myself a hard core engineer by any means. However, I do take pride of my work and I usually dislike solutions that are not elegant or efficient. Let alone half baked or unsafe.

We have all spent a great deal of time and effort to learn and master our craft, thus, solutions of this nature feel more like a gimmick.

But at the end of the day, our job is to solve problems and deliver value.

This principle stands the same regardless whether you work for a large organisation or simply helping out a friend.

So in this case, I'll just put my pride aside and simply enjoy this odd feeling of satisfaction we all get when we manage to solve a problem without starting a fire in the kitchen!
