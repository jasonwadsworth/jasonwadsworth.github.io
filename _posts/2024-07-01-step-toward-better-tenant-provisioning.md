---
layout: post
title: Step Toward Better Tenant Provisioning
tags:
  - aws
  - SaaS
  - CDK
  - Step Functions
---

In a multi-tenant SaaS application, you often need to manage resources that are tenant-specific. Whether it's a tenant-specific role, isolated DynamoDB tables, or per-tenant Cognito user pools, you need to have a way to deploy and update these resources across your application. In this blog, I'll show you how I have approached this problem.

There are three components of managing tenant-specific resources; creating new resources as a tenant is added, updating resources as your application needs changes, and deleting resources when a tenant is deleted.

## The Old Way

![EventBridge -> Lambda -> SDK to create resources](https://jason.wadsworth.dev/images/2024-07-01/eb-Lambda-sdk.png)

In the past, I would use a model that looks something like the image above to create and delete resources. When a new tenant is added to the system an event is sent out and that would trigger a Lambda function. That Lambda function would use the SDK to create the necessary resources. Similarly, a delete would send out an event that would trigger a Lambda function that would use the SDK to delete tenant-specific resources.

There are some problems with this approach.

First, I like to use CDK, specifically [L2 constructs](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html), for all of my infrastructure. The SDK is very different, so there is a cognitive cost of using the SDK. You often need to remember more details, and the structure is very different.

Second, there isn't a place to go see all the resources associated with a tenant. This isn't a big deal when you have one resource per tenant, but as that grows it's nice to be able to go to a single spot to see everything that belongs to that tenant.

Third, while creating new resources when a tenant is created, and deleting them when a tenant is deleted, is pretty straightforward, updating is not. There isn't an event that you can use to trigger the update; at least not one that is a system event. Updates, unlike creates and deletes, are deploy time changes. As the application changes, you need to update the infrastructure of your tenants. That's a very different process than creating and deleting. Updating requires code that is aware of the current state and understands how to go from one to the other. It requires code to handle rolling back into a previous state when something goes wrong. Updates are complicated, and anytime I can remove complicated code I'm going to do it.

## A Better Approach

Hearing others have the same problem, I wanted to find a solution to make things easier. I knew I wanted CDK and CloudFormation to be a part of the solution, and my thoughts quickly went to [Step Functions](https://aws.amazon.com/step-functions/). Could there be an answer there?

Here is what I figured out.

It starts with CDK. I create a Stack in CDK that holds all the resources for a tenant. This stack includes the use of `CfnParameter` to pass in the identifier of the tenant. Any resources that you need to create are added to this stack. The code looks something like this.

```typescript
export class TenantTemplateStack extends Stack {
    constructor(scope: Construct, id: string, props: StackProps) {
        super(scope, id, props);

        const parameterTenantId = new CfnParameter(this, 'TenantId', {
            type: 'String',
            description: 'The ID of the tenant',
        });

        const tenantId = parameterTenantId.valueAsString;

        // any resources you want to provision per tenant go here
    }
}
```

The stack needs to be synthesized and made, so in our tenant management stack I include an S3 bucket where the template will be deployed, I synthesize the above template, and I deploy the synthesized template to the bucket. The keys to this are the use of the `BootstraplessSynthesizer` in the template stack and the `Stage` that I'll use to synthesize it. This creates a sort of CDK Inception, where your "cdk.out" will have your synthesized stack(s) and each stack will have another synthesized stack for your tenant template. Accessing the assembly of the stage allows us to grab the output and push it to S3 using the BucketDeploy construct.

```typescript
export class TenantManagement extends Construct {
    public readonly templateBucketName: string;

    public readonly templateBucketKey: string;

    constructor(scope: Construct, id: string, props: TenantManagementProps) {
        super(scope, id);

        const stack = Stack.of(this);

        const templateBucket = new Bucket(this, 'TemplateBucket', {
            blockPublicAccess: {
                blockPublicAcls: true,
                blockPublicPolicy: true,
                ignorePublicAcls: true,
                restrictPublicBuckets: true,
            },
            objectOwnership: ObjectOwnership.OBJECT_WRITER,
            encryption: BucketEncryption.S3_MANAGED,
            enforceSSL: true,
            publicReadAccess: false,
            versioned: false,
            // important so that updates can be trigger based on this event
            eventBridgeEnabled: true,
        });

        const stage = new Stage(this, 'SynthStage');

        new TenantTemplateStack(stage, 'TenantTemplate', {
            // this allows the synthesis to generate a template without resolving CDK values like account and region
            synthesizer: new BootstraplessSynthesizer(),
        });

        // synthesize the template stack
        const assembly = stage.synth();

        // the stage only has one stack, so it's safe to grab index zero here to get the path of the output
        const templateFullPath = assembly.stacks[0].templateFullPath;

        // the bucket deployment construct will copy the resources in the specified path to S3
        new BucketDeployment(this, 'EachTenantStackDeployment', {
            destinationBucket: templateBucket,
            sources: [Source.asset(dirname(templateFullPath))],
        });
    }
}
```

At this point, I have a template that is being created that can be used for each of our tenants, but I still need to run the template at the right times. I'll create a few Step Functions to do this.

I'll start with the create because it's really hard to test doing an update and delete if you don't first create it. :) The create is triggered in the same way our Lambda function that was making SDK calls was triggered; via a tenant-created event sent to EventBridge.

The flow looks a bit like this:

![EventBridge to Step Functions to CloudFormation](https://jason.wadsworth.dev/images/2024-07-01/eb-sfn-cfn.png)

The detail of the Step Function looks like this:

![Create tenant resources Step Function](https://jason.wadsworth.dev/images/2024-07-01/create-tenant.png)

The Step Function is actually rather simple. It makes a call to CreateStack using the template I uploaded to the S3 bucket. After calling CreateStack it calls DescribeStack in a loop, checking to see that it has completed and failing if the stack fails. This way I can add metric alarms to notify the team if there are failures.

Next, I'll do the delete. Like the create, the delete is triggered from a tenant-deleted event sent to EventBridge. This runs a Step Function that looks a bit like this:

![Delete tenant resources Step Function](https://jason.wadsworth.dev/images/2024-07-01/delete-tenant.png)

This one is a bit more complicated than the create because it first checks to see that the template is in a state that allows it to be deleted. This way you don't end up with errors if you try to delete a stack while it's in the process of being updated.

Finally, the update.

![CDK to S3 to EventBridge to Step Functions to CloudFormation](https://jason.wadsworth.dev/images/2024-07-01/cdk-s3-eb-sfn-cfn.png)

![Update tenant resources Step Function](https://jason.wadsworth.dev/images/2024-07-01/update-tenants.png)

This Step Function is started whenever the template is updated in the S3 bucket. This means that when the deployment of our tenant management sends the updated template to the bucket this Step Function will run, which will automatically update all of your tenant's stacks. The State Machine looks a bit overwhelming, but when broken down into its parts it's pretty easy to understand.

![update tenant resources Step Function with tenant loop highlighted](https://jason.wadsworth.dev/images/2024-07-01/update-tenants-tenant-loop.png)

The first part of this is just getting a list of tenants. I am storing our tenants in a DynamoDB table so I can query the data from there. DynamoDB uses paging, so I have to have some logic to loop over the data and call back into DynamoDB to get the next page.

![Update tenant resources Step Function with the first describe loop highlighted](https://jason.wadsworth.dev/images/2024-07-01/update-tenants-describe-loop.png)

The next bit is checking to see if the tenant's stack can be updated. This is important because a tenant may be in the process of being created when you deploy a new template. If you don't do this loop the update will fail and the new tenant won't get the updates.

![Update tenant resources Step Function with the update stack and describe loop highlighted](https://jason.wadsworth.dev/images/2024-07-01/update-tenants-update-describe-loop.png)

Lastly, the UpdateStack call, and subsequent looping, looks just like our create logic adjusted for an update.

## Conclusion

When you are building a multi-tenant SaaS app it's important to have a strategy for managing any tenant-specific resources you may have. Using Step Functions with the CDK is a great way to manage those updates. With this approach I get to continue to use CDK to model our resources, I have one place to go to see all the resources for a tenant (the CloudFormation stack for that tenant), and the complexities of updating and rolling back changes are managed by CloudFormation.
