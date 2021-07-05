---
layout: post
title: Working With Hierarchy Data In DynamoDB
tags:
  - aws
  - serverless
  - DynamoDB
  - hierarchy
---

Working with hierarchies in DynamoDB can be a little intimidating. In this post I'll show you two ways to work with hierarchies, and hopefully take away some of the fear.

## The Path Pattern

The first, and most common, way to deal with hierarchy data in DynamoDB is what I refer to as the path pattern. If you think of you hierarchy data like a directory or folder structure on a computer, it's pretty easy to get your head around how this works. Each element in the tree represents a folder and finding what is in a folder is as simple as knowing the path to the folder. With the right table structure, you can query for items in a folder as well as all items in all folders below a given folder. Here is how that works.

We'll start with a simple folder structure. Imagine you have some data that is something like this:

<details open>
  <summary>Example data</summary>

```text
  Drive C
    Folder I
    Folder II
  Drive D
    Folder III
      Folder a
      Folder b
    Folder IV
      Folder c
    Folder V
      Folder d
        Folder i
        Folder ii
        Folder iii
      Folder e
```

</details>

When storing this data in DynamoDB you would include the full path of each item with that item. You might have a table that looks something like this:

<details open>
  <summary>Data with hierarchy data field</summary>

| ID | Root | Path | Folder Name
| -- | ------ | ------ | -----------
| C | | | Drive C
| I | C | / | Folder I
| II | C | / | Folder II
| D | | | Drive D
| III | D | / | Folder III
| a | D | /III/ | Folder a
| b | D | /III/ | Folder b
| IV | D | / | Folder IV
| c | D | /IV/ | Folder c
| V | D | / | Folder V
| d | D | /V/ | Folder d
| i | D | /V/d/ | Folder i
| ii | D | /V/d/ | Folder ii
| iii | D | /V/d/ | Folder iii
| e | D | /V/ | Folder e

</details>

With the data in this format, you can get any single folder by its ID. You can list all folders directly under a folder by using the GSI (which will be the `Root` and the `Path`) and specifying the root and the path. You can also get all folders below a folder (all the way down the tree) by using the GSI and specifying the root and that the path `begins_with` the path of the folder.

The code to get the `Folder d` looks something like this:

```javascript
const documentClient = new AWS.DynamoDB.DocumentClient();
const result = await documentClient.get({
            TableName: 'working-with-hierarchy-data-in-dynamodb',
            Key: {
                ID: 'd'
            },
        }).promise();
```

Querying for a list of the items directly below `Folder V` would look something like this:

```javascript
const documentClient = new AWS.DynamoDB.DocumentClient();
const result = await documentClient.query({
            TableName: 'working-with-hierarchy-data-in-dynamodb',
            IndexName: 'gsi1',
            KeyConditionExpression: '#root = :root AND #path = :path',
            ExpressionAttributeNames: {
                '#root': 'Root',
                '#path': 'Path'
            },
            ExpressionAttributeValues: {
                ':root': 'D',
                ':path': '/V/'
            },
        }).promise();
```

Because the condition is `=` the only results will be those directly below `Folder V`; `Folder d` and `Folder e`.

Getting a list of all items below `Folder V` would look something like this:

```javascript
const documentClient = new AWS.DynamoDB.DocumentClient();
const result = await documentClient.query({
            TableName: 'working-with-hierarchy-data-in-dynamodb',
            IndexName: 'gsi1',
            KeyConditionExpression: '#pk = :root AND begins_with(#sk, :path)',
            ExpressionAttributeNames: {
                '#root': 'Root',
                '#path': 'Path'
            },
            ExpressionAttributeValues: {
                ':root': 'D',
                ':path': '/V/'
            },
        }).promise();
```

In this case the condition is `begins_with`, so the results will include all values below `Folder V`; `Folder d`, `Folder i`, `Folder ii`, `Folder iii`, and `Folder e`.

This pattern allows for a great deal of power when dealing with hierarchical data. I would say that most often this is the pattern you'll want to use.

## The Replication Pattern

While the above pattern is a very powerful tool in working with data in DynamoDB, I use it all the time, sometimes, you might find that it doesn't work for you. The above approach assumes that you will always know the structure above the node you are searching. In the examples, querying for items below a folder requires that you not only know the ID of the folder you wish to query, but you needed to know all the folders above it as well. In most cases that's not a problem, and even when you don't want to expose that information (maybe for security reasons), doing a quick lookup of that data is likely the best route (i.e., you can get the folder you want to query, and use that to know the path information above it without requiring that the caller supply the full path). So, when might you want to do something different?

### Querying varying levels of depth

In the first example you were limited to only being to query either items directly below a node, or all items below a node.

If you wanted to get all items _except_ those directly below a node you'd have to query all items and filter the direct items out. That's not so bad if the number of direct items is a small number, but if you have a lot of data to exclude it's not ideal.

If you wanted to get just the items below a node and the items directly below those items (i.e., two levels deep from the node you're querying) you'd have to either query all the items and filter out the ones you don't want, or run multiple queries, one for the direct items, and one query for each of the results from that query. The first option may be okay if there aren't too many records you want to exclude, but if you are a large hierarchy, say you are working with employee data and you're the size of Amazon, doing that for the root node (e.g., the CEO) would probably not be the best idea. The second option may not be terrible, assuming the number of items directly below the node you are querying is small, but it's certainly not going to be as fast as querying it all at once. Yes, you can run the each of the sub node queries in parallel, but they all require the first query to return first.

If you want three levels, or four levels, want to skip a level or two, or some other combination of ranges, well, I think you can see how complicated, and non-performant, that might get.

### Faster querying when hierarchy is unknown

As I mentioned, if you don't know the structure of the hierarchy you can always request the node you want to start at and get that information before querying the hierarchy. In the world of DynamoDB you're talking about single digit millisecond latency on that get. For some applications that level of added latency is a problem (if it wasn't there wouldn't be a service like DAX). Most applications are probably fine accepting this small hit, but if you find that you need something with better read performance keep reading.

### Reduced read costs

This one isn't a slam dunk, and there are even some cases where it might not be true (though most of those cases aren't a good use for this pattern anyway), but the size of that hierarchy data can get to be big if your hierarchy is deep. Small amounts can add up to a lot if you process a lot of data. This pattern will be more expensive on writes (and storage), but the read costs can be quite a bit lower in many cases.

### The solution

Okay, let's get into how it works. At its core is one simple thing; every record needs to have a copy made for each node above it in the hierarchy. The data will include some extra information, specifically the relative depth and the id of the ancestor item. Using the above data, you'd end up with records like this:

<details open>
  <summary>Example data for replicated data</summary>

| ID | Ancestor ID | Relative Depth | Folder Name
| -- | ----------- | -------------- | -----------
| C | I | 0 | Drive C
| I | I | 0 | Folder I
| I | C | 1 | Folder I
| II | II | 0 | Folder II
| II | C | 1 | Folder II
| D | D | 0 | Drive D
| III | III | 0 | Folder III
| III | D | 1 | Folder III
| a | a | 0 | Folder a
| a | III | 1 | Folder a
| a | D | 2 | Folder a
| b | b | 0 | Folder b
| b | III | 1 | Folder b
| b | D | 2 | Folder b
| IV | IV | 0 | Folder IV
| IV | D | 1 | Folder IV
| c | c | 0 | Folder c
| c | IV | 1 | Folder c
| c | D | 2 | Folder c
| V | V | 0 | Folder V
| V | D | 1 | Folder V
| d | d | 0 | Folder d
| d | V | 1 | Folder d
| d | D | 2 | Folder d
| i | i | 0 | Folder i
| i | d | 1 | Folder i
| i | V | 2 | Folder i
| i | D | 3 | Folder i
| ii | ii | 0 | Folder ii
| ii | d | 1 | Folder ii
| ii | V | 2 | Folder ii
| ii | D | 3 | Folder ii
| iii | iii | 0 | Folder iii
| iii | d | 1 | Folder iii
| iii | V | 2 | Folder iii
| iii | D | 3 | Folder iii
| e | e | 0 | Folder e
| e | V | 1 | Folder e
| e | D | 2 | Folder e
</details>

As you can see, there are a lot more records in the table. You'll also notice that each record is smaller than the first example. The `Relative Depth` is a number, so the size of that is fixed, and because we don't have to store the entire hierarchy on each row the extra data of the `Ancestor ID` is smaller than the `Path`, in the first pattern, for most rows.

Let's now look at some of the query patterns this makes available.

The code to get a folder, like `Folder d`, looks like it did before, but we need to add the relative depth:

```javascript
const documentClient = new AWS.DynamoDB.DocumentClient();
const result = await documentClient.get({
            TableName: 'working-with-hierarchy-data-in-dynamodb',
            Key: {
                ID: 'd',
                'Relative Depth': 0
            },
        }).promise();
```

Querying for a list of the items directly below `Folder V` would look something like this:

```javascript
const documentClient = new AWS.DynamoDB.DocumentClient();
const result = await documentClient.query({
            TableName: 'working-with-hierarchy-data-in-dynamodb',
            IndexName: 'gsi1',
            KeyConditionExpression: '#ancestorId = :ancestorId AND #relativeDepth = :relativeDepth',
            ExpressionAttributeNames: {
                '#ancestorId': 'Ancestor ID',
                '#relativeDepth': 'Relative Depth'
            },
            ExpressionAttributeValues: {
                ':ancestorId': 'V',
                ':relativeDepth': 1
            },
        }).promise();
```

Because the relative depth is set to `1` the only results will be those directly below `Folder V`; `Folder d` and `Folder e`.

Getting a list of all items below `Folder V` would look something like this:

```javascript
const documentClient = new AWS.DynamoDB.DocumentClient();
const result = await documentClient.query({
            TableName: 'working-with-hierarchy-data-in-dynamodb',
            IndexName: 'gsi1',
            KeyConditionExpression: '#ancestorId = :ancestorId AND #relativeDepth > :zero',
            ExpressionAttributeNames: {
                '#ancestorId': 'Ancestor ID',
                '#relativeDepth': 'Relative Depth'
            },
            ExpressionAttributeValues: {
                ':ancestorId': 'V',
                ':zero': 0
            },
        }).promise();
```

In this example we are stating that the relative depth needs to be greater than `0`, so it will not return `Folder V`, but it will return everything below it; `Folder d`, `Folder i`, `Folder ii`, `Folder iii`, and `Folder e`.

Now, let's say you want everything directly in `Drive D` as well as everything in those folders, but nothing beyond that (i.e., two levels deep). The query would look like this:

```javascript
const documentClient = new AWS.DynamoDB.DocumentClient();
const result = await documentClient.query({
            TableName: 'working-with-hierarchy-data-in-dynamodb',
            IndexName: 'gsi1',
            KeyConditionExpression: '#ancestorId = :ancestorId AND #relativeDepth BETWEEN :start AND :end',
            ExpressionAttributeNames: {
                '#ancestorId': 'Ancestor ID',
                '#relativeDepth': 'Relative Depth'
            },
            ExpressionAttributeValues: {
                ':ancestorId': 'D',
                ':start': 1,
                ':end': 2,
            },
        }).promise();
```

This would return `Folder III`, `Folder a`, `Folder b`, `Folder IV`, `Folder c`, `Folder V`, `Folder d`, and `Folder e`, but it would not return `Folder i`, `Folder ii`, or `Folder iii`.

If you want three (3), or four (4) levels you just change the parameters. You can even include the node you are querying by starting at zero (0) or skip a level by starting at a depth of two (2).

As you can see, this pattern opens up some new access patterns that didn't exist before, or that might have been slower performing.

### Tradeoffs

Of course, there are tradeoffs to this method, beyond the write vs. read tradeoffs we already mentioned.

First, any time a record is updated you'll have to update the record and all the replicated records as well. The cost of that operation isn't too expensive because it's a linear value based on the depth in the tree. For most trees you probably wouldn't need to do more than a handful of updates.

Second, re-parenting can be a bit complex. In the first pattern re-parenting just means updating the `Path` for the records at and below the re-parented record. With this pattern you need to remove records from former ancestors and add them to new ancestors. I've linked to example code for doing this, and it's not too terribly complex, but it's not a particular cheap operation, so if you move nodes around a lot you may want to consider the cost of that.

### Streams

In the example code I've linked to this blog post I'm doing all the work of duplicating and updating the records at the time of saving. A better approach to this would be to use DynamoDB streams. As a record is added you can use the stream `INSERT` event to trigger the code to replicate the data to each of the ancestors. You can also use the stream `MODIFY` event to trigger re-parenting and updating of name data (or any other data that needs to be updated), and the `REMOVE` event to remove records from ancestors.

## Conclusion

Hierarchies are an important part of working with data, and data stored in DynamoDB is no exception. By using one of the above patterns, I hope you find that working with hierarchy data in DynamoDB isn't something to fear. Once you find yourself using them, you'll open up a whole world of potential query patterns you might not have thought possible before.

## Try It Out

If you want to give the second pattern a try, I have a working example on GitHub that you can deploy to your AWS account. [https://github.com/jasonwadsworth/blog-code/tree/main/working-with-hierarchy-data-in-dynamodb](https://github.com/jasonwadsworth/blog-code/tree/main/working-with-hierarchy-data-in-dynamodb)
