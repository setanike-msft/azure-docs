---
title: Change feed pull model
description: Learn how to use the Azure Cosmos DB change feed pull model to read the change feed and the differences between the pull model and Change Feed Processor
author: timsander1
ms.author: tisande
ms.service: cosmos-db
ms.subservice: cosmosdb-sql
ms.devlang: dotnet
ms.topic: conceptual
ms.date: 08/02/2021
ms.reviewer: sngun
---

# Change feed pull model in Azure Cosmos DB
[!INCLUDE[appliesto-sql-api](../includes/appliesto-sql-api.md)]

With the change feed pull model, you can consume the Azure Cosmos DB change feed at your own pace. As you can already do with the [change feed processor](change-feed-processor.md), you can use the change feed pull model to parallelize the processing of changes across multiple change feed consumers.

## Comparing with change feed processor

Many scenarios can process the change feed using either the [change feed processor](change-feed-processor.md) or the pull model. The pull model's continuation tokens and the change feed processor's lease container are both "bookmarks" for the last processed item (or batch of items) in the change feed.

However, you can't convert continuation tokens to a lease container (or vice versa).

> [!NOTE]
> In most cases when you need to read from the change feed, the simplest option is to use the [change feed processor](change-feed-processor.md).

You should consider using the pull model in these scenarios:

- Read changes from a particular partition key
- Control the pace at which your client receives changes for processing
- Perform a one-time read of the existing data in the change feed (for example, to do a data migration)

Here's some key differences between the change feed processor and pull model:

|Feature  | Change feed processor| Pull model |
| --- | --- | --- |
| Keeping track of current point in processing change feed | Lease (stored in an Azure Cosmos DB container) | Continuation token (stored in memory or manually persisted) |
| Ability to replay past changes | Yes, with push model | Yes, with pull model|
| Polling for future changes | Automatically checks for changes based on user-specified `WithPollInterval` | Manual |
| Behavior where there are no new changes | Automatically wait `WithPollInterval` and recheck | Must check status and manually recheck |
| Process changes from entire container | Yes, and automatically parallelized across multiple threads/machine consuming from the same container| Yes, and manually parallelized using FeedRange |
| Process changes from just a single partition key | Not supported | Yes|

> [!NOTE]
> Unlike when reading using the change feed processor, you must explicitly handle cases where there are no new changes. 

## Consuming an entire container's changes

You can create a `FeedIterator` to process the change feed using the pull model. When you initially create a `FeedIterator`, you must specify a required `ChangeFeedStartFrom` value, which consists of both the starting position for reading changes and the desired `FeedRange`. The `FeedRange` is a range of partition key values and specifies the items that will be read from the change feed using that specific `FeedIterator`.

You can optionally specify `ChangeFeedRequestOptions` to set a `PageSizeHint`. When set, this property sets the maximum number of items received per page. If operations in the monitored collection are performed
through stored procedures, transaction scope is preserved when reading items from the Change Feed. As a result, the number of items received could be higher than the specified value so that the items changed by the same transaction are returned as part of
one atomic batch.

The `FeedIterator` comes in two flavors. In addition to the examples below that return entity objects, you can also obtain the response with `Stream` support. Streams allow you to read data without having it first deserialized, saving on client resources.

Here's an example for obtaining a `FeedIterator` that returns entity objects, in this case a `User` object:

```csharp
FeedIterator<User> InteratorWithPOCOS = container.GetChangeFeedIterator<User>(ChangeFeedStartFrom.Beginning(), ChangeFeedMode.Incremental);
```

Here's an example for obtaining a `FeedIterator` that returns a `Stream`:

```csharp
FeedIterator iteratorWithStreams = container.GetChangeFeedStreamIterator<User>(ChangeFeedStartFrom.Beginning(), ChangeFeedMode.Incremental);
```

If you don't supply a `FeedRange` to a `FeedIterator`, you can process an entire container's change feed at your own pace. Here's an example, which starts reading all changes starting at the current time:

```csharp
FeedIterator iteratorForTheEntireContainer = container.GetChangeFeedStreamIterator<User>(ChangeFeedStartFrom.Now(), ChangeFeedMode.Incremental);

while (iteratorForTheEntireContainer.HasMoreResults)
{
    FeedResponse<User> response = await iteratorForTheEntireContainer.ReadNextAsync();

    if (response.StatusCode == HttpStatusCode.NotModified)
    {
        Console.WriteLine($"No new changes");
        await Task.Delay(TimeSpan.FromSeconds(5));
    }
    else 
    {
        foreach (User user in response)
        {
            Console.WriteLine($"Detected change for user with id {user.id}");
        }
    }
}
```

Because the change feed is effectively an infinite list of items encompassing all future writes and updates, the value of `HasMoreResults` is always true. When you try to read the change feed and there are no new changes available, you'll receive a response with `NotModified` status. In the above example, it is handled by waiting 5 seconds before rechecking for changes.

## Consuming a partition key's changes

In some cases, you may only want to process a specific partition key's changes. You can obtain a `FeedIterator` for a specific partition key and process the changes the same way that you can for an entire container.

```csharp
FeedIterator<User> iteratorForPartitionKey = container.GetChangeFeedIterator<User>(
    ChangeFeedStartFrom.Beginning(FeedRange.FromPartitionKey(new PartitionKey("PartitionKeyValue")), ChangeFeedMode.Incremental));

while (iteratorForThePartitionKey.HasMoreResults)
{
    FeedResponse<User> response = await iteratorForThePartitionKey.ReadNextAsync();

    if (response.StatusCode == HttpStatusCode.NotModified)
    {
        Console.WriteLine($"No new changes");
        await Task.Delay(TimeSpan.FromSeconds(5));
    }
    else
    {
        foreach (User user in response)
        {
            Console.WriteLine($"Detected change for user with id {user.id}");
        }
    }
}
```

## Using FeedRange for parallelization

In the [change feed processor](change-feed-processor.md), work is automatically spread across multiple consumers. In the change feed pull model, you can use the `FeedRange` to parallelize the processing of the change feed. A `FeedRange` represents a range of partition key values.

Here's an example showing how to obtain a list of ranges for your container:

```csharp
IReadOnlyList<FeedRange> ranges = await container.GetFeedRangesAsync();
```

When you obtain of list of FeedRanges for your container, you'll get one `FeedRange` per [physical partition](../partitioning-overview.md#physical-partitions).

Using a `FeedRange`, you can then create a `FeedIterator` to parallelize the processing of the change feed across multiple machines or threads. Unlike the previous example that showed how to obtain a `FeedIterator` for the entire container or a single partition key, you can use FeedRanges to obtain multiple FeedIterators, which can process the change feed in parallel.

In the case where you want to use FeedRanges, you need to have an orchestrator process that obtains FeedRanges and distributes them to those machines. This distribution could be:

* Using `FeedRange.ToJsonString` and distributing this string value. The consumers can use this value with `FeedRange.FromJsonString`
* If the distribution is in-process, passing the `FeedRange` object reference.

Here's a sample that shows how to read from the beginning of the container's change feed using two hypothetical separate machines that are reading in parallel:

Machine 1:

```csharp
FeedIterator<User> iteratorA = container.GetChangeFeedIterator<User>(ChangeFeedStartFrom.Beginning(ranges[0]), ChangeFeedMode.Incremental);
while (iteratorA.HasMoreResults)
{
    FeedResponse<User> response = await iteratorA.ReadNextAsync();

    if (response.StatusCode == HttpStatusCode.NotModified)
    {
        Console.WriteLine($"No new changes");
        await Task.Delay(TimeSpan.FromSeconds(5));
    }
    else
    {
        foreach (User user in response)
        {
            Console.WriteLine($"Detected change for user with id {user.id}");
        }
    }
}
```

Machine 2:

```csharp
FeedIterator<User> iteratorB = container.GetChangeFeedIterator<User>(ChangeFeedStartFrom.Beginning(ranges[1]), ChangeFeedMode.Incremental);
while (iteratorB.HasMoreResults)
{
    FeedResponse<User> response = await iteratorA.ReadNextAsync();

    if (response.StatusCode == HttpStatusCode.NotModified)
    {
        Console.WriteLine($"No new changes");
        await Task.Delay(TimeSpan.FromSeconds(5));
    }
    else
    {
        foreach (User user in response)
        {
            Console.WriteLine($"Detected change for user with id {user.id}");
        }
    }
}
```

## Saving continuation tokens

You can save the position of your `FeedIterator` by obtaining the continuation token. A continuation token is a string value that keeps of track of your FeedIterator's last processed changes and allows the `FeedIterator` to resume at this point later. The following code will read through the change feed since container creation. After no more changes are available, it will persist a continuation token so that change feed consumption can be later resumed.

```csharp
FeedIterator<User> iterator = container.GetChangeFeedIterator<User>(ChangeFeedStartFrom.Beginning(), ChangeFeedMode.Incremental);

string continuation = null;

while (iterator.HasMoreResults)
{
    FeedResponse<User> response = await iterator.ReadNextAsync();

    if (response.StatusCode == HttpStatusCode.NotModified)
    {
        Console.WriteLine($"No new changes");
        continuation = response.ContinuationToken;
        // Stop the consumption since there are no new changes
        break;
    }
    else
    {
        foreach (User user in response)
        {
            Console.WriteLine($"Detected change for user with id {user.id}");
        }
    }
}

// Some time later when I want to check changes again
FeedIterator<User> iteratorThatResumesFromLastPoint = container.GetChangeFeedIterator<User>(ChangeFeedStartFrom.ContinuationToken(continuation), ChangeFeedMode.Incremental);
```

As long as the Cosmos container still exists, a FeedIterator's continuation token never expires.

## Next steps

* [Overview of change feed](../change-feed.md)
* [Using the change feed processor](change-feed-processor.md)
* [Trigger Azure Functions](change-feed-functions.md)
