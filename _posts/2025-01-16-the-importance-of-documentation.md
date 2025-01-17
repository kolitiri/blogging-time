---
title: The importance of documentation
description: >-
  An opinionated guide about the importance of documentation in software engineering
date: 2025-01-16 07:10:00 +0100
categories: [General Engineering]
tags: [documentation]
tok: true
image:
  path: /assets/img/illustrations/documentation.jpg
  alt: Documentation
---

---

## The importance of documentation
During the past ten or so years working as a software engineer, I often found myself contemplating the importance of documentation.

To begin with, we would probably all agree that documentation in general is important.

Whether it comes in the form of a [Swagger UI](https://github.com/swagger-api/swagger-ui) that documents your API, or a README file that simply describes the overall purpose of your application, it can always provide some useful information.

A good documentation can offer a valuable insight to a system, it can provide clear instructions of how to use it and also help new developers get up to speed quickly.

It might also save your future self some time when re-visiting a project that you thought was long gone..

There is however a small catch.. It is a double edged sword and if not done correctly it can cause more harm than good.


## Is it a trap?
Through out my career, I've worked in environments where information was often scattered all over the place.

In the odd case where you would come across something promising, it was written so poorly that could make a nuclear reactor manual read like the instructions of my IKEA table.

Further more, the doubt of it being still accurate was always there. We've all come across that nice document that was unfortunately last timestamped five years ago.

That being said, documentation is only useful when it is easily accessible, well maintained and written in a concise way that can be easily understood.

So where exactly do we draw that fine line that separates a useful block of information from a pile of garbage, littered with low level details that is anything but useful?


## The hard truth
There is a school of thought which holds that an application, a library or even a single programming function, should be designed in a way that is self explanatory, rendering documentation unnecessary.

But let's be realistic. People come and go, projects are handed over, priorities change.. In a real world scenario, a project will reach a peak before it gradually starts declining, drowning in technical debt.

There is also the element of seniority. You cannot expect that only experts in the field will ever be involved.

As the time goes by, details will be forgotten, things will get complicated and everything will turn into a black box.

Ok, that last one might be an exaggeration, but you get my point. Documentation in one form or another will always be necessary.

And this is presumably one of the main reasons we've been seeing a surge in tools that attempt to auto generate documentation.


## Putting things together
The first issue we have to address is the mess that can be caused when we mindlessly start dumping documents all over the place.

I am a firm believer that all types of documentation should be stored in the same space.

There should be no need to have to search through multiple sources to get information about a specific item.

I've seen a lot of places where certain information is stored for example in [Confluence](https://www.atlassian.com/software/confluence), while other directly in application repositories. Business related information might even be kept in Google drives.

I don't mind Confluence per se, but I've figured that a static site generator like [astro](https://astro.build/) or [Jekyll](https://jekyllrb.com/) works better.

A few benefits of working with a static site generator:
- You can find tons of templates and it is highly customizable
- Updates are version controlled (something the developers are used to) and most importantly can be reviewed
- Pages, which are essentially Markdown files, can be injected directly from external repositories, avoiding duplication

It does however come with a few drawbacks too:
- Business people will probably not be familiar with Markdown syntax
- Documents that are not written in Markdown will have to be re-structured
- The static site will be another application to maintain


## Identifying what needs to be documented
So, let's assume that we've now agreed on some centralized solution to store our docs, what is it exactly that needs to be documented?

I personally like to logically separate the documentation into two different categories.
- Business documentation
- Technical documentation

### Business documentation
Business documentation will probably vary depending on the industry.

However, there will always be one thing in common. Some sort of a service level agreement (SLA), that needs to clearly highlight the expectations.

Many developers might not even be aware of this type of documentation, until their boss comes around shouting that an application is "crawling" and cannot process more than a thousand requests per second.

Chances are that the developers never thought they should be able to handle a hundred requests per second to begin with.

This type of documentation is extremely valuable and should be kept widely accessible.

Some of my suggestions would be the following:

`Provide a summary of the SLA`

SLAs are usually polluted with tons of unnecessary information that in reality a developer will never need.

Create a summary of the items that are important for the system, such as performance requirements, or subscription details.

`Provide links to low level components`

If an SLA refers to a certain component of the system, make sure to add a link to its own documentation.

### Technical documentation
Technical documentation refers, as the name suggests, to the technical details of a project.

Take a look at this [Casio DW-5000 manual](https://www.casio.com/content/dam/casio/global/support/manuals/watches/pdf/35/3576/qw3576_EN.pdf).

Stunning! It has a table of contents, nice visuals and step by step instructions to configure your watch. Nothing too much, nothing too little.

Unfortunately, we are not in the watch business, but surely we can copy some of these patterns.

I like splitting technical documentation further down into two categories.
- System documentation
- Application/Library documentation

#### System documentation
System documentation refers to the overall structure of a system and is probably the hardest thing to write.

The system might be comprised of a plethora of applications, connected in obscure ways that can make even the most senior engineers cry.

In my opinion, this type of documentation should provide high level information focusing on the interaction between the applications within the system.

On the positive side, maintaining this type of documentation is probably a lot easier, because systems as a whole don't tend to change dramatically over time.

Some of my suggestions would be the following:

`Describe the overall purpose of the system`

Describe what the system as a whole tries to achieve. Is it a messaging system? Is it a monitoring platform? Or simply an online marketplace?

`Describe the architecture`

One important aspect of the human nature is that we tend to do better when visuals are involved. Remember that Casio manual?

Draw some diagrams, depicting the relationship between applications. What is the overall execution flow? What protocols are used for inter-communication?

I am personally using [draw.io](https://app.diagrams.net/) which allows you to export the diagrams in various formats (including HTML and XML) that can easily be stored in a version control system.

`Provide links to low level components`

Be precise and avoid adding low level details at this stage.

Instead, add links to the documentation of each individual component. This way, anyone viewing the diagrams can quickly drill down and inspect lower level entities.

#### Application/Library documentation

This type of documentation usually refers to low level details and specifics about an individual unit.

It comes in the form of a README file, a CHANGELOG, or even code comments and it is probably the easiest one to write.

However, in my opinion, it is the one that is most often overlooked.

There have been so many times that I run into ten year old legacy code with overly complicated functions and barely any docstrings. SHAME!!!

Some of my suggestions would be the following:

`Add READMEs`

Add README files that describe the purpose of each application and how to use it.

Point out odd cases and give reasons about certain decisions made that might appear unreasonable just by looking at the code.

Finally, add links to external resources, such as libraries that might have been used in the application.

If you decide to use a static site generator, a good thing about the READMEs is that updating them will automatically reflect back to your static site.

`Add CHANGELOGs`

If you are lucky to work on a versioned application/library, never forget to add a CHANGELOG.

Changelogs are great at maintaining a history of changes that occur in every new version release.

`Document your code`

Write well structured and meaningful docstrings. Add code comments, where necessary, to address corner cases.

Use tools like [sphinx](https://www.sphinx-doc.org/en/master/) to automatically generate code documentation.

`Document your APIs`

You can use an [OpenAPI](https://swagger.io/specification/) compliant framework to automatically document your API.

This is a great option as it usually comes with a nice UI that you can use not only to inspect your API endpoints, but also to interact with them.

`Add tests`

Tests are usually one of the best ways of documenting your code as, in theory, they set the expectations and describe the functionality.


## Problems will still arise
All these might sound nice, but there is one challenge that you will never manage to conquer and that is keeping your documentation updated.

There will always be some form of information that cannot be automatically generated and relies solely on the human element.

Some of us will find it hard to document due to lack of understanding while others will simply forget.

And of course there will be a few that will blatantly deny contributing "out of principle".

Nevertheless, we should still try as hard as possible to point out the benefits of documentation, because in my opinion, pure code is not the only important aspect in the field.

That's all folks! I'd be happy to hear your thoughts.
