---
layout: post
title: State of Serverless
tags:
  - aws
  - serverless
---

The latest [report on the state of serverless from DataDog](https://www.datadoghq.com/state-of-serverless) was a bit disappointing for me. Here’s why.

There is a lot of talk about what serverless means. There are those who say we shouldn’t gate keep, essentially saying everyone’s opinion matters. There is, of course, truth to that, but at some point we have to come to an agreement about what something means in order to have meaningful conversations. Conversations like “what is the state of serverless?” can’t happen if we don’t have some agreed upon understanding of what serverless is. That is where my disappointment with DataDog’s report lies. They’ve chosen to decide for the community what serverless means, and, in my (and many others) opinion, they’re wrong.

## Why do I care?

First, let me tell you why I even care. As I’ve already stated, an agreed understanding is vital to continued conversations about a subject. If we cannot agree on what something is how can we reasonably discuss it? But why, then, do I care whether the view DataDog has taken is the agreed upon view?

I care because I’m tired of companies taking over terminology. I’m tired of a word or phrase turning into a marketing tool rather than having actual meaning. Let me give you a recent example.

What do you think of when you hear DevOps? Do you think of a culture shift in engineering that shifts the way we work and how we think about supporting software? Or do you think of a team that manages K8 clusters and CI/CD tooling? DevOps meant something at one point and it was hijacked to mean something very different. In its early days you could talk to people about DevOps and, assuming they were familiar with the term, you’d be talking about the same thing. Now days I get constant messages from recruiters about a “DevOps Role” and I know right away they aren’t talking about the DevOps I believe in. It’s rare that I can have a conversation about DevOps anymore and expect people to be on the same page as to what it means.

## What is serverless?

So, what do I think serverless means, or should mean? Let’s start with what [Wikipedia says](https://en.wikipedia.org/wiki/Serverless_computing) (at the time of this writing):

> Serverless computing is a cloud computing execution model in which the cloud provider allocates machine resources on demand, taking care of the servers on behalf of their customers. "Serverless" is a misnomer in the sense that servers are still used by cloud service providers to execute code for developers. However, developers of serverless applications are not concerned with capacity planning, configuration, management, maintenance, fault tolerance, or scaling of containers, VMs, or physical servers. Serverless computing does not hold resources in volatile memory; computing is rather done in short bursts with the results persisted to storage. When an app is not in use, there are no computing resources allocated to the app. Pricing is based on the actual amount of resources consumed by an application. It can be a form of utility computing.

There are some key points here.

### On demand

This one is a bit of a no brainer, but serverless needs to be on demand. Of course what that means, exactly, can be debated. After all, I can spin up an EC2 in AWS “on demand”, but most would agree that doing so would not qualify as serverless. On demand is part of the picture, but certainly not all of it.

### Consumption based pricing

Again, pretty obvious. And again, it can be applied to things that aren’t serverless, too.

### Ilities

Serverless should have all the ilities, like scalability, availability, and so on. These things are inherent in serverless and not something to be configured. That’s not to say you can’t have some controls in place (e.g., Lambda concurrency limits) but if autoscaling groups and multi-AZ are things you have to worry about then, to me, you aren’t doing serverless.

### Single execution

This one may not be as obvious when reading the Wikipedia definition, but it’s in there.

> Serverless computing does not hold resources in volatile memory; computing is rather done in short bursts with the results persisted to storage.

I think the “results persisted to storage” bit is a bit out of line, but the essence is that there is no expectation that you’ll have access to anything from a previous operation. We do have some ability to cache things in Lambda, but it’s very limited and isn’t something you rely on by any stretch.

### Scale to zero

This is probably the most significant, and maybe controversial, part of the definition. Serverless doesn’t cost anything when there isn’t anything being executed. It’s important because it means I can afford to have entire teams of engineers, each with a fully functional serverless application, and not worry about the cost. This leads to more innovation, faster development, and greater understanding. As soon as you take away scale to zero you take away the freedom of individual engineers to try things that might break the system because doing so will impact everyone.

## Containers are not serverless

Okay, I said it. I’m ignoring the fact that you can use a Docker image in a Lambda (it’s still a Lambda, not a container). Yes, I’m including Fargate in that. Now, before you list all the use cases for a container and tell me how I’m wrong about Fargate let me say a few things.

Note that I didn’t say you can’t have containers in a serverless architecture. What I’m saying here is that just because you are using Fargate (or it’s equivalent) doesn’t mean you are serverless. Fargate may have a place in a serverless world, but it’s in a limited role. If you spin up a container to perform a single task, and shut the container down when it’s done then you are possibly doing serverless. Be mindful of what you are doing in the container though. If it’s just a giant batch job that is really a bunch of smaller jobs put into a single execution environment you have probably crossed the line (and might want to consider Step Functions instead).

When I say containers aren’t serverless I mean that if you aren’t doing all the things that make something serverless then it’s not serverless. When DataDog includes Fargate in its report it’s unable to know how the container is being used, and thus can’t know if it’s being used in a true serverless mode. Frankly, I'd bet that it most often is not.

We may not all agree on the minor details of what makes something serverless (personally I’d like to exclude anything that requires networking), but we need to have a general understanding to be able to have conversations about it. More importantly, we need to do what we can to avoid losing the term completely, lest it become like “cloud native” (which isn’t).