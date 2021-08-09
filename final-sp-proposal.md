# Streaming Producer: Proposal
Publishing events using the producer client is optimized for high and consistent throughput scenarios, allowing applications to collect a set of events as a batch and publish in a single operation.  In order to maximize flexibility, developers are expected to build and manage batches according to the needs of their application, allowing them to prioritize trade-offs between ensuring batch density, enforcing strict ordering of events, and publishing on a consistent and predictable schedule.

The primary goal of the streaming producer is to provide developers the ability to queue individual events for publishing without the need to explicitly manage batch construction, population, or service operations.  Events are collected as they are queued, organized into batches, and published by the streaming producer as batches become full or a certain amount of time has elapsed.  When queuing events, developers may request automatic routing to a partition or explicitly control the partition in the same manner supported by the `ProducerClient`, with the streaming producer managing the details of grouping events into the appropriate batches for publication.

## Things to know before reading

- The names used in this document are intended for illustration only. Some names are not ideal and will need to be refined during discussions.

- Some details not related to the high-level concept are not illustrated; the scope of this is limited to the high level shape and paradigms for the feature area.

- Fake methods are used to illustrate "something needs to happen, but the details are unimportant."  As a general rule, if an operation is not directly related to one of the Event Hubs types, it can likely be assumed that it is for illustration only.  These methods will most often use ellipses for the parameter list, in order to help differentiate them.

## Why this is needed
When applications need to process low frequency or sparse event streams, the current approach for publishing a single event can introduce inefficiencies that could negatively impact throughput. In order to avoid this, developers need to include non-trivial overhead to their code to manage decisions around caching batches-in-progress, routing events to the correct batch, and publishing partial batches when events aren't being published frequently . With the streaming producer, applications can queue events into the producer as they are needed and the producer will take care of efficiently managing batches and publishing. 

A method to publish a single event was available in legacy versions of the Event Hubs client library. However, this method used a naïve approach of  simply publishing a batch containing one event, rendering it highly inefficient.  Developers often assumed that there was an intelligence behind the method, expecting it to leverage efficient batching, leading to overuse and poor application performance. The streaming producer returns the ability to publish a single event, but with a more efficient implementation that helps to ensure throughput and reduce resource use.

The streaming producer is also directly competitive with Kafka's Producer. Kafka's producer is used by calling their send method on records (i.e. events) one at a time. The producer then adds them to a buffer and groups them for publishing, all of which is very similar to the approach intended to be taken here. To read more about the Kafka producer see their [documentation](https://kafka.apache.org/10/javadoc/org/apache/kafka/clients/producer/KafkaProducer.html). The final section in this proposal, the competitive analysis, goes into more detail about the Kafka producer as compared to the streaming producer.

## Goals

Support methods that allow applications to request to queue an event for publishing and receive notification when publishing of that event was successful or has failed.  The client should handle all details around queuing events, building efficient batches, and scheduling publishing.  

## High level scenarios

### User telemetry 

A shopping website wants to determine how users interact with their images, suggested products section, and reviews. To do this they gather telemetry data from each user as they are performing actions on the site, such as clicking buttons or using drop-downs. The user behavior data is collected through an Event Hub. The data frequency and volume varies greatly with a user's activity and is not predictable. 

The streaming producer allows this to happen seamlessly. When the user is actively interacting with the website, batches will be filled and sent right away, whereas if the user were to step away from the computer or pause interaction to read on the page for awhile, the batch may need to be sent partially empty. This takes advantage of the timeout and batch construction features found in the streaming producer. 

### Online radio station listeners

An online radio application requires an account to use. The account has limited demographic information about each user such as an age range, gender, and general regional location. For the first 15 seconds of every song played the application allows the radio station to observe how many listeners change the song at each second. They also observe the key, genre, artist, and tempo of the song, as well as demographics about which users changed the station. The application publishes all of this data to be aggregated, so that the application can better select songs to cater to their targeted audience. 

Since each person's profile is an individual event that is not dependent on any other person's profile, the application would like to just publish each profile to an Event Hub when they switch away from the song in the first 15 seconds. For the rest of the song the application doesn't publish any events, since it's just trying to analyze user's first reaction to the song, and how it changes over time. In order to maximize throughput the application publishes events by queueing them one at a time, since sometimes there may be many people changing the station and sometimes there are only a few. This takes advantage of the streaming producer since not all batches of events will be full and there are long gaps in between spurts of events being published.

### Processing activity sign-ups 

A large summer community sports club uses an application to process member sign-ups for golfing, swimming, tennis, and other activity slots. Time slots for all of the activities open up at 10:00 am Sunday morning for the upcoming week. Since this club services a large community, all of the slots fill up quickly and there are a lot of people submitting requests as soon as the slots open up. An individual member can submit requests for multiple events on multiple days at once, and if they have a family membership, they can do this for all of the members of their family at once.

Each of these sign-ups are individual, if a member submits requests to play tennis every morning of the week at 9:00 am, it's possible that only some days are accepted if others are processed first. The application uses an Event Hub to publish all of the events from each member and determine if there is space for that member to do the activity at that time. All of the events for a certain activity are sent to a specific partition. The application does this by using the streaming producer to publish each event to its designated partition. This scenario takes advantage of the streaming producer's functionality by queueing events by activity as they arrive. Since many arrive at once, and then very sparsely after that, it's very useful for the application to be able to just queue them as they arrive and when there are a lot of events they will send right away, and then when there are less the batches can be sent on timeout instead.

### Kafka developers working with Event Hubs 

When creating an application that leverages Azure, developers familiar with Kafka may choose to use the Kafka client library with the Event Hubs compatibility layer in order to pursue a familiar development experience and avoid the learning curve of a new service.  Because the publishing models align, a Kafka developer working with the Event Hubs streaming producer is able to leverage their existing knowledge and use familiar patterns for publishing events, reducing the learning curve and helping to deliver their Azure-based application more quickly.  This allows the developer to more fully embrace the Azure ecosystem and take advantage of cross-library concepts, such as `Azure.Identity` integration and a common diagnostics platform.  For applications taking advantage of multiple Azure services, this unlocks greater cohesion across areas of the application and amore consistent experience overall.

## Key concepts

- Each event queued for publishing is considered individual; there is no support for bundling events and forcing them to be batched together. 

- The streaming functionality should be contained in a dedicated client type; the `EventHubProducerClient` API should not be made more complicated by supporting two significantly different sets of publishing semantics and guarantees.  

- Streaming support requires a stateful client for each partition or partition key used, since partition assignment will be dynamic, resource cost won't be known at the creation of the producer

- Similar to other Event Hubs client types, the streaming producer has a single connection with the Event Hubs service. This allows developers to better anticipate and control resource use and understand scalability in order to meet their application's throughput needs

## Usage examples

The streaming producer supports the same set of constructors that are allowed by the `EventHubProducerClient`.
### Creating a default streaming producer

```csharp
var connectionString = "<< CONNECTION STRING >>";
var eventHubName = "<< EVENT HUB NAME >>";

// Create the streaming producer with default options
var producer = new StreamingProducer(connectionString, eventHubName);
```

### Creating the client with custom options

```csharp  
var connectionString = "<< CONNECTION STRING >>";
var eventHubName = "<< EVENT HUB NAME >>";

// Create the streaming producer
var producer = new StreamingProducer(connectionString, eventHubName, new StreamingProducerOptions
{
    Identifier = "My Custom Streaming Producer",
    MaximumWaitTime = TimeSpan.FromMilliseconds(500),
    MaximumQueuedEventLimit = 500
    RetryOptions = new EventHubsRetryOptions { TryTimeout = TimeSpan.FromMinutes(5) }
});    
```

### Creating the client with a fully qualified namespace and credential

```csharp
TokenCredential credential = new DefaultAzureCredential();
var fullyQualifiedNamespace = "<< NAMESPACE (likely similar to {your-namespace}.eventhub.windows.net) >>";
var eventHubName = "<< NAME OF THE EVENT HUB >>";

// Create the streaming producer with default options
var producer = new StreamingProducer(fullyQualifiedNamespace, eventHubName, credential);
```

### Creating client to send batches to the same partition concurrently

The application can decide if it wants to send multiple batches to the same partition concurrently. The default value of `MaximumConcurrentSendsPerPartition` is 1, meaning that the producer will try to publish a batch to a partition and then finish applying the retry policy before trying to publish another batch to that same partition. This is useful when events are independent and can be processed in parallel without worrying about which was processed first. This functionality cannot be used at the same time as idempotency however, since idempotency inherently depends on event ordering. 

```csharp 
var connectionString = "<< CONNECTION STRING >>";
var eventHubName = "<< EVENT HUB NAME >>";

// Create the streaming producer

var producer = new StreamingProducer(connectionString, eventHubName, new StreamingProducerOptions
{
    // Producer will try to publish up to 4 batches at the same time to each partition
    MaximumConcurrentSendsPerPartition = 4; 
});    
```

### Publish events using the Streaming Producer

```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub);

// Define the Handlers
Task SendSuccessfulHandler(SendEventBatchSuccessEventArgs args)
{
    Console.WriteLine($"The following batch was published by { args.PartitionId }:");
    foreach (var eventData in args.EventBatch)
    {
        Console.WriteLine($"Event Body is: { eventData.EventBody.ToString() }");
    }
    return Task.CompletedTask;
}

Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine($"Publishing FAILED due to { args.Exception.Message } in partition { args.PartitionId }");
    return Task.CompletedTask;
}

// Add the handlers to the producer
producer.SendEventBatchSucceededAsync += SendSuccessfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    // Enqueue some events
    for (var eventNum = 0; eventNum < 10; eventNum++)
    {
        var eventBody = new EventData($"Event # { eventNum }");

        // This method waits until the queue has room for the given event
        await producer.EnqueueEventAsync(eventBody);
    }
}
finally
{
    // Close sends all pending queued events and then shuts down the producer
    await producer.CloseAsync();
}
```

### Publish an enumerable of events using the Streaming Producer
If the application would like to send an enumerable full of events, the `EnqueueEventAsync(IEnumerable<EventData> eventData, CancellationToken cancellationToken = default)` overload provides functionality for this. An important thing to note however, is that events queued together will not necessarily be sent in the same batch. This allows the application to enqueue as many events as they want without worrying about the size of a batch. If a partition id or partition key is not specified, they also may not get sent to the same partition.

This overload is not available in the synchronous enqueuing method, since that introduces partial failures and successes, which in turn introduces additional unnecessary complexity to the application.
```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub);

// Define the Handlers
Task SendSuccessfulHandler(SendEventBatchSuccessEventArgs args)
{
    Console.WriteLine($"The following batch was published by { args.PartitionId }:");
    foreach (var eventData in args.EventBatch)
    {
        Console.WriteLine($"Event Body is: { eventData.EventBody.ToString() }");
    }
    return Task.CompletedTask;
}

Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine($"Publishing FAILED due to { args.Exception.Message } in partition { args.PartitionId }");
    return Task.CompletedTask;
}

// Add the handlers to the producer
producer.SendEventBatchSucceededAsync += SendSuccessfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    var eventList = new List<EventData>();

    // Define some events
    for (var eventNum = 0; eventNum < 10; eventNum++)
    {
        var eventBody = new EventData($"Event # { eventNum }");
        eventList.Add(eventBody);
    }

    // Enqueue them
    await producer.EnqueueEventAsync(eventList);
}
finally
{
    // Close sends all pending queued events and then shuts down the producer
    await producer.CloseAsync();
}
```

### Publish events to a specific partition
```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub);

// Define the Handlers
Task SendSuccessfulHandler(SendEventBatchSuccessEventArgs args)
{
    Console.WriteLine($"The following batch was published by { args.PartitionId }:");
    foreach (var eventData in args.EventBatch)
    {
        Console.WriteLine($"Event Body is: { eventData.EventBody.ToString() }");
    }
    return Task.CompletedTask;
}

Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine($"Publishing FAILED due to { args.Exception.Message } in partition { args.PartitionId }");
    return Task.CompletedTask;
}

// Add the handlers to the producer
producer.SendEventBatchSucceededAsync += SendSuccessfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    var enqueueOptions = new EnqueueEventOptions
    {
        PartitionId = "0"

        // Alternatively, you could use a partition key:
        // PartitionKey = "SomeKey"
    };

    // Enqueue some events
    for (var eventNum = 0; eventNum < 10; eventNum++)
    {
        var eventBody = new EventData($"Event # { eventNum }");

        // This method waits until the queue has room for the given event
        await producer.EnqueueEventAsync(eventBody, enqueueOptions);
    }
}
finally
{
    // Close sends all pending queued events and then shuts down the producer
    await producer.CloseAsync();
}
```

### Failure recovery: poison messages
When publishing to the Event Hub occasionally an error that is not resolved through retires may occur.  While some may be recovered if allowed to continue retrying, others may be terminal.  For example, a poison message that will never succeed. For Event Hubs, this could be the result of an event exceeding the maximum message size allowed by the Event Hubs service. Another could be if the application requires that an event be processed within a specific timeframe, so messages that are too old are meaningless. Since both of these failure indicate that the message can never be published/processed, the application will need to deal with the failure elsewhere in order to make forward progress.
```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub);

// Define the Handlers
Task SendSuccessfulHandler(SendEventBatchSuccessEventArgs args)
{
    Console.WriteLine($"The following batch was published by { args.PartitionId }:");
    foreach (var eventData in args.EventBatch)
    {
        Console.WriteLine($"Event Body is: { eventData.EventBody.ToString() }");
    }
    return Task.CompletedTask;
}

Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine($"Publishing FAILED due to { args.Exception.Message } in partition { args.PartitionId }");
    foreach (var eventData in args.EventBatch)
    {
        var eventBody = eventData.EventBody.ToString();
        if (IsPoisonEvent(eventData, args.Exception, ...))
        {
            Console.WriteLine($"\t\t{ eventBody } was too large to send.");
            LogPoisonEvent(eventData, ...);
        }
        else
        {
            LogFailure(eventData, args.Exception, ...);
        }
    }

    return Task.CompletedTask;
}

// Add the handlers to the producer
producer.SendEventBatchSucceededAsync += SendSuccessfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    while (TryGetNextEvent(out var eventData))
    {
        await producer.EnqueueEventAsync(eventData);
        Console.WriteLine($"There are { producer.TotalPendingEventCount } events queueud for publishing.")
    }
}
finally
{
    // Close sends all pending queued events and then shuts down the producer
    await producer.CloseAsync();
}
```

### Sending events immediately
Even though the streaming producer publishes events in the background, the application may want to force events to publish immediately. Awaiting `FlushAsync` will attempt to publish all events that are waiting to be published in the queue, and upon return it will have attempted to send all events and applied the retry policy when necessary.
```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub);

// Define the Handlers
Task SendSuccessfulHandler(SendEventBatchSuccessEventArgs args)
{
    Console.WriteLine($"The following batch was published by { args.PartitionId }:");
    foreach (var eventData in args.EventBatch)
    {
        Console.WriteLine($"Event Body is: { eventData.EventBody.ToString() }");
    }
    return Task.CompletedTask;
}

Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine($"Publishing FAILED due to { args.Exception.Message } in partition { args.PartitionId }");
    return Task.CompletedTask;
}

// Add the handlers to the producer
producer.SendEventBatchSucceededAsync += SendSuccessfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    // Enqueue some events
    for (var eventNum = 0; eventNum < 10; eventNum++)
    {
        var eventBody = new EventData($"Event # { eventNum }");

        // This method waits until the queue has room for the given event
        await producer.EnqueueEventAsync(eventBody);

        // Send all events on the queue before trying to send the next
        await producer.FlushAsync();
    }
}
finally
{
    // Close sends all pending queued events and then shuts down the producer
    await producer.CloseAsync();
}
```

### Closing the producer without publishing pending events
If the application would like to shut down the producer quickly without trying to publish all pending events, there is an optional argument in the close method that allows this to happen: `CloseAsync(bool abandonPendingEvents = false, CancellationToken cancellationToken = default)`.
```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub);

// Define the Handlers
Task SendSuccessfulHandler(SendEventBatchSuccessEventArgs args)
{
    Console.WriteLine($"The following batch was published by { args.PartitionId }:");
    foreach (var eventData in args.EventBatch)
    {
        Console.WriteLine($"Event Body is: { eventData.EventBody.ToString() }");
    }
    return Task.CompletedTask;
}

Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine($"Publishing FAILED due to { args.Exception.Message } in partition { args.PartitionId }");
    return Task.CompletedTask;
}

// Add the handlers to the producer
producer.SendEventBatchSucceededAsync += SendSuccessfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    // Enqueue some events
    for (var eventNum = 0; eventNum < 10; eventNum++)
    {
        var eventBody = new EventData($"Event # { eventNum }");

        // This method waits until the queue has room for the given event
        await producer.EnqueueEventAsync(eventBody);

        // Send all events on the queue before trying to send the next
        await producer.FlushAsync();
    }
}
finally
{
    // Close with the clear flag set to true clears all pending queued events and then shuts down the producer
    await producer.CloseAsync(true);
}
```

### Using idempotent retries

```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Use the streaming producer options to enable idempotent retries
var clientOptions = new StreamingProducerOptions
{
    EnableIdempotentRetries = true
};

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub, clientOptions);

// Define the Handlers
Task SendSuccessfulHandler(SendEventBatchSuccessEventArgs args)
{
    Console.WriteLine($"The following batch was published by { args.PartitionId }:");
    foreach (var eventData in args.EventBatch)
    {
        Console.WriteLine($"Event Body is: { eventData.EventBody.ToString() }");
    }
    return Task.CompletedTask;
}

Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine($"Publishing FAILED due to { args.Exception.Message } in partition { args.PartitionId }");
    return Task.CompletedTask;
}

// Add the handlers to the producer
producer.SendEventBatchSucceededAsync += SendSuccessfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    // Enqueue some events
    for (var eventNum = 0; eventNum < 10; eventNum++)
    {
        var eventBody = new EventData($"Event # { eventNum }");

        // This method waits until the queue has room for the given event
        await producer.EnqueueEventAsync(eventBody);
    }
}
finally
{
    // Close sends all pending queued events and then shuts down the producer
    await producer.CloseAsync();
}
```
## Future Enhancements for Discussion
- `Flush(int partitionId)` which flushes the queue for partition `partitionId` 
- `Flush(string partitionKey)` which flushes the queue for the partition mapped to by `partitionKey`
- `Clear(int partitionId)` which clears the queue for partition `partitionId` 
- `Clear(string partitionKey)` which clears the queue for the partition mapped to by `partitionKey`
- Customizable hash functions for mapping a partition key to a partition, could be:
    - Func passed through the options 
    - overloadable method

## API skeleton
This API skeleton includes the planned additional features as demonstrated above.
### `Azure.Messaging.EventHubs.Producer`

```csharp
// These are used by the producer to pass information to the application
// on the successful publish of a batch of events. 
public class SendEventBatchSuccessEventArgs : EventArgs
{
    public IEnumerable<EventData> EventBatch { get; init; }
    public string PartitionId {get; init; }
    public SendEventBatchSuccessEventArgs(IEnumerable<EventData> events, string partitionId);
}

public class SendEventBatchFailedEventArgs : EventArgs
{
    public IEnumerable<EventData> EventBatch { get; init; }
    public Exception Exception { get; init; }
    public string PartitionId { get; init; }
    public SendEventBatchFailedEventArgs(IEnumerable<EventData> events, Exception ex, string partitionId);
}

public class StreamingProducerOptions : EventHubProducerClientOptions
{
    public TimeSpan? MaximumWaitTime { get; set; } // default = 250 ms
    public int MaximumPendingEventCount { get; set; }  // default = 2500
    public boolean EnableIdempotentRetries { get; set; } // default = false
    public boolean MaximumConcurrentSendsPerPartition { get; set; } // default = 1
}

public class EnqueueEventOptions : SendEventOptions{}

public class StreamingProducer : IAsyncDisposable
{
    public event Func<SendEventBatchSuccessEventArgs, Task> SendEventBatchSucceededAsync;
    public event Func<SendEventBatchFailedEventArgs, Task> SendEventBatchFailedAsync;

    public string FullyQualifiedNamespace { get; }
    public string EventHubName { get; }
    public string Identifier { get; }
    public int TotalPendingEventCount { get; }
    public bool IsClosed { get; protected set; }
    
    public StreamingProducer(string connectionString, string eventHubName = default , StreamingProducerOptions streamingOptions = default);
    public StreamingProducer(string fullyQualifiedNamespace, string eventHubName, AzureNamedKeyCredential credential, StreamingProducerOptions streamingOptions = default);
    public StreamingProducer(string fullyQualifiedNamespace, string eventHubName, AzureSasCredential credential, StreamingProducerOptions streamingOptions = default);
    public StreamingProducer(string fullyQualifiedNamespace, string eventHubName, TokenCredential credential, StreamingProducerOptions streamingOptions = default);
    public StreamingProducer(EventHubConnection connection, EventHubProducerClientOptions clientOptions = default);
    
    public int GetPartitionQueuedEventCount(string partition = default);

    public virtual async Task EnqueueEventAsync(EventData eventData, CancellationToken cancellationToken);
    public virtual async Task EnqueueEventAsync(EventData eventData, CancellationToken cancellationToken, EnqueueEventOptions options);
    
    public virtual async Task EnqueueEventAsync(IEnumerable<EventData> eventData, CancellationToken cancellationToken);
    public virtual async Task EnqueueEventAsync(IEnumerable<EventData> eventData, CancellationToken cancellationToken, EnqueueEventOptions options);

    public virtual Task SendAsync(CancellationToken cancellationToken);
    internal Task ClearAsync(CancellationToken cancellationToken);

    public virtual ValueTask CloseAsync(CancellationToken cancellationToken);
    public virtual ValueTask CloseAsync(Boolean abandonPendingEvents, CancellationToken cancellationToken);
    public virtual ValueTask DisposeAsync(CancellationToken cancellationToken);

    protected virtual void OnSendEventBatchSucceededAsync(IEnumerable<EventData> events);
    protected virtual void OnSendEventBatchFailedAsync(IEnumerable<EventData> events, Exception ex, int partitionId);
}
```

## Competitive Analysis: Kafka 

The streaming producer will offer parity with most of the features provided by Kafka's producer. Both allow events to be added one at a time to a queue of events which are asynchronously published after batching them together. Both also support the following options: allowing retries or not, restricting the number of events held in the queue, setting the timeout value for sending a partial batch, and enabling idempotent retries. 

### Sending a single message
#### Kafka: Asynchronously adding to the buffer pool
In order to create a producer using Kafka the user needs to define the properties that the producer will use. The producer requires a list of host:port brokers, these are used to create the connection at the beginning. It also requires serializers for both the key and value type, so that the producer can serialize the key or value object to a byte array. The example below uses the built in serializers that Kafka provides. 
```java
// Creating the producer with basic properties
private Properties kafkaProperties = new Properties(); 
kafkaProperties.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProperties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer"); 
kafkaProperties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
producer = new KafkaProducer<String, String>(kafkaProperties);

// Creating a callback class
// This is called for each record that is sent
private class FakeCallback implements Callback{
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e){
        if (e != null){
            // This is what to do in the case of a failure 
            e.printStackTrace();
        }
        // Anything here would happen upon failure or success
    }
}

// Sending a simple record through Kafka's send
ProducerRecord<String, String> record = new ProducerRecord<>("Topic", "SomeKey", "SomeValue"); 

producer.send(record, new FakeCallback());
```

#### Streaming Producer: Asynchronously enqueuing an event
The process from above is mimicked below using the Event Hubs streaming producer instead.
```csharp
// Create the producer client
var connectionString = "<< CONNECTION STRING >>";
var eventHubName = "<< EVENT HUB NAME >>";
var client = new EventHubProducerClient(connectionString, eventHubName)

// Create the streaming producer with default options
var producer = new StreamingProducer(client);

// Define a handler, this will only get called in the case of a failure
// This is called for each batch that is failed
Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine(args.Exception.StackTrace);
    
    return Task.CompletedTask;
}

// The fail handler will get called automatically when necessary
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    var eventData = new EventData("SomeValue");
    
    // Create a property for the key
    var IDictionary<string, String> Properties = new Dictionary<string, String>();
    Properties.add("Key", "SomeKey");
    eventData.Properties = Properties;

    // Use the key to send all events with the same key to the same partition
    var sendOptions = new EnqueueEventOptions
    {
        PartitionKey = "SomeKey"
    };

    await producer.EnqueueEventAsync(eventData, sendOptions);
}
finally
{  
    await client.DisposeAsync();
}
```

#### Summary
All of Kafka's messages are sent in key, value pairs. It is possible to send a body without a key, but this is not the common case. The key is used for additional information about the message, as well as to assign it to a partition. The key is hashed and then used to assign the message to a partition. Without a key, the producer uses a round robin approach to assign it to a partition, which is the same approach as the streaming producer. 

However, the recommended approach with the streaming producer is to send it without a key or partition id unless events need to be sent to the same partition. This allows the application to take advantage of all the available partitions.

### Core Failures
#### Kafka: Dealing with non-retriable errors
```java
private Properties kafkaProps = new Properties();
kafkaProperties.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProperties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer"); 
kafkaProperties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
producer = new KafkaProducer<String, String>(kafkaProperties);

private class ProducerCallback implements Callback {
 @Override
 public void onCompletion(RecordMetadata recordMetadata, Exception e) {
    if (e == someReason) {
        // APPLICATION LOGIC TO DEAL WITH ERROR 
    }
    if (e != null){
        LogFailure(recordMetadata, e, ...);
    }
 }
}
ProducerRecord<String, String> record = new ProducerRecord<>("Topic", "Key", "Value"); 
producer.send(record, new ProducerCallback()); 
```


#### Streaming Producer: Dealing with non-transient errors
```csharp
// Define handlers for the streaming events
    
protected async Task SendSuccessfulHandler(SendEventBatchSucceededEventArgs args)
{
    foreach (var eventData in args.Events)
    {
        var eventId = eventData.Properties["event-id"];
        Console.WriteLine($"Event: { eventId } was published.");
    }
}

protected async Task SendFailedHandler(SendEventBatchFailedEventArgs args)
{
    Console.WriteLine("Publishing FAILED!");   
    if (args.Exception.Reason == SomeReason)
    {
        // APPLICATION LOGIC TO DEAL WITH ERROR
    } 
    LogFailure(args.EventBatch, args.Exception, ...);
}

var connectionString = "<< CONNECTION STRING >>";
var eventHubName = "<< EVENT HUB NAME >>";

// Create the streaming producer with default options
var producer = new StreamingProducer(connectionString, eventHubName);

producer.SendEventBatchSuccessAsync += SendSucceededfulHandler;
producer.SendEventBatchFailedAsync += SendFailedHandler;

try
{
    while (TryGetNextEvent(out var eventData))
    {
        await producer.EnqueueEventAsync(eventData);
    }
}
finally
{
    await client.DisposeAsync();
}
```

#### Kafka: Dealing with retriable errors
```java
private Properties kafkaProps = new Properties();
kafkaProperties.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProperties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer"); 
kafkaProperties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// Increasing the timeout value to better deal with transient errors
kafkaProperties.put("delivery.timeout.ms", 240000)
producer = new KafkaProducer<String, String>(kafkaProperties);
```

#### Streaming producer: Dealing with retriable errors
```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Increasing the number of times the producer retries publishing
var producerOptions = new StreamingProducerOptions
{
    RetryOptions = new EventHubsRetryOptions
    {
        MaximumRetries = 10
    }
};

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub, producerOptions);
```

#### Summary
Kafka's producer and the Streaming Producer both utilize pre-defined retry policies when dealing with transient or retriable errors. Rather than checking for recoverable errors in the error handler, it is recommended to determine reliability needs prior to creating the producer, and then defining the retry policy accordingly. Both Kafka and Event Hubs allow customizable retry policies, but also have their own default retry policies that can be used.

Kafka prefers users to define retry policies in terms of timeouts, where the producer tries essentially as many times as possible until the timeout is reached, Event Hubs prefers users to define the maximum number of retries the producer will try to send to the service.

### Customizing the producer
#### Kafka: Producer configuration options
```java
private Properties kafkaProps = new Properties();
kafkaProperties.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProperties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer"); 
kafkaProperties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
// SPECIFY MORE OPTIONS HERE (same form as the above)

producer = new KafkaProducer<String, String>(kafkaProperties);
```

#### Streaming Producer: Streaming producer options 
```csharp
// Accessing the Event Hub
var connectionString = "<< CONNECTION STRING >>";
var hub = "<< EVENT HUB NAME >>";

// Increasing the number of times the producer retries publishing
var producerOptions = new StreamingProducerOptions
{
    // SPECIFY OPTIONS HERE
};

// Create the streaming producer
var producer = new StreamingProducer(connectionString, hub, producerOptions);
```

#### Summary
Kafka's producer and the streaming producer both give applications options for customizing the producer. These include things like the retry policy, the amount of time before sending partial batches, and the number of pending events that can be held inside the producer. In this regard, both the Kafka producer and the Streaming producer are relatively similar.