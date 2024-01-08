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

But what are you doing to make sure these roles are only being used to connect the correct tenant accounts? In this blog, we examine methods of securing cross-account access using STS to ensure our customers' data is secure and isolated.

## Challenges of Assuming Cross-Account Roles

In a multi-tenant application, you have to be aware of the potential for one tenant to use the information of another tenant to gain access to that tenant's data, using your application. AWS best practices have a solution to this problem, known as the [Confused Deputy problem](https://aws.amazon.com/blogs/apn/securely-using-external-id-for-accessing-aws-accounts-owned-by-others/). You can read the details in the linked article, but the short version of it is for you, the SaaS provider, to generate the external ID for your customers and have them add that ID to the assume role policy of the role you are going to assume. If a tenant attempts to use a role for an account they don't own your application will fail to assume that role because the external ID you send won't match the one the real customer put in their assume role policy.  As long as your application controls the external ID, and you are making sure the ID is unique, you can be sure one tenant can't maliciously use another tenant's role to access data that doesn't belong to them.

In a SaaS application, I like to take things a step further. Keeping bad actors at bay is important, and following AWS best practices for security will help with that, but what about your application itself? I have a saying -- "if your idea of data protection is a where clause in SQL, you aren't protecting my data."

What do I mean by that? Well, consider the most extreme example. What if, for some reason, your application has a hard-coded value for what tenant to access? Your customer goes to view some data and, instead of seeing their data, they see the data of the tenant hard-coded into your application. This example is extreme, but it's also not unheard of. How often have engineers edited code while debugging a problem and forgotten to revert that change? And what about the missing where clause that someone doesn't notice while testing because of flaws in the test data? There are several ways that code can do something you didn't intend. It's important that you plan for this and make sure that bad code doesn't lead to leaked data.

I talk about how to do this with data in my blog post on [Multi-tenant Security Implementation](https://jason.wadsworth.dev/multi-tenant-security-implementation/). In it, I give an example of how we generate temporary credentials for each tenant, by assuming a role in _our_ account and using dynamic policies, to access each tenant's data, rather than using the permissions of the Lambda. By doing this we can be sure that bad code doesn't lead to an issue because the bad code would fail if the tenant in the code didn't match the tenant in the temporary credentials. The  example in that blog is for getting data from a DynamoDB table. For DynamoDB, we use the `dynamodb:LeadingKeys` condition to be sure the partition key includes the tenant's ID. We need a way to do something similar when assuming a customer's role.

To do this, we take advantage of the `iam:ResourceTag` condition. This condition allows you to require that the role being assumed includes a tag with a specific value. For our example, we'll call the tag MyApplicationTenantId. The condition will require that the role being assumed is tagged with a tag called MyApplicationTenantId and a value of that tenant's ID in our application. It looks something like this:

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

With this in place, you can safely assume your tenants' roles without being concerned you'll get data for the wrong tenant.

## Conclusion

Building multi-tenant SaaS applications comes with important tenant isolation challenges, especially if you connect your application to customer-owned AWS accounts. Being aware of the potential for your application to be used as an attack vector to your tenants' data is a crucial first step in understanding how to protect it. Making sure you’ve taken all steps to secure cross-account access with solutions like the one I've outlined in this blog helps keep your tenants’ data safe and your application secure.
