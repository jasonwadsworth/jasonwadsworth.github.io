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

Before getting into the details of how this solution works let's be sure we understand the problem we are trying to solve. If you are building a multi-tenant SaaS application your application must be built in such a way that one tenant isn't able to access another tenant's data. I talk about this in some of my talks and have written some blogs about it as well. It's not enough to write code that isn't supposed to access the wrong tenant's data, you need to build the protection into the system; so that the code doesn't work at a permission level if it attempts to access the wrong tenant. An attempt to access the wrong tenant's data shouldn't just be a bug, it should be a failed operation. It just shouldn't be possible. This is where the problem with AppSync direct integrations comes in. When you have AppSync querying DynamoDB, for instance, you grant AppSync specific permission to do so. Those permissions aren't unique to the caller, they are only unique to the specific integration. So if tenant 1 calls the API it looks the same, at the permission level, as if tenant 2 makes the call. Not great for isolation.

The typical solution to this problem is to have your AppSync talk to a Lambda function. Somewhere along the way, you do an STS AssumeRole operation to get credentials specifically for the tenant on which you want to operate and use those credentials to talk to DynamoDB. These credentials are scoped to the tenant, so you can only get data for that tenant. There are some different ways of accomplishing this, but in the end, it comes down to each call to the DynamoDB table being made with credentials specific to the tenant making the request. If you were to request data for another tenant the permissions wouldn't allow it.

![Typical SaaS solution]({{ site.baseurl }}/images/2024-03-05/AppSync-Lambda.png)

Unfortunately, that option isn't available to us with direct integrations. I'd love to see that change, but for now, it's just not possible.

## Step Functions Cross Account Access

Sometime back in 2023, the [Step Functions team announced a feature that would allow you to run a state machine task from one account and have it access resources in another](https://aws.amazon.com/blogs/compute/introducing-cross-account-access-capabilities-for-aws-step-functions/). It turns out that this feature has a use within the same account, too.

While Step Functions Cross Account Access was designed for...well, cross-account access, it's really just telling Step Functions what role to assume to perform the task. You can use that same mechanism within an account to have the state machine assume a specific role for a task. For example, let's say you want your state machine to assume a role for a specific tenant, with permissions scoped down to just that tenant's data. See where I'm going here? Because the role in the state machine can be dynamic, you can have a Step Function that assumes the role of the specified tenant, and reuse the Step Function across all your tenants, much like you would a Lambda function.

## Step Functions SDK Integrations

One of the great things about Step Functions is that it has literally hundreds of integrations available. You can make calls to most AWS services via the [SDK integrations](https://docs.aws.amazon.com/step-functions/latest/dg/supported-services-awssdk.html), or use the [optimized integrations](https://docs.aws.amazon.com/step-functions/latest/dg/connect-supported-services.html) for a smaller set of integrations. And with the ability to specify the role you want to be assumed you can call them with the permissions scoped to just the current tenant.

## AppSync and Step Functions

So far we have talked about how to have Step Functions access data for a particular tenant. What we want is for an integration with AppSync that doesn't require a Lambda function. This is where the [Express State Machine integration with AppSync](https://aws.amazon.com/blogs/mobile/invoking-aws-step-functions-short-and-long-running-workflows-from-aws-appsync/) comes into play. With Express functions you can make synchronous Step Function calls that run the state machine directly.

![AppSync to Step Functions]({{ site.baseurl }}/images/2024-03-05/AppSync-StepFunctions.png)

There isn't anything new about this feature, so I won't go into the details. The main point is that you can call a Step Function from AppSync and return data from there.

## Putting it All Together

Now that we understand that we can use Step Functions to make direct API calls with a tenant-specific IAM role, and we can call Step Functions from AppSync, how do we put this together?

To get this all to work safely we need to step back a bit and talk about the authentication. If you've read any of my previous posts on SaaS you've seen that I'll typically have a [custom authorizer](https://aws.amazon.com/blogs/mobile/appsync-lambda-auth/) that not only validates the user (typically via JWT validation) but also obtains credentials for that user's tenant. In this case, we'll take a slightly different approach. Because Step Functions will be assuming the role for us we don't need to provide credentials, but we do want to provide the role that the Step Function should use. We'll add the tenant-specific role ARN to the `resolverContext` of the custom authorizer. This value will be available as part of the input to your Step Function. Specifically, you can access anything that you put in the `resolverContext` at `$.identity.resolverContext` in your state machine.

> Tip:
> You can access the input arguments of your Step Functions state machine from anywhere in your state machine by going through the context object, which is accessible by using the `$$` notation. More information about the [context object can be found here](https://docs.aws.amazon.com/step-functions/latest/dg/input-output-contextobject.html).

You may be tempted to make the name of the role something like `TenantRole<tenant id>` so that you can easily put together the role name anywhere that needs it. Doing so is not advisable because it can lead to the very problem we are trying to avoid. If the Step Function is determining the role to assume then it can make a mistake and use the wrong role. This, combined with a mistake about which tenant to access, allows the wrong tenant's data to be returned. Instead, you should have the tenant's role names be somewhat random. I like to use a ULID and store the name of the role with information about the tenant.

There is one more thing to keep in mind here. You probably want to limit what roles your Step Function can assume, but you don't know the names of the roles. I like to take advantage of the `path` of the role to make this easier. All my tenant-specific roles have a path that is something like `/tenant-role/`, which allows me to create an IAM policy that only allows assuming roles that are at that path. You can also limit what services can assume the role via the assume role policy document. Just be sure to keep in mind all the places you might want to assume this role (it's probably not just Step Functions).

## The Tradeoffs

This may all sound a bit too good to be true. That's probably because there are some tradeoffs to be aware of.

- The first, and probably the most important, is that each tenant must have their own role. Quite often we use a [single role, with a dynamic policy, for tenant isolation](https://aws.amazon.com/blogs/apn/isolating-saas-tenants-with-dynamically-generated-iam-policies/). This has several advantages, not the least of which is that you don't have to manage all the roles. Unfortunately, the Step Functions integration doesn't allow for a dynamic policy (wouldn't that be nice?). The importance of this tradeoff can't be overstated. There is a hard limit of 5000 IAM roles per account. If you expect to have more than maybe 1000-2000 tenants you need to consider how you will manage the role limit. You might look into tenant sharding to help (Bill Tarr talks about this a bit in his talk [SaaS architecture pitfalls: Lessons from the field](https://youtu.be/sPk_-wdbl8U?si=yD06LKEDu4dGAUhB) from re:Invent 2023). In addition to the IAM limits, you need to be able to update these roles if and when your application's needs change. There are several options here, just know that this is something you have to deal with that isn't an issue when using dynamic policies.
- Another tradeoff here is that there aren't utility functions in either AppSync or Step Functions for unmarshalling DynamoDB formatted data. Interestingly there is a way to marshall the data in AppSync, but the AppSync direct integration automatically unmarshalls the data on the way out, so there isn't a way to do that. What does this mean? A lot of very specific mapping code that has to convert the ugliness of DynamoDB JSON into something a bit more useful.

## Conclusion

AppSync direct integrations are a great way to allow your API to get data without needing a Lambda function. Until recently, these integrations didn't work for multi-tenant SaaS apps. With the introduction of cross-account Step Function tasks, we can now leverage the direct integrations in AppSync and Step Functions to allow us to build a multi-tenant API using AppSync without using a Lambda function, all while still isolating each tenant's data.