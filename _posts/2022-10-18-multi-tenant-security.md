---
layout: post
title: Multi-tenant Security
tags:
  - security
  - SaaS
  - multi-tenancy
---

Security is hard. Multi-tenant security is harder. Multi-tenancy, however, is what makes the SaaS model work, and so security becomes something that needs to be at the forefront of your system's architecture.

Let me start by telling you a bit more about what I mean when I talk about security in a multi-tenant platform. Security has many levels, from user authentication and authorization, to encryption of data at rest and in flight, and probably a million other things (I'm not a security expert and won't pretend to be one). Those things are important, but providing user authentication and encrypting data doesn't solve the multi-tenancy problem. A former colleague of mine once summed it up for me something like this:

> If a "where" clause on a SQL statement is your idea of security then you aren't protecting my data.

When I first heard that I was stunned. That's what I had been doing. Everywhere I worked, everyone I talked to, was doing just that. Why was that not enough? What else should I be doing?

## The multi-tenancy problem

Why _was_ that not enough? Well, let me make this easy for you. What happens when a developer makes a mistake and leaves off that "where" clause? Now every tenant sees every other tenant's data. It's a very simple problem with very real probability and significant consequences. How significant the consequences probably depends a bit of the type of data, but at a minimum it impacts the trust customers will have in your platform.

## Testing

The problem is easy enough to identify, so the next step is to address it. Your first thought might be to add some testing that makes sure you are including your tenant identifier in your queries. I'm not going to tell you to not write tests -- tests are great -- but this problem isn't one you can test yourself out of. Tests are only as good as your ability to remember to write them. Sometimes you can generalize things in a way that allows you to always include them, and perhaps you can do that here (that would seem to remove some degree of team autonomy, but that's a different topic). Tests, however, are not part of the system that is running, and while you hope you'd catch something before it is deployed it doesn't always work out that way.

## So...?

So we know we have a big problem to solve. How do we systematically make sure tenant A can't see tenant B's data, and visa versa? This requires a change to how you think about accessing data within a system.

## How it's typically done

Typically you have some bit of software that uses a database of some sort, and that software has credentials to allow it to access that database. These credentials can read and/or write to any of the data in that database, and it's up to the software to control what data is being read/written. That, as we've already stated, is the problem. The software is prone to mistakes, and the credentials do nothing to protect us from those mistakes.

## A better way

What if you could protect yourself from those mistakes? I'm not suggesting you won't make the mistakes. Rather, I'm suggesting that the mistake doesn't result in exposing data it shouldn't. The solution is, frankly, pretty simple. Instead of the software having credentials to read/write to any of the data in the database it doesn't have any credentials. Okay, obviously it needs to have some credentials to work, but what if those credentials weren't so much a part of the software as much as a part of the process of the software. In other words it's not a configuration value that is global to the software but is something that the software gets as a part of the process. That alone doesn't help, but one small step does. Instead of having a set of credentials that allows access to the entire database you need to have credentials that are unique to the tenant. Those credentials should fundamentally limit access to only that tenant's data. Think of this as row-level security (though you could implement it a number of ways). The software gets the credentials for the tenant it's currently processing and therefore can only access that tenant's data.

If we go back to our missing "where" clause problem we can see that this would no longer be an issue. Well, it's still an issue because now your application, or feature, isn't working, but an error message is a lot easier to explain than "oops, you weren't supposed to see that". And hopefully those tests you were writing at least found that error.

I'm not foolish enough to think this solution is bullet proof, but I do have a pretty high level of confidence that we aren't going to make a mistake that will accidentally show one tenant's data to another. At the very least I can say with confidence that "a 'where' clause in a SQL statement" is not my idea of security.

Stay tuned for my next post where I'll show you how we did this at ByteChek