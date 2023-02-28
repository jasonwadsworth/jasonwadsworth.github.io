---
layout: post
title: Solving the DynamoDB EventBridge Pipes Problem
tags:
  - aws
  - serverless
  - DynamoDB
  - EventBridge Pipes
---

I was really excited when AWS announced [EventBridge Pipes](https://aws.amazon.com/eventbridge/pipes/) at re:Invent last year. This was going to simplify all the CDC (change data capture) code I find myself writing, and probably reduce my Lambda spend.

At first, everything was going great. I was able to create EventBridge events directly from my DynamoDB stream records with some simple JSON path. Then I ran into a problem, and I wasn't alone.

## The Problem

Everything worked great when you have simple records in DynamoDB, and even complex objects would be easy enough. Where things fell apart was when I had a list. It wasn't a complicated record. The data looks like this:

```JSON
{
	"id": "ABCDEFG",
	"firstName": "Jason",
	"lastName": "Wadsworth",
	"email": "jasonwadsworth@outlook.com",
	"groups": ["Administrator"]
}
```

In DynamoDB it ends up looking like this:

```JSON
{
	"id": {
		"S": "ABCDEFG"
	},
	"firstName": {
		"S": "Jason"
	},
	"lastName": {
		"S": "Wadsworth"
	},
	"email": {
		"S": "jasonwadsworth@outlook.com"
	},
	"groups": {
		"L": [
			{
				"S": "Administrator"
			}
		]
	}
}
```

Now, if you've played with EventBridge Pipes you know that you can do a bit of a transform in target, via the input template. It's a little odd to work with, but it gets the job done. The input template for the above would end up looking something like this (I intentionally left out the `groups` because...well, that's the problem).

```JSON
{
	"id": <$.dynamodb.NewImage.id.S>,
	"firstName": <$.dynamodb.NewImage.firstName.S>,
	"lastName": <$.dynamodb.NewImage.lastName.S>,
	"email": <$.dynamodb.NewImage.email.S>
}
```

Okay, so what about the groups? Well, turns out that this syntax only supports _some_ of JSON path, and it doesn't help here. With the help of some others I tried this, but it didn't work.

```JSON
{
	"id": <$.dynamodb.NewImage.id.S>,
	"firstName": <$.dynamodb.NewImage.firstName.S>,
	"lastName": <$.dynamodb.NewImage.lastName.S>,
	"email": <$.dynamodb.NewImage.email.S>,
	"groups": <$.dynamodb.NewImage.groups.L[*].S>,
}
```

## The Solution

After being very frustrated by this I felt there had to be a path forward. Turns out there is. The solution is in the enrichment of EventBridge Pipes. One of the enrichment options is Step Functions Express State Machines. After some trial and error I came up with the following solution (code is in CDK).

```TypeScript
const userCreatedEnrichment = new StateMachine(this, 'UserCreatedEnrichment', {
	definition: new Map(this, 'UserCreatedEnrichmentMap', {}).iterator(
		new Pass(this, 'UserCreatedEnrichmentPass', {
			parameters: {
				'id.$': '$.dynamodb.NewImage.id.S',
				'email.$': '$.dynamodb.NewImage.email.S',
				'firstName.$': '$.dynamodb.NewImage.firstName.S',
				'groups.$': '$.dynamodb.NewImage.groups.L[*].S',
				'lastName.$': '$.dynamodb.NewImage.lastName.S',
			},
		}),
	),
	stateMachineType: StateMachineType.EXPRESS,
});

const pipeRole = new Role(this, 'PipeRole', {
	assumedBy: new ServicePrincipal('pipes.amazonaws.com'),
	inlinePolicies: {
		sourcePolicy: new PolicyDocument({
			statements: [
				new PolicyStatement({
					resources: [table.tableStreamArn],
					actions: ['dynamodb:DescribeStream', 'dynamodb:GetRecords', 'dynamodb:GetShardIterator', 'dynamodb:ListStreams'],
				}),
			],
		}),
		enrichmentPolicy: new PolicyDocument({
			statements: [
				new PolicyStatement({
					resources: [userCreatedEnrichment.stateMachineArn],
					actions: ['states:Start*'],
				}),
			],
		}),
		targetPolicy: new PolicyDocument({
			statements: [
				new PolicyStatement({
					resources: [defaultEventBus.eventBusArn],
					actions: ['events:PutEvents'],
				}),
			],
		}),
	},
});


new CfnPipe(this, 'UserCreatedPipe', {
	description: 'Sends UserCreated events',
	roleArn: pipeRole.roleArn,
	source: table.tableStreamArn,
	target: defaultEventBus.eventBusArn,
	sourceParameters: {
		dynamoDbStreamParameters: {
			startingPosition: 'LATEST',
			batchSize: 1,
		},
	},
	enrichment: userCreatedEnrichment.stateMachineArn,
	targetParameters: {
		eventBridgeEventBusParameters: {
			detailType: 'UserCreated',
			source: `MySource`,
		},
	},
});
```

The key here is that Step Functions DO support full JSON path. So by passing the raw data to a state machine I was able to manipulate the data exactly how I wanted it. Sure, it's an extra step, and it would be nice if EventBridge Pipes would fix it, but this is still better than writing more Lambda code.