---
layout: post
title: Using Step Functions to Eliminate Your NAT Gateway
---

I love serverless, for a lot of reasons. One huge benefit is the cost; if you’re not using it you aren’t paying for it. So, it bothers me whenever I find a need to have some bit of infrastructure that I have to pay for all the time. If you run your lambdas inside a VPC you may know what I’m talking about (also, the title of the article might have given it away). NAT gateways are a necessary evil when your function needs to talk to anything outside the VPC. Or are they?

## VPC Endpoints

Sure, we have [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html) gateway endpoints for S3 and DynamoDB, and you should definitely use them (because they’re free). We even have [PrivateLink, interface endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html) for many of the other AWS services. PrivateLink can also be used to talk to third party APIs, if they support it. PrivateLink, however, is not free, so be aware of the cost before using them. That’s not to say you shouldn’t use them, security is important, and they provide a layer that is really quite nice.

## The NAT Gateway

But what about that third party API you use that doesn’t have PrivateLink support? If you are making a call to that API in your lambda, and that lambda lives inside a VPC, you need a NAT gateway. You just added at least $30/month to your bill, more if you want the resiliency of multiple availability zones. That might not seem like a big deal, especially if you are a company with money to spend. If you are just getting started with an idea and every penny matters, or maybe if you need dozens of accounts, and that $30 quickly becomes $1000, you might be looking for options to get rid of that NAT gateway. A knee jerk reaction would be to run all your lambdas outside a VPC. There are some cases where that might be doable, especially if you are 100% serverless, but just because you can doesn’t mean you should, and even if it’s technically possible, it may not be compliant with requirements you have. So, you may end up with something like this:

![NAT Gateway Example]({{ site.baseurl }}/images/2021-03-14/nat-gateway.png)

So, what can you do?

## Enter Step Functions

[Step Functions](https://aws.amazon.com/step-functions/) allow you to run different bits of code and combine them together into a sort of work flow. Not long ago they were added as an [integration for API gateway](https://aws.amazon.com/blogs/compute/introducing-amazon-api-gateway-service-integration-for-aws-step-functions/), opening up a whole new world of possibilities. So, how do Step Functions allow you to remove the NAT gateway?

When you need to make a call out to a third-party API there some inputs you need to send, and likely some outputs you need from the response (if nothing else, a confirmation of success). Typically, you’d make that call inside your code. You might do some work, maybe look up some values, make the API call, and then do something with the results before ultimately returning a response to the end user. Step Functions allow you to break up that logic into independent functions. You might first have a function that does some initial work, a second that calls the third-party API, and a final one that processes the results and returns a response to the caller. Step Functions can stitch that all together for you (and may even be able to do some things without a lambda). How does that help you? Well, by isolating the call to the third-party API you are able to isolate the function that needs internet access. You can have that one function live outside the VPC, keeping the rest of your functions safe inside their cozy little isolated network. Step Functions will send the results of the call to the third-party API to your function that’s inside the VPC. It might look something like this:

![Step Functions Example]({{ site.baseurl }}/images/2021-03-14/step-functions.png)

You may be thinking that it looks a lot more complicated, and complexity is something you want to avoid. I won't say there isn't anything here that is more complex, but keep in mind that what Step Functions is doing here is the same logic you'd have in your code; you've just moved it to the state machine instead.

## Cost

The whole reason you are interested in this is to reduce cost and adding Step Functions to the flow does increase the per request cost; keep that in mind as you consider this. If you are processing hundreds of requests per second, I'm pretty sure this is not the answer for you, but hopefully you fall into the "have money to spend" category. For the rest of you, it takes quite a lot of requests to reach the point of costing $30. You may also find that you can save a little because the function that is calling the third-party API can probably run on 128MB, even if you need more than that for the rest of your code. Jeremy Daly wrote a great post on that subject: [Serverless Tip: Don’t overpay when waiting on remote API calls](https://www.jeremydaly.com/serverless-tip-dont-overpay-when-waiting-on-remote-api-calls/)

## Closing

One final thought. I've focused this post on API Gateway integration, but this is really true of a lot of things that need to make calls out to third party APIs. If you are using [EventBridge](https://aws.amazon.com/eventbridge/) it can call Step Functions directly too.
