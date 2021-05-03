---
layout: post
title: Lambda Retries and Dead Letter Queues
tags:
  - aws
  - serverless
  - Lambda
  - events
  - error handling
  - dlq
  - retry
---

As you may know, I'm a big fan of serverless in AWS. The primary compute component of serverless in AWS is [AWS Lambda](https://aws.amazon.com/lambda/), so as you might imagine, I use it a lot. When using Lambda, I try to follow best practices for retries and dead-letter-queues (DLQs) or error destinations, but there are so many ways to do it I often find myself needing to look them up. So, I thought it might be useful to have a simple guide. Here it is.

To make this easier to use quickly I'm including a quick reference table for each integration or integration type. The table will look something like this:

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Up to 2 | ❌ No | ✅ Yes | ✅ Yes |

I'll do my best to include more information when necessary.

## Asynchronous or Synchronous

There are two ways to invoke a Lambda function. The type of invocation is key to how you handle errors and retries. Understanding the difference between asynchronous and synchronous invocations will help you narrow down the options pretty quickly.

### Asynchronous Invocations

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Up to 2 | ❌ No | ✅ Yes | ✅ Yes |



All [asynchronous invocations](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html) will retry a message up to two times (default). You can use a DLQ on the function, and/or an error destination. Asynchronous invocations do not support DLQs at the integration because the integration returns immediately after Lambda has received the request. The requests are placed in an internal queue that is managed by Lambda.

> Lambda manages the function's asynchronous event queue and attempts to retry on errors. If the function returns an error, Lambda attempts to run it two more times, with a one-minute wait between the first two attempts, and two minutes between the second and third attempts. Function errors include errors returned by the function's code and errors returned by the function's runtime, such as timeouts.

The following integrations asynchronously invoke your Lambda function.

* [CloudWatch Events (aka EventBridge)](https://docs.aws.amazon.com/lambda/latest/dg/services-cloudwatchevents.html)
* [CloudWatch Logs](https://docs.aws.amazon.com/lambda/latest/dg/services-cloudwatchlogs.html)
* [CloudFormation](https://docs.aws.amazon.com/lambda/latest/dg/services-cloudformation.html)
* [CodePipeline](https://docs.aws.amazon.com/lambda/latest/dg/services-codepipeline.html)
* [Cognito (customer sender triggers)](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-Lambda-triggers.html)
* [EC2](https://docs.aws.amazon.com/lambda/latest/dg/services-ec2.html)
* [IoT](https://docs.aws.amazon.com/lambda/latest/dg/services-iot.html)
* [IoT Events](https://docs.aws.amazon.com/lambda/latest/dg/services-iotevents.html)
* [S3](https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html)
* [SNS](https://docs.aws.amazon.com/lambda/latest/dg/with-sns.html)
* [Step Functions (asynchronous invoke)](https://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html)

### Synchronous Invocations

For [synchronous invocations](https://docs.aws.amazon.com/lambda/latest/dg/invocation-sync.html) you won't get a DLQ or error destination at the function itself. For these integrations it's up to the caller to handle retries and errors, so you may get retries and DLQs in some cases, but not others.

> When you invoke a function synchronously, Lambda runs the function and waits for a response. When the function completes, Lambda returns the response from the function's code with additional data, such as the version of the function that was invoked.

> If Lambda isn't able to run the function, the error is displayed in the output.

Here is a list of each integration and how its retries and DLQs work.

#### CLI & SDK

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Some | ❌ No | ❌ No | ❌ No |

[CLI & SDK](https://docs.aws.amazon.com/lambda/latest/dg/invocation-retries.html) invocations can call your function either synchronously or asynchronously. There are some limited cases where the call will be retried, but exceptions thrown by your code will not lead to a retry.

#### API Gateway

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ❌ None | ❌ No | ❌ No | ❌ No |

[API Gateway](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html) invokes your function synchronously. There are no retries when making calls to your function, and the integration does not support a DLQ.

#### Cognito (except custom sender triggers)

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ 2 | ❌ No | ❌ No | ❌ No |

[Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-Lambda-triggers.html) invokes your function synchronously except for custom sender triggers.

> When called, your Lambda function must respond within 5 seconds. If it does not, Amazon Cognito retries the call. After 3 unsuccessful attempts, the function times out.

#### DynamoDB Streams

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Until Expired* | ✅ Yes | ❌ No | ❌ No |

[DynamoDB Streams](https://docs.aws.amazon.com/lambda/latest/dg/with-ddb.html) invoke your function synchronously in batches. If the function returns an error or times out the entire batch will be retried until the message expires by default. The integration is an [event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html), which includes several configuration options to control retries and DLQs. Although the Lambda console does make it seem as though you can configure an error destination, that configuration is really part of the event source mapping.

* BisectBatchOnFunctionError allows you to isolate a single record that is causing a problem. Every error will result in splitting the batch in order to isolate the error. These retries do not count toward the MaximumRetryAttempts.
* DestinationConfig allows you to send failures to either an SQS queue or an SNS topic.
* MaximumRetryAttempts controls the number of times a message can fail before being discarded (or sent to a failure destination if configured). The default value is -1, which will retry until the message expires.

#### Elastic Load Balancing - Application Load Balancer

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ❌ No | ❌ No | ❌ No | ❌ No |

[Application Load Balancer](https://docs.aws.amazon.com/lambda/latest/dg/services-alb.html) invokes your function synchronously. There are no retries when making calls to your function, and the integration does not support a DLQ.

#### Kinesis Firehose

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ User controlled | ❌ No | ❌ No | ❌ No |

[Kinesis Firehose](https://docs.aws.amazon.com/firehose/latest/dev/data-transformation.html) calls your function synchronously. There are options within the integration to control the number of retries. There is no option for a DLQ.

#### Kinesis Streams

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Until Expired* | ✅ Yes | ❌ No | ❌ No |

[Kinesis Streams](https://docs.aws.amazon.com/lambda/latest/dg/with-kinesis.html) invoke your function synchronously in batches. If the function returns an error or times out the entire batch will be retried until the message expires by default. The integration is an [event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html), which includes several configuration options to control retries and DLQs.

* BisectBatchOnFunctionError allows you to isolate a single record that is causing a problem. Every error will result in splitting the batch in order to isolate the error. These retries do not count toward the MaximumRetryAttempts.
* DestinationConfig allows you to send failures to either an SQS queue or an SNS topic.
* MaximumRetryAttempts controls the number of times a message can fail before being discarded (or sent to a failure destination if configured). The default value is -1, which will retry until the message expires.

#### Lex

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ❌ None | ❌ No | ❌ No | ❌ No |

[Lex](https://docs.aws.amazon.com/lambda/latest/dg/services-lex.html) invokes your function synchronously. There are no retries when making calls to your function, and the integration does not support a DLQ.

#### Amazon MQ

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ❌ None | ❌ No | ❌ No | ❌ No |

[Amazon MQ](https://docs.aws.amazon.com/lambda/latest/dg/with-mq.html) invokes your function synchronously. The integration is an [event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html), but does not include any settings for DLQ or retries. Those can be handled within [ActiveMQ](https://activemq.apache.org/message-redelivery-and-dlq-handling) itself.

#### S3 Batch

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Yes | ❌ No | ❌ No | ❌ No |

[S3 Batch](https://docs.aws.amazon.com/lambda/latest/dg/services-s3-batch.html) invokes your function synchronously. There is no option for a DLQ. If the Lambda function returns a `TemporaryFailure` response code, Amazon S3 retries the operation.

#### SQS

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Configured in SQS | ❌ No | ❌ No | ❌ No |

[SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html) invokes your function synchronously. The integration is an [event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html), however, no DLQ or retry options are available on the integration itself. Instead, you configure a redrive policy on the queue itself. This policy controls how many times a message can be received before being discarded or sent to a DLQ.

SQS also handles throttling in a rather unique way. Because the queue management is handled by Lambda it often will request more than your function can process. When this happens, your message will be tried again until the time remaining on the message timeout is less than the function timeout. At that point the message will be allowed to timeout, allowing the message to be retried or discarded based on your redrive policy. These retries do not count toward the message delivery count.

#### Step Functions (synchronous invoke)

|Retries|DLQ at integration|DLQ at function|Error destination at function|
|-------|------------------|---------------|-----------------------------|
| ✅ Controlled in retry configuration | ✅ Controlled as a state | ❌ No | ❌ No |

[Step Functions](https://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html) can invoke a Lambda synchronously. When doing so you can use the [retry](https://states-language.net/spec.html#retrying-after-error) setting to control the retries. Any error logic can be handled in the state machine using the [catch](https://states-language.net/spec.html#fallback-states) setting and sending the message to an SQS queue or SNS topic within the state machine itself.
