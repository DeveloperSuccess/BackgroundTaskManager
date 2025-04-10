[![ru](https://img.shields.io/badge/lang-ru-green.svg)](./README.ru.md)

# Event-driven Deferred Task Manager C#

[![NuGet version (DeferredTaskManager)](https://img.shields.io/nuget/v/DeferredTaskManager.svg?style=flat-square)](https://www.nuget.org/packages/DeferredTaskManager)


The implementation allows you to use multiple background tasks (or "runners") for deferred processing of consolidated data. Runners are based on the PubSub template for asynchronous waiting for new tasks, which makes this approach more reactive but less resource-intensive.

## Distinctive advantage

The solution allows data consolidation in the current instance with the possibility of variable deduplication or any other operations at the discretion of the developer, which can reduce resources during further transmission and processing, as well as increase performance.

## Usage example

### 1️⃣ Injection of the Singleton dependency with the required data type

```
services.AddDeferredTaskManager<string>(options =>
{
    options.PoolSize = Environment.ProcessorCount;
    options.CollectionType = CollectionType.Bag;
    options.SendDelayOptions = new SendDelayOptions()
    {
        MillisecondsSendDelay = 60000,
        ConsiderDifference = true
    };
    options.RetryOptions = new RetryOptions<string>
    {
        RetryCount = 3,
        MillisecondsRetryDelay = 10000,
    };
});
```
#### ⚪ `PoolSize` — pool size (number of available runners)
The pool size is variable and is selected by the developer for a specific range of tasks, focusing on the speed of execution and the amount of resources consumed.
#### ⚪ `CollectionType` — collection type
You can also specify the collection type, «Bag» for the Unordered collection of objects (it works faster) or «Queue» for the Ordered collection of objects. It is advisable to use «Queue» only if `poolSize = 1`, otherwise the execution order is not guaranteed.
#### ⚪ `SendDelayOptions` — setting up sending events at a time interval
Настраивает отправку добавленных событий на обработку через определенный промежуток времени с возможностью переменного вычета времени предыдущей операции. Имеет смысл указывать, когда при добавлении событий используется флаг `sendEvents = false`, который добавляет события без отправки на обработку.
#### ⚪ `RetryOptions` — configuring exception handling
You can also specify parameters for repeated attempts to process events in case of exceptions.

### Modules and their redefinition
  
The solution consists of 5 modules, each of which registers in DI.

  ⚪ `IDeferredTaskManagerService` is the public interface of the main module, which implements the main public methods for working with the deferred task manager.
  
The following public interfaces are used for internal interaction: 

  ⚪ `IEventSender` — responsible for creating runners and contains the logic of their behavior.
  
  ⚪ `IEventStorage` is a layer for interacting with the event repository.
  
  ⚪ `IStorageStrategy` — used to implement event storage.

  ⚪ `IPoolPubSub` — performs lending with a pool of background runners.
  
All these interfaces are public, but, in fact, only the `IDeferredTaskManagerService` is used for external interaction. The implementation of each module can be redefined by adding its own dependencies and logic (which is why they also have public interfaces). 

You can redefine modules by sending custom types to the DI `services.AddDeferredTaskManager` registration method. The type being redefined must be inherited from one of the above public interfaces.

### 2️⃣ Creating a Background Service
An example is the creation of a background service for `DeferredTaskManager<string>`:
```
internal sealed class EventManagerService : BackgroundService
{
    private readonly IDeferredTaskManagerService<string> _deferredTaskManager;

    public EventManagerService(IDeferredTaskManagerService<string> deferredTaskManager)
    {
        _deferredTaskManager = deferredTaskManager;
    }

    protected override Task ExecuteAsync(CancellationToken cancellationToken)
    {
       // A delegate for custom logic that receives events from running runners.
       // As an example, events are concatenated in it,
       // but any other variable processing or sending is possible.
       Func<List<string>, CancellationToken, Task> eventConsumer = async (events, cancellationToken) =>
        {
            // Concatenation of events
            var concatenatedEvents = string.Join(",", events);

            //throw new Exception("Test exception");

            // Any further processing/sending of concatenated events
            Thread.Sleep(1000);
            await Task.Delay(1000, cancellationToken);        
        };
        
        // The delegate where we get to in case of exceptions in the main eventConsumer delegate
        Func<List<string>, Exception, int, CancellationToken, Task> eventConsumerRetryExhausted = async (events, ex, retryCount, cancellationToken) =>
        {
            Console.WriteLine($"Повтор по счету: {retryCount}; {ex}");
        };

        return _deferredTaskManager.StartAsync(eventConsumer, eventConsumerRetryExhausted, cancellationToken);
    }
}
```

#### ⚪ `EventConsumer' is the main delegate for custom logic

All custom logic is placed in the `EventConsumer` delegate, which receives a collection of consolidated events. This is where you can perform the necessary operations on them before further transmission/processing. You can also handle exceptions in the delegate (this is important if events are handled separately) by sending unprocessed events to the next session after the `MillisecondsRetryDelay` time delay specified in the parameters. In the example above, the delegate concatenates incoming events from running runners.

You can add your own exception handling logic to it, which can be useful if events are processed separately: successfully completed events can be deleted from the main event collection, then incomplete events will go to retry.
```
try
{
    // Custom operation on received events

    // Test exception
    throw new Exception("Test exception");     
}
catch (Exception ex)
{
    // Any custom logic (logging, etc.)

    // In case of event handling separately,
    // you can delete successfully completed events from the event collection,
    // then the unfinished events will go to retry
    events.RemoveRange(successEvents);   
}
```
#### ⚪ `EventConsumerRetryExhausted` — delegate for exception handling
Optionally, you can specify this delegate, which gets into in case of exceptions in the main delegate `EventConsumer'. Logging and other custom operations can be performed in it.

### 3️⃣ Getting an embedded dependency and implementing the addition of event(s)

```
_deferredTaskManager.Add(events);
```
## Alternative uses
The `DeferredTaskManager` can be used as a regular event store, receiving events on demand using the `GetEventsAndClearStorage` method, bypassing runners, or sending available events to a delegate to any available runner on demand using the `SendEvents` method.
