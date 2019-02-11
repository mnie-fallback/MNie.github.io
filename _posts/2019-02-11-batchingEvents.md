---
layout: post
title: Batching Events via BatcherBlock
---

Hi,

as a response to some situations in our systems, we want to send real-time notifications to users. Because of that, we are using an [Ably.io](https://www.ably.io/) library. But as long as the application grows we encounter a problem. Because our application starts working slowly, consume more resources which means it costs us more and the most important thing is that we see that some of the events were duplicated which causes a lot of errors in web and mobile application.

Based on that we decided to implement the mechanism, which responsibility would be to collect events, group them, eliminate duplicates and then after some interval send them to the end devices (mobile, web). This mechanism should be fully transparent to a client of the batcher (developer), he shouldn't know at first how it works under the hood. The only thing we want from the developer is to decide which strategy he wants to choose when selecting events to send. So the only thing we want from him is a registration in an IoC container. Which looks like this in our codebase (Autofac):

```csharp
services.AddSingleton(new Batcher<int>(2, 500, new LastOneStrategy(), (d) => doSomething(d)));
```

Of course, he also needs to inject batcher in a class where he wants to use it and then on the injected service (batcher) invoke a `SendAsync` method which implies sending a message(s). Which looks like this:

```csharp
_batcher.SendAsync(msg, cancellationToken);
```

So going into details, implementation of a batcher looks like this:

```csharp
public class Batcher<TMessage> : IBatcher<TMessage>
{
    private readonly BufferBlock<TMessage> _bufferBlock = new BufferBlock<TMessage>(new DataflowBlockOptions
    {
        EnsureOrdered = true
    });

    private readonly TransformBlock<TMessage, TMessage> _bufferInterceptor = new TransformBlock<TMessage, TMessage>(x =>
    {
        Console.WriteLine($"Get a message with value: {x}");
        return x;
    });
    private readonly TransformBlock<TMessage, TMessage> _timeoutInterceptor = new TransformBlock<TMessage, TMessage>(x =>
    {
        Console.WriteLine($"Move out from transformation block with a value: {x}");
        return x;
    });

    public Batcher(int size, int interval, IStrategy<TMessage> strategy, Action<IEnumerable<TMessage>> triggerFunc)
    {
        var batchBlock = new BatchBlock<TMessage>(size, new GroupingDataflowBlockOptions()
        {
            EnsureOrdered = true
        });
        var timer = new Timer(async _ =>
        {
            try
            {
                batchBlock.TriggerBatch();
                var data = await batchBlock.ReceiveAsync();
                var toSend = strategy.Apply(data);
                triggerFunc(toSend);
            }
            catch (Exception e)
            {
                Console.WriteLine($"Error occurs while trying to invoke action on batch", e);
            }
        }, null, 0, 500);
        var timeoutBlock = new TransformBlock<TMessage, TMessage>(v =>
        {
            timer.Change(interval, Timeout.Infinite);
            return v;
        });
        
        _timeoutInterceptor.LinkTo(batchBlock);
        timeoutBlock.LinkTo(_timeoutInterceptor);
        _bufferInterceptor.LinkTo(timeoutBlock);
        _bufferBlock.LinkTo(_bufferInterceptor);    
    }

    public async Task<Result<Unit>> SendAsync(TMessage msg, CancellationToken token = new CancellationToken())
    {
        try
        {
            var result = await _bufferBlock.SendAsync(msg, token);
            return result
                ? ResultFactory.CreateSuccess()
                : ResultFactory.CreateFailure<Unit>("Message was refused by queue");
        }
        catch (Exception e)
        {
            return ResultFactory.CreateFailure<Unit>(e.Message);
        }
    }
}
```

We could see that we used here a TPL library. So going through the code of a Batcher. We used a BatchBlock, whose main responsibility is to trigger a function every `x` milliseconds. This was achieved via `Timer`. Because of this, we could collect events on our side and then do something with them thanks to an `IStrategy` service and at the end send them in groups, where the size of this group is defined by us in a constructor. So we also have a guarantee that we wouldn't be sent more than `n` events at once which could be very helpful in terms of not exceeding quota per second/minute/day limit. `IStrategy` which you could see here is a service reponsible for grouping/filtering events additionally if needed and it's implementation looks like this;

```csharp
public interface IStrategy<TPayload>
{
    IReadOnlyCollection<TPayload> Apply(IEnumerable<TPayload> data);
}

public class LastOneStrategy : IStrategy<IExample>
{
    public IReadOnlyCollection<IExample> Apply(IEnumerable<IExample> data) => 
        data
            .GroupBy(x => x.Id)
            .Select(x => new Example(x.Key, x.Last().Payload))
            .ToList();
}
```

Where Example element looks like this:

```csharp
public interface IExample
{
    string Id { get; }
    int Payload { get; }
}

public class Example : IExample
{
    public string Id { get; }
    public int Payload { get; }

    public Example(string id, int payload)
    {
        Id = id;
        Payload = payload;
    }
}
```

The only thing which should be done additionally by me is informing about the state of a Batcher. I have to do this because when I add the event to the queue at the beginning. Which then is passed to the next blocks I lost an information about errors. I started by implementing interceptors via `TransformBlock`. Their task is to log information about the flow between blocks because as you may notice I have 3 blocks. Receiving, Storing and Triggering block. Going between each other is logged. Beyond of that, I decided that the whole mechanism responsible for sending finally a message to an external provider should be captured in a try/catch block because we don't know what could happen in an external library. If we don't do that this exception would be `catch` by a block and we wouldn't notice that something bad happened.

As you may see thanks to a TPL library we were able to implement a small mechanism which batches/stores data because of date and size which wasn't provided by `Ably.IO` (or we didn't "google" enough) and which is available for example in `EventBus` thanks to a partition key. But beyond that, the code still looks pretty easy and clear and could be used across the whole system. 

The code could be found here: [github](https://github.com/MNie/RequestBatcher)

Thanks for reading!