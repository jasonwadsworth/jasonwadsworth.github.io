---
layout: post
title: Creating a Unique Constraint with DynamoDB
tags:
  - aws
  - serverless
  - DynamoDB
  - unique constraint
---

There are a lot of reasons why switching from SQL to NoSQL is a good idea for much of what we as developers do. The vast majority of our work is OLTP, transactional data processing, where we know what the access patterns are and can design our NoSQL data storage in a way that supports those access patterns.

Of course there are inevitably things that are not natively supported in NoSQL databases like DynamoDB, and often these things are a hurdle to those looking to make the transition. One of those things is the unique constraint.

## Definition

If you're not familiar with unique constraints, they are a way of guaranteeing that there is only one instance of a particular value (or set of values if it is a composite constraint) in a table. It's different than the primary key in that it isn't...well...the primary key. A good example of this is a user table where the primary key would be a `userId` of some sort, and a unique constraint would be the user's `email`. In a SQL database you can have this constraint keep you from being able to have two users with the same email because the table will not allow duplicates.

In DynamoDB there isn't a unique constraint, but there is a way to get the same behavior. Here is how you do it.

## Transactions to the rescue

There was a Twitter thread recently debating the value of DynamoDB transactions. I know there are some who don't like them, but this is one example of why I think they are valuable.

With a DynamoDB transaction you can create a limited ACID transaction on a set of records. For the example of the `email` constraint on a user you need just two (DynamoDB supports up to 100 records at the time of this writing, with a 4MB total size limit). So what does this transaction look like?

Basically you have a `Put` for each unique constraint and one for the primary record. If you're doing a single table design this can all be targeting the same table, but DynamoDB transactions can work across many tables. Each constraint record's key (combination of Partition Key and Sort Key) uniquely identifies it on the table. Each `Put` includes a condition that requires that the record either doesn't already exist or is owned by the user being updated.

Here's an example what that might look like:

```TypeScript
const user = {
	userId: 'User1',
	email: 'john@example.com',
	first: 'John',
	last: 'Doe'
};

await documentClient.send(new new TransactWriteCommand({
    TransactItems: [
        {
            Put: {
                Item: { pk: user.email, sk: 'EmailConstraint', userId: user.userId },
                TableName: 'User',
                ConditionExpression: 'attribute_not_exists(pk) OR userId = :userId',
                ExpressionAttributeValues: {
                    ':userId': user.userId,
                }
            }
        },
        {
            Put: {
                Item: { ...user, pk: user.userId, sk: 'User' },
                TableName: 'User'
            }
        }
    ]
}));
```

The first `Put` has a unique value of the `email` and the value `'EmailConstraint'`. The second `Put` has a unique value of the `userId` and the value `'User'`. That means that you can only have one record with a particular email in the table, and only one record with a particular userId. The `ConditionExpression` on the first `Put` limits the operation by saying that the record being put must be a new record (`attribute_not_exists(pk)`) or the userId of the current record must be the same as the record being saved (`userId = :userId`).

Now imagine if we try add a second user that looks like this:

```TypeScript
const user = {
	userId: 'User2',
	email: 'john@example.com',
	first: 'John',
	last: 'Roe'
};
```

Using the above transaction this would fail because the first item in the transaction would violate the `ConditionExpression`. Specifically, the record would already exist, so the `attribute_not_exists(pk)` would be false, and because the userId of the existing record is for a different user (User1), that would also be false.

If we change the record to have a different email it succeeds:

```TypeScript
const user = {
	userId: 'User2',
	email: 'john.roe@example.com',
	first: 'John',
	last: 'Roe'
};
```

Now, let's say we want to update the first record, so we do another transaction to update it to the following:

```TypeScript
const user = {
	userId: 'User1',
	email: 'john@example.com',
	first: 'Johnathan',
	last: 'Doe'
};
```

This one will succeed. The `EmailConstraint` record's `ConditionExpression` is satisfied. While the `attribute_not_exists(pk)` would be false, the `userId = :userId` would be true. Essentially nothing is changing on this record.

What about when you want to change the email of a user?

```TypeScript
const user = {
	userId: 'User2',
	email: 'john@example.com',
	first: 'John',
	last: 'Roe'
};
```

Again, this would fail because the `EmailConstraint` record already exists and it does not belong to this user.

```TypeScript
const user = {
	userId: 'User1',
	email: 'johnanthan@example.com',
	first: 'Johnathan',
	last: 'Doe'
};
```

This would succeed because there is no record with johnanthan@example.com as it's key. Of course this creates a different problem. Now you have an orphaned record, `john@example.com`. Your first thought might be to include a `Delete` in the transaction, but that won't quite work because we don't know the email address to delete. You could look it up first, of course, but you can't look it up within the transaction, so you'd end up possibly having out of sync data if two updates happened at the same time. Probably not a big concern for a user's email, but it could be an issue with other data. You could make sure that the record you're updating is still the record you read, and fail if it is not. That is a good option in some cases, when the probability of a collision is low. If it fails you can look it up again.

## Embracing eventual consistency

Personally, I like to take a different approach to this problem. By using DynamoDB streams you can check for a change to the email address whenever there is a `MODIFY`, and if there is a change you can delete the old record at that time. This means that there is a period when the email is still unavailable, but it is a great way to guarantee the delete. You'll want to do the same on a `REMOVE`, since a delete has the same problem of not knowing what the email is at the time of the delete.

## Conclusion

No, a unique constraint in DynamoDB isn't as easy as it is in a SQL database, and you should only use it when you absolutely need it, but at least now you know that it is an option. One more excuse to to avoid NoSQL is gone.