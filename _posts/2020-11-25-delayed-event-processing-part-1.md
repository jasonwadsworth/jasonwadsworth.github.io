---
layout: post
title: Delayed Event Processing - Part 1
tags:
  - aws
  - serverless
  - events
---

Processing event data is a basic concept in today's cloud based architectures. We recently came across a situation where processing EVERY event was too much. Imagine if you are running radar and someone is exceeding the speed limit. The radar is constantly reporting the speed, and that speed may even change, but you really only need to take one action; pull the driver over and write a ticket. So how did we do this?

## The Data

To show how this works we are going to work with a simple set of data. Each message contains three pieces of information, the level, the service that reported it, and the time the message was received. We'll also include a message, just to make it look more like a log entry.

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

## Matching

Each action is triggered based on a message matching some configured value. For this example we are going to assume that we have two sets of matching criteria, one for errors, and one for warnings. We could have one match criteria that covered both, but we want errors to trigger within 5 minutes, and we'll let warnings go for 15.

|Criteria        |Trigger Time|
|----------------|------------|
|Level = Error   |5 minutes   |
|Level = Warning |15 minutes  |

## Unique Actions

A big part of what makes this solution work is having a means of identifying unique actions to be taken. In our example we are just going to log a message, but in a real world scenario you'd call another lambda, or trigger some other sort of compute. Whatever it is you're doing, you'll want to pass along some information about what it is that needs to be done. For example, say we wanted to run some analysis on the service reporting the errors/warnings. We'd need to tell the downstream process which service to analyze. It's this data, the parameter(s) passed to the downstream service, that makes the action unique.  So we need something more in our criteria; we need to know where in our message to get this value from. We used JSON path for this.

|Criteria        |Trigger Time|Parameter Path|
|----------------|------------|--------------|
|Level = Error   |5 minutes   |$.service     |
|Level = Warning |15 minutes  |$.service     |

With this criteria we will end up with the following records in our table.

|Record #|Level  |Service|Time Received|Scheduled Time|
|--------|-------|-------|-------------|--------------|
|1       |Warning|A      |13:00        |13:15         |
|2       |Error  |A      |13:05        |13:10         |
|3       |Error  |B      |13:05        |13:10         |
|4       |Warning|A      |13:05        |13:20         |
|5       |Warning|C      |13:05        |13:20         |

> Note: the "Record #" column is added just for this conversation, it is not an attribute in the table. Times are modified to exclude date to simplify this view, but the entire date is stored in the action table.

## Processing

With all this in place we need to process the records. A lambda function is set to run once a minute. The lambda will query the table to find records that are due to run. For each unique record it then finds any others that are the same, and considers them all as processed.

In our example, if the lambda runs at 13:10 it will identify records 2 & 3 as needing to be processed. Since records 1 & 4 are matches for record 2 they will also be marked as processed, leaving only record 5 to be processed in the future.

That's it! With this simple pattern you can process a large number of messages and only trigger downstream services a few times, potentially saving a lot of compute and noise.

## Try It Out

If you want to give it a try I have a working example on GitHub. [https://github.com/jasonwadsworth/blog-code/tree/main/delayed-event-processing-part-1](https://github.com/jasonwadsworth/blog-code/tree/main/delayed-event-processing-part-1)

## Next Step

In [part 2](../delayed-event-processing-part-2) of this blog we'll look into how we can trigger immediately on the first message received.
