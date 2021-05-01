---
layout: post
title: Delayed Event Processing - Part 2
tags:
  - aws
  - serverless
  - events
---

In my [previous post](../delayed-event-processing-part-1) I showed you how you can handle multiple events but only trigger downstream processes in batches. There was one catch to that processing; everything was delayed. What if you want to respond immediately to the first event, but then delay the next time you process until at least some time has passed? Today I'll expand upon the previous post to do just that.

We'll use the same data as the previous post, as well as the same matching and unique value criteria. Jump over to the previous post to see how that works, but here is the data for a refresher.

```json
[
    {
        "level": "WARNING",
        "message": "This is your first warning",
        "receivedAt": "2020-11-14T13:00:00.00000Z",
        "service": "Service A"
    },
    {
        "level": "ERROR",
        "message": "This is an error",
        "receivedAt": "2020-11-14T13:05:00.00000Z",
        "service": "Service A"
    },
    {
        "level": "ERROR",
        "message": "This is an error",
        "receivedAt": "2020-11-14T13:05:00.00000Z",
        "service": "Service B"
    },
    {
        "level": "WARNING",
        "message": "This is your second warning",
        "receivedAt": "2020-11-14T13:05:00.00000Z",
        "service": "Service A"
    },
    {
        "level": "WARNING",
        "message": "This is your first warning",
        "receivedAt": "2020-11-14T13:05:00.00000Z",
        "service": "Service C"
    }
]
```

We'll do the same matching, and insert the same records as we did in the previous post, but we'll add one new record type and change how we calculate the scheduled time.

## The Run Status Record

To make this all work we need a new record that allows us to trigger quickly. This will track the last time we ran for a given service, allowing us to adjust the delay based on the last time we ran, rather than just setting it based on the current time.

You'll recall that in the previous post our table ended up looking like this after the above data was processed:

|Record #|Level  |Service|Time Received|Scheduled Time|
|--------|-------|-------|-------------|--------------|
|1       |Warning|A      |13:00        |13:15         |
|2       |Error  |A      |13:05        |13:10         |
|3       |Error  |B      |13:05        |13:10         |
|4       |Warning|A      |13:05        |13:20         |
|5       |Warning|C      |13:05        |13:20         |

The new table will have some extra rows and columns, but to really see how this works we have to show the table as we process the data.

We receive the first record at 13:00, resulting in the following:

|Record #|Level  |Service|Time Received|Scheduled Time|
|--------|-------|-------|-------------|--------------|
|1       |Warning|A      |13:00        |13:00         |

Right from the start we see a difference; notice that the scheduled time is 13:00, not 13:15, like it was before. That's because of our new logic. Now, when we receive an event we go and find the record indicating the last time it was run, and set the delay based on that time, not the current time. Since there was no record (this is the first event we received for this service) the default behavior is to set it to now.

Now the scheduled processor runs sometime in the next minute and after it runs the table looks like this (we'll assume it started at 13:01):

|Record #|Level  |Service|Time Received|Scheduled Time|Last Run Time|
|--------|-------|-------|-------------|--------------|------------|
|1       |       |A      |             |              |13:01|

This is our new run status record. Every time the scheduler runs it updates the last run time to the current time. Nothing too tricky here.

> Tip: If you want to keep from having a bunch of records you no longer need you can add a [TTL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) to the status record so DynamoDB will automatically delete it for you. Just be sure the TTL is at least as far out as your largest delay time.

Okay, fast-forward a few minutes to 13:05, when we get several new records. Here is what the table would look like right after getting those records:

|Record #|Level  |Service|Time Received|Scheduled Time|Last Run Time|
|--------|-------|-------|-------------|--------------|------------|
|1       |       |A      |             |              |13:01|
|2       |Error  |A      |13:05        |13:06         |
|3       |Error  |B      |13:05        |13:05         |
|4       |Warning|A      |13:05        |13:16         |
|5       |Warning|C      |13:05        |13:05         |

What's most interesting here is that the records for services B and C have a scheduled time of 13:05, but the records for service A are out into the future. Notice, however, that even those aren't as far out as they were before.

So what happened here? Well, just like with the first record for service A, the first records for services B and C didn't have a status record and so the times were set to "now". The records for service A, however, did have a status record. For those records we set the next time based on the delay and the last run time, rather than the delay and now. So record 2 in the table gets set to five (5) minutes past the last run time (which was 13:01), and record 4 gets set to 15 minutes past the last run time.

The next time the scheduler runs it will end up looking like this (assuming it runs just before 13:06)

|Record #|Level  |Service|Time Received|Scheduled Time|Last Run Time|
|--------|-------|-------|-------------|--------------|------------|
|1       |       |A      |             |              |13:01|
|2       |Error  |A      |13:05        |13:06         |
|3       |       |B      |             |              |13:05|
|4       |Warning|A      |13:05        |13:16         |
|5       |       |C      |             |              |13:05|

And a minute later it would look like this:

|Record #|Level  |Service|Time Received|Scheduled Time|Last Run Time|
|--------|-------|-------|-------------|--------------|------------|
|1       |       |A      |             |              |13:06|
|2       |       |B      |             |              |13:05|
|3       |       |C      |             |              |13:05|

That's it. Now you can not only delay your event processing to reduce the noise, you can continue to respond quickly when things are important.

## Try It Out

If you want to give it a try I have a working example on GitHub. [https://github.com/jasonwadsworth/blog-code/tree/main/delayed-event-processing-part-2](https://github.com/jasonwadsworth/blog-code/tree/main/delayed-event-processing-part-2)
