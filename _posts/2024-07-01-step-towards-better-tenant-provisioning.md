---
layout: post
title: Step Towards Better Tenant Provisioning
tags:
  - aws
  - SaaS
---



In a multi-tenant SaaS application you often need to manage resources that are tenant specific. Whether it's a tenant specific role, isolated DynamoDB tables, or per tenant Cognito user pools, you need to have a way to deploy and update these resources across your application. In this blog I'll show you how we have approached this problem.

There are three components of managing tenant specific resources; creating new resources as a tenant is added, updating resources as your application needs changes, and deleting resources when a tenant is deleted.

## The Old Way

![EventBridge -> Lambda -> SDK to create resources]()

In the past I would use a model that looks something like the image above to create and delete resource. When a new tenant is added to the system an event is send out and that would trigger a lambda function that would use the SDK to create the necessary resources. Similarly, a delete would send out an event that would trigger a lambda function that would use the SDK to delete tenant specific resources.

There are some problems with this approach.

First, we use CDK, specifically [L2 constructs](), for all of our infrastructure. The SDK is very different, so there is a cognitive cost of using the SDK. You often need to remember more details, and the structure is very different.

Second, there isn't a place to go see all the resources associated with a tenant. This isn't a big deal when you have on resource per tenant, but as that grows it's nice to be able to go to a single spot to see everything that belongs to that tenant.

Third, while creating new resources when a tenant is created, and deleting them when a tenant is deleted, is pretty straight forward, updating is not. There isn't an event that you can use to trigger the update; at least not one that is a system event. Updates, unlike creates and deletes, are deploy time changes. As the application changes you need to update the infrastructure of your tenants. That's a very different process than creating and deleting. Updating requires code that is aware of the current state and understanding how to go from one to the other. It requires code to handle rolling back into a previous state when something goes wrong. Updates are complicated, and anytime I can remove complicated code I'm going to do it.

##

Hearing others have the same problem, I wanted to find a solution to make things easier. I knew I wanted CDK and CloudFormation to be a part of the solution, and my thoughts quickly went to Step Functions. Could there be an answer there? I felt like there was, but life is busy and I didn't get around to looking into it. We continued on with the approach we'd always been using, despite its complexities.

At some point I became aware of some capabilities withink CDK and it got my mind back on the problem. I really wanted to have a better solution.

Here is what I figured out.

It starts with CDK. We create a Stack in CDK that holds all the resources for a tenant. This stack includes the use of `CfnParameter` to pass in the identifier of the tenant. Any resources that you need to create are added to this stack.

We need to synthesize the stack and make it available, so in our tenant management stack we include an S3 bucket where the template will be deloyed, we synthesize the above template, and we deploy the synthesized template to the bucket. The keys to this are the use of the BootstraplessSynthesizer in the template stack, and the Stage that we'll use to synthesize it. This creates a sort of CDK Inception, where your cdk.out will have your synthesized stack(s) and each stack will have another sythesized stack for your tenant template. Accessing the assembly of the stage allows us to grab the output and push it to S3 using the BucketDeploy construct.

At this point we have a template that is being created for each of our tenants, but we still nedd to run the template at the right times. We'll create a few Step Functions to do this.

We'll start with the create, because it's really hard to test doing an update and delete if you don't first create it. :) The create is triggered in the same way our lambda function that was making SDK calls was triggered; via a tenant created event sent to EventBridge.

The flow looks a bit like this:

![create tenant resources visual including the event bridge rule and the step functions flow]()

The step function is actually rather simple. It calls CreateStack using the template we uploaded to the S3 bucket. After calling CreateStack it calls DescribeStack in a loop, checking to see that is has completed and failing if the stack fails. This way we can add metric alarms to notify the team if there are failures.

Keeping things on the simple side, we'll do the delete next. Just like the create, the delete is triggered from an tenant deleted event sent to EventBridge. This runs a Step Function that looks a bit like this:

![delete tenant resources visual including the event bridge rule and the step functions flow]()

You'll notice that this looks very similar to the create, except that it's doing a delete and checking that the stack deletes correctly.

The more complicated one is the update, but even this one isn't bad.

![update tenant resources visual including the event bridge rule and the step functions flow]()

This step function is started whenever the template is updated in the S3 bucket. This means that after the deployment sends the updated template to the bucket this step function will run, which will automatically update all of your tenant's stacks. The State Machine looks a bit overwhelming, but when we break down its parts it's pretty easy to understand.

![same as above but highlight list of the tenants in a loop]()

The first part of this is just getting a list of tenants. We are storing our tenants in a DynamoDB table so we can query the data from there. DynamoDB using paging, so we have to have some logic to loop over the data and call back into DynamoDB to get the next page.

![same as above but highlighting the first describe loop]()

The next bit is checking to see if the tenant's stack can be updated. This is important because a tenant may be in the process of being created when you deploy a new template. If you don't do this loop the update would fail and the new tenant wouldn't get the updates.

![same as above but highlighting the update and loop]()

Lastly, the UpdateStack call, and subsequent looping, looks just like our create and delete logic adjusted for an update.

## Conclusion

When you are building a multi-tenant SaaS app it's important to have a strategy for managing any tenant specific resources you may have. Using Step Functions with the CDK is a great way to manage those updates.
