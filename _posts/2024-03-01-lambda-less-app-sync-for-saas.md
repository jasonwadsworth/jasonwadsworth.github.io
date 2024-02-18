---
layout: post
title: Lambda-less AppSync for SaaS
tags:
  - aws
  - SaaS
  - AppSync
  - Step Functions
---

As a builder of SaaS software, I often find myself looking at services like AppSync with a bit of jealousy. See, AppSync has a way for you to interact directly with services like DynamoDB, removing the need for a Lambda function, and the cold starts that come with it. As a SaaS builder, these direct integrations have always been out of reach because of the inability to secure the data at the tenant level. Due to some features introduced by the Step Functions team last year, there now is a way. In this post, I'll walk you through how you can access DynamoDB data from an AppSync API without the need for a Lambda function, all while maintaining tenant data isolation.

## Tenant Isolation

Before getting into the details of how this solution works let's be sure we understand the problem we are trying to solve. If you are building a multi-tenant SaaS application it's important that your application be built in such a way that one tenant isn't able to access another tenant's data. I talk about this in some of my talks and have written some blogs about it as well. It's not enough to write code that isn't supposed to access the wrong tenant's data, you need to build the protection into the system; so that it doesn't work at a permission level. An attempt to access the wrong tenant's data shouldn't just be a bug, it should be a failed operation. It just shouldn't be possible. This is where the problem with AppSync direct integrations comes in. When you have AppSync querying DynamoDB, for instance, you grant AppSync specific permission to do so. Those permissions aren't unique to the caller, they are only unique to the specific integration. So if tenant 1 calls the API it looks the same, at the permission level, as if tenant 2 makes the call. Not great for isolation.

The typical solution to this problem is to have your AppSync talk to a Lambda function. Somewhere along the way, you do an STS AssumeRole operation to get credentials specifically for the tenant on which you want to operate and use those credentials to talk to DynamoDB. These credentials are scoped to the tenant, so you can only get data for that tenant. There are some different ways of accomplishing this, but in the end, it comes down to each call to the DynamoDB table being made with credentials specific to the tenant making the request.

Unfortunately, that option isn't available to us with direct integrations. I'd love to see that change, but for now it's just not possible.

## Step Functions Cross Account Access

Sometime back in 2023, the Step Functions team announced a feature that would allow you to run a state machine task from one account and have it access resources in another. It turns out that this feature has a use within the same account, too.

While Step Functions Cross Account Access was design for...well, cross account access, it's really just telling it what role to assume to perform the task. You can use that same mechanism within an account to have the state machine assume a specific role for a task. For example, let's say you want your state machine to assume a role for a specific tenant, with permissions scoped down to just that tenant's data. See where I'm going here? Because the role in the state machine can be dynamic, you can have a Step Function that assumes the role of the specified tenant, and reuse the Step Function across all your tenants, much like you would a Lambda function.

## Step Functions SDK Integrations

One of the great things about Step Functions is that it has literally hundreds of integrations available. You can make calls to most AWS services via the SDK integrations, and with the ability to specify the role you want to be assumed you can call them with the permissions scoped to just the current tenant.

## AppSync and Step Functions

So far we have talked about how to have Step Functions access data for a particular tenant. What we want is for an integration with AppSync that doesn't require a Lambda function. This is where the Express State Machine integration with AppSync comes into play. With Express functions you can make synchronous Step Function calls that run the state machine directly.

There isn't anything new about this feature, so I won't go into the details. The main point is that you can call a Step Function from AppSync and return data from there.

## The Tradeoffs

This may all sound a bit too good to be true. That's probably because there are some tradeoffs to be aware of.

- The first, and probably the most important, is that each tenant must have their own role. Quite often we use a single role, with a dynamic policy, for tenant isolation. This has a number of advantages, not the least of which is that you don't have to manage all the roles. Unfortunately, the Step Functions integration doesn't allow for a dynamic policy (wouldn't that be nice?). The importance of this tradeoff can't be overstated. There is a hard limit of 5000 IAM roles per account. If you expect to have more than maybe 1000 tenants you need to consider how you will manage the role limit. You might look into tenant sharding to help. In addition to the IAM limits, you need to be able to update these roles if and when your application's needs change. There are a number of options here, just know that this is something you have to deal with that isn't an issue when using dynamic policies.
- Another tradeoff here is that there aren't utility functions in either AppSync or Step Functions for unmarshalling DynamoDB formatted data. Interestingly there is a way to marshall the data in AppSync, but the AppSync direct integration automatically unmarshalls the data on the way out, so there isn't a way to do that. What does this mean? A lot of very specific mapping code that has to convert the ugliness of DynamoDB JSON into something a bit more useful.

## Conclusion

AppSync direct integrations are a great way to allow your API to get data without needing a Lambda function. Until recently, these integrations didn't work for multi-tenant SaaS apps. With the introduction of cross-account Step Function tasks, we can now leverage the direct integrations in AppSync and Step Functions to allow us to build a multi-tenant API using AppSync without using a Lambda function, all while still isolating each tenant's data.