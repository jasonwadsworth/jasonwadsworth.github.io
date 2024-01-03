---
layout: post
title: Assuming Customer Roles in a Multi-Tenant SaaS Application
tags:
  - aws
  - security
  - SaaS
  - AssumeRole
  - STS
---

SaaS applications are a big part of business in today's software landscape. If you're building a SaaS application you probably are aware of the importance of protecting your customer's data from other customers (often referred to as tenants). If you happen to be building an application on AWS that allows your tenants to integrate their AWS account(s) with your application you are probably using the Assume Role API to get temporary credentials for each account (if you're not, you should be). But what are you doing to make sure that role is only being used to connect the correct tenant?

## The Problem

If you're using the Assume Role API you are, hopefully, using the external ID as part of the assume role document. This external ID allows you to limit who can assume the role by requiring the caller to supply the external ID during the process of getting temporary credentials. It's a solution to the [Confused Deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html). In a multi-tenant SaaS application, there is yet another problem that can happen.

Suppose a customer grants your application access to assume a role in their AWS account. They give you the ARN of the role and the external ID and now your application can make calls to gather data from that account (hopefully with well-scoped permissions). According to AWS guidance, the external ID is not a secret, which means it's visible to anyone who can see the assume role policy document and isn't likely stored as a secret by you. Even if it were, secrets have their way of becoming known to the wrong person.

Now let's say another customer starts using your application. This customer is a bad actor and hopes to leverage your application to get access to another customer's data. To do this they need only know the ARN and external ID of the role in the other customer's account. By using this data they can add the other customer's AWS account to their tenant in your application. When your application goes to assume the role it will work because that role is allowed access by ANY tenant in your application; it doesn't know any better.

As a provider of a SaaS application, you certainly don't want your software to be part of an attack vector for your customers. What can you do to stop this from happening?

Your first thought may be to store the external ID in an encrypted form. While this isn't a bad idea it only keeps your application from being the source of the information needed to become an attack vector; it doesn't actually stop the attack. In other words, if the bad actor gets the external ID from somewhere else your application can still be used to get the other tenant's data.

One option that can work pretty well is to limit integrations into other AWS accounts to a single tenant in your system. With this approach, the bad actor would attempt to add the other tenant's account and that would fail because it's already integrated with the other tenant. Your application would need to enforce this restriction. This solution has a couple of problems.

First, you may need to connect the same account to your application more than once. Perhaps the customer uses the same account for production and testing and wants to have a separate account in your application for testing. You can try to make your application robust enough to not need to have another account but you still may find that some customers really want it that way.

Second, a customer may remove an account from your application but not remove the role in their AWS account. In this case, the bad actor is free to use the information to add the account to your application and start to collect data.

## The Solution

What you need is a way to make sure that the role being assumed is only being used by the tenant who should be using it. We accomplish this using a dynamic policy, taking advantage of conditions on policies, and the use of tags.

If you're not familiar with dynamic policies I encourage you to check out my blog post on [Multi-tenant Security Implementation](https://jason.wadsworth.dev/multi-tenant-security-implementation/). In it I give an example of how we used dynamic policies are part of our multi-tenant solution at a former employer. As a quick refresher, dynamic policies allow you to further limit the permissions of an Assume Role API call by passing in an additional policy. Your first thought may be that you can just add the dynamic policy to the call when you assume your tenant's role. Unfortunately, all that would do is limit how you use that role. It wouldn't keep you from assuming it. And since you can't expect your customers to change their tagging policy to help you secure your system that option is probably off the table. So what can you do?

Just like when we want to get data from a DynamoDB table, when we assume a tenant's role we first assume a role that restricts what data we can access. In the DynamoDB case, we use the `dynamodb:LeadingKeys` condition to be sure the partition key includes the tenant's ID. For our call to the Assume Role API, we take advantage of the `iam:ResourceTag` condition. This condition allows you to require that the role being assumed includes a tag with a specific value. For our example, we'll call the tag MyApplicationTenantId. The condition will require that the role being assumed is tagged with a tag called MyApplicationTenantId and a value of that tenant's ID in our application. It looks something like this:

```TypeScript
{
	Action: ['sts:AssumeRole'],
	Effect: 'Allow',
	Resource: ['*'],
	Condition: {
	    StringEquals: {
	        'iam:ResourceTag/MyApplicationTenantId': tenantId,
	    },
	},
},
```

> NOTE: You may have noticed that we are allowing all resources. You can add some things to further restrict that if you'd like, but the condition should be good enough. I've used the path of the role in the past but since the console doesn't allow you to set the path it limits how customers can configure this role. Always apply least-privilege permissions in your production environments.

When the customer creates the role in their AWS account they include the tag `MyApplicationTenantId` on that role and set its value to their tenant ID in your application.

If you're familiar with the Assume Role API you may be aware of some of its limits, particularly as it relates to what is referred to as role chaining. Role chaining, simply put, is assuming one role to then assume another role. There are several limitations to be aware of if you are going to use role chaining. One that is important to our scenario is that role chaining requires that you have permission to set the source identity when calling the API. This permission must exist on both the assume role policy document of the role being assumed as well as the permissions of the role doing the assuming. So for our code to work, we need to add a couple of things.

First, we need to add permissions to the assume role policy document. The full policy will look something like this:

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::012345678912:root"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "my-not-so-secret-external-id"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::012345678912:root"
            },
            "Action": "sts:SetSourceIdentity"
        }
    ]
}
```

It's important to note that you can't include the condition statement on `sts:SetSourceIdentity`, but that's okay because it's only used in the context of assuming a role.

We also need to add the `sts:SetSourceIdentity` permission to the role doing the assuming. Our dynamic policy code now looks something like this:

```TypeScript
{
    Action: ['sts:AssumeRole'],
    Effect: 'Allow',
    Resource: ['*'],
    Condition: {
        StringEquals: {
            'iam:ResourceTag/MyApplicationTenantId': tenantId,
        },
    },
},
{
    Action: ['sts:SetSourceIdentity'],
    Effect: 'Allow',
    Resource: ['*'],
},

```

With this in place, a bad actor can no longer leverage your application to gain access to another tenant's data. If the bad actor tries to use the role ARN and external ID it will fail because the dynamic policy generated for the bad actor's tenant will not have the correct tenant ID in the tag.

## Conclusion

Building multi-tenant SaaS applications can be rewarding work, but it comes with its own set of challenges. Being aware of how your application can be used as an attack vector to your tenants' data is a crucial first step in understanding how to protect it. Once you understand it you can find solutions, like the one I've just outlined, to keep your tenants safe and your application secure.