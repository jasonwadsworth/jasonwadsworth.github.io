---
layout: post
title: Securing Cross-Account Access in Multi-Tenant SaaS Applications
tags:
  - aws
  - security
  - SaaS
  - AssumeRole
  - STS
---

If you’re building a SaaS solution, it’s critically important that you protect and isolate your customer's data from other customers (often referred to as tenants). For companies building SaaS on AWS, one aspect of their isolation strategy is to connect the data that resides in tenant-owned AWS account(s) with your SaaS application running in your, SaaS provider-owned, AWS accounts.

AWS recommends using the [AWS Security Token Service (STS) API](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html) to make calls to get [temporary credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html) for this type of cross-account access. This API leverages [AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/iam/) roles to provide access between AWS accounts.

But what are you doing to make sure these roles are only being used to connect the correct tenant accounts? In a multi-tenant application, there are two primary concerns for protecting your tenants' accounts; the potential for one tenant, a bad actor, to use the information of another tenant to gain access to that tenant's data using your application, and your own mistakes. In this blog, we examine methods of securing cross-account access using STS to ensure our customers' data is secure and isolated.

## Protection From Bad Actors - The Confused Deputy Problem

When your application needs to access data in your tenants' accounts, you use the [Assume Role API](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) to get temporary credentials. Without some extra protection, this opens the door for a bad actor to take advantage of the [Confused Deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html). I won't go into the problem in detail here, but stated simply, it allows one tenant to access another tenants data simply by knowing the ARN of the role in the other tenant's account. AWS has a solution to combat this problem — the use of an `ExternalId`. There is a [great post from AWS](https://aws.amazon.com/blogs/apn/securely-using-external-id-for-accessing-aws-accounts-owned-by-others/) about how to implement this in your application. One important element to this is that you, the SaaS provider, need to supply the `ExternalId` and make sure it is a unique value in your application. By being the owner of the ID you can be sure that a second tenant cannot use the same ID in their tenant. When a tenant adds an AWS account to your application they must take the ID you supplied and include it in the assume role policy for the role they want to assume. This makes sure the tenant has access to that role.

## Protection From Yourself - The Bad Code Problem

I have a saying — "if your idea of data protection is a where clause in SQL, you aren't protecting my data." That saying doesn't translate perfectly well to this subject, but the point is that you can't rely on your code to protect your tenants' data. You need something more; something that makes sure bad code doesn't lead to a break in isolation.

As much as we try to avoid it, mistakes happen when writing code. We have processes in place to help avoid the mistakes — code reviews, automated tests — but we can't eliminate them entirely. We can, however, plan for them and build a system that fails safely. I talk about this subject a lot, but it's typically talking about things like DynamoDB or S3 data. Data that you, the SaaS provider control. It turns out we can leverage the same sort of techniques to be sure we don't accidentally assume a role for the wrong tenant.

First, let's talk a bit about what the problem is we are trying to solve. Imagine the worst-case scenario where there is hard-coded data in your application that makes its way to production. This hard-coded data uses a single tenant's role ARN and external ID to make calls into their AWS account. Now all your tenants are seeing that one tenant's data. While this example is extreme, there are less extreme ways to have the same, or similar, results. What we need is a way that even bad code can't lead to a break in isolation.

If you've heard me speak on the subject, or have read my blog post on [Multi-tenant Security Implementation](https://jason.wadsworth.dev/multi-tenant-security-implementation/), you know that we don't give our code (typically running in Lambda) permission to access tenant data. Instead, we pass in credentials that are used to access the data. These credentials are created using a dynamic policy that limits access to only that tenant's data. We'll do the same thing here, but instead of accessing data, we'll limit the ability to call the Assume Role API for the specific tenant's role.

To do this, we take advantage of the `iam:ResourceTag` condition. This condition allows you to require that the role being assumed includes a tag with a specific value. For our example, we'll call the tag MyApplicationTenantId. The condition will require that the role being assumed is tagged with a tag called MyApplicationTenantId and a value of that tenant's ID in our application. The dynamic policy looks something like this:

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

> NOTE: You may have noticed that we are allowing all resources. Add specific resources to further restrict that if you'd like, but the condition is good enough for this use case. I've used the path of the role in the past but since the console doesn't allow you to set the path it limits how customers can configure this role.

When the customer creates the role in their AWS account they include the tag `MyApplicationTenantId` on that role and set its value to their tenant ID in your application.

### Setting the Source Identity to Secure AssumeRole

If you're familiar with the [Assume Role API](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) you may be aware of some of its limits, particularly as it relates to [role chaining](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html#iam-term-role-chaining). Role chaining, simply put, is assuming one role and then using those credentials to assume another role. There are other limitations to be aware of if you are going to use role chaining but one that is important to our scenario is that role chaining requires that you have permission to [set the source identity](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html) when calling the API. This permission must exist on both the assume role policy document of the role being assumed as well as the permissions of the role doing the assuming. So for our code to work, we need to add a couple of things.

First, we need to add permissions to the assume role policy document on the role we are assuming. The full policy will look something like this:

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
                    "sts:ExternalId": "my-application-provided-external-id"
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

We also need to add the `sts:SetSourceIdentity` permission to the role doing the assuming. Our dynamic policy now looks something like this:

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

With this in place, you can safely assume your tenants' roles without being concerned you'll get data for the wrong tenant. Even if you have a hard-coded value in your code the permissions of the credentials your code uses won't have permission to assume the role.

## Conclusion

Building multi-tenant SaaS applications comes with important tenant isolation challenges, especially if you connect your application to customer-owned AWS accounts. Being aware of the potential for bad actors or bad code to break that isolation is a key step in understanding how to protect against it. Making sure you’ve taken all steps to secure cross-account access with solutions like the one I've outlined in this blog helps keep your tenants’ data safe and your application secure.
