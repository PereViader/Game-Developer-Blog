---
layout: post
title: "Rules for having a nice time with Tasks and Cancellation in Unity3d"
date: 2025-05-10 00:00:00 +1
---

Working with async and CancellationToken in Unity3d can feel like juggling knives—one wrong move and something crashes silently or never cancels when it should. After running into enough edge cases, crashes, and confusing bugs, I’ve settled on a set of simple, consistent rules that help keep async code predictable, debuggable, and safe. These are the conventions I follow in all my Unity3d projects. Hope they help you too.


### 1. There should be a single `async void` method in the codebase

Exception: Event callbacks that need to trigger async flows can break this rule, otherwise they can't actually be called

All async flows must start within `async void`, otherwise the errors get captured and silently ignored by the last Task in the async state machine. When the last method is an `async void` instead of an `async Task` or `Task`, that method will actually unwrap the error and properly crash the application for uncaptured exceptions.

The need to have root methods be implemented using `async void` but following awaited methods as `Task` will make implementation more confusing.

For this reason, I advocate for having a single `async void` extension method on Task that is used everywhere else.
Doing it this way also helps make it clear that the method is a branching async Task flow.


```csharp
public static async void RunAsync(this Task task)
{
    try
    {
        await task;
    }
    catch(OperationCancelledException ex)
    {
        //Supressed
    }
}

public void MethodSync()
{
    //This clearly shows that an async flow is starting
    MethodAsync(CancellationToken.None).RunAsync();
}

public async Task MethodAsync(CancellationToken ct) { ... }
```


### 2. All async methods should receive a CancellationToken as the last parameter

When all methods implement cancellation properly, implementation complexity is nil. Providing cancellation everywhere as parameter means that we can implement any cancellation strategy at any of the layers.

Make sure to use an abbreviation and to name it `ct`. Recurring patterns warrant abbreviations for conciseness sake. Think of it as a common pattern just like an index variable in a for loop `for(int i ...`

Avoid using `= default` for CancellationToken parameters. Relying on the default parameter makes it easy for callers to accidentally omit the token. Since cancellation paths are frequently less tested, these omissions often lead to production issues. If you genuinely intend not to support cancellation, explicitly pass `CancellationToken.None` instead. This clearly communicates your intent and prevents accidental omissions.

```csharp
public async Task Method1Async(CancellationToken ct) { ... }
public Task Method3Async(Guid id, float time, CancellationToken ct) { ... }
```

### 3. Avoid using CancellationToken.None

Implementing cancellation effectively is only beneficial if your application actively utilizes it. Often, the use of `CancellationToken.None` stems from the difficulty of correctly managing and obtaining a valid `CancellationToken` within the codebase, rather than a deliberate decision to forgo cancellation.

To address this, you need to simplify the process of working with CancellationTokens. My current preferred approach is using a utility wrapper around CancellationTokenSource called TaskRunner.

The `TaskRunner` (available [here](https://github.com/PereViader/PereViader.Utils/blob/962b404dbab4642f54dc073a23eef42d9eb8e3c8/PereViader.Utils.Common/PereViader.Utils.Common/TaskRunners/TaskRunner.cs)) is designed to be created within classes and provides a consistent way to run tasks with associated cancellation. It handles the internal creation and disposal of `CancellationTokenSource` instances upon cancellation.

Ideally, `TaskRunner` should be shared across parts of your application that require unified cancellation behavior. Once the relevant context is no longer needed, disposing of the `TaskRunner` instance will cancel all running tasks and prevent new ones from being initiated on it.

```csharp
class SomeFeature
{
    private readonly TaskRunner _taskRunner;

    public SomeFeature(TaskRunner taskRunner)
    {
        _taskRunner = taskRunner;
    }

    public void DoSomethingThatWillTriggerAnAsync()
    {
        // You can run the task instantly
        _taskRunner.RunInstantly(DoSomethingAsync).RunAsync();

        //Alternatively you can queue it for execution
        _taskRunner.RunSequenced(DoSomethingAsync).RunAsync();
    }

    public Task DoSomethingAsync(CancellationToken ct) { ... }
}
```


### 4. Forward CancellationToken and avoid interaction with it

We can differentiate between (non leaf) methods that forward a cancellation token and those (leaf) that where waiting actually occurs.

For instance in this example below the method `NonLeafMethod` is a non leaf method thus it only forwards the token, it itself will not do any actual waiting.
Then the method calls `Task.Delay` which is where the actual waiting will happen, thus internally it will use the CancellationToken to make sure it can be canceled at any time.

```csharp
public async Task NonLeafMethod(CancellationToken ct)
{
    //Notice that the CancellationToken is only forwarded to the next method
    //Task.Delay will use the CancellationToken internally accordingly
    await Task.Delay(1000, ct); 
}
```


### 5. Stay clear of cancellation bolierplate

By embracing these two rules all cancellation boilerplate can be eliminated 
1. Cancellable Task methods should assume they are never started cancelled
2. Tasks should fail with `OperationCancelledException` when cancelled.

For rule 1, by embracing the assumption, you can be freed of the need to check the status of the CancellationToken on all methods in the codebase. You will only need to make sure that async flows are never started with a token that is already cancelled. This is much easier and a good practice in any case.

For rule 2, `OperationCancelledException` comes from `CancellationToken.ThrowIfCancellationRequested` or `TaskCompletionSource.SetCancelled`. This is the preferred C# convention and the way all of the System and third party libraries work. Following the same pattern the language and third parties work will ease integration and avoids complexity.+


```csharp
public async Task SomeAsyncMethod(CancellationToken ct)
{
    // Async task methods should always assume that ct.IsCancellationRequested == false
    // When called by other async tasks this will happen naturally
    // When they are the root of some async excution, the programmer should take caution to make sure the rule is followed
    // By assuming this you can do work direclty without needing to check it
    await OtherMethod1Async(ct);
    // If it reaches here, it means that the CancellationToken was not cancelled while awaiting the method above
    await OtherMethod2Async(ct);
    // If it reaches here, it means that the CancellationToken was not cancelled while awaiting the method above

    // Notice that there is 0 boilerplate related to cancellation 
    // At the same time the code can be cancelled at any point during its execution
}
```


### 6. Avoid deadlocks, leaks and complexity when using TaskCompletionSource

TaskCompletionSource offers the perfect tools to turn any callback based API as tasks.
Great care should be taken when using it in order to avoid issues in your application.
Here are some rules that will help you

- Dispose of the `CancellationTokenRegistration`, otherwise resources will leak for long living CancellationTokens
- TaskCompletionSource should be an implementation detail. Consumers should only be provided with Tasks, exposing TaskCompletionSource is a source of unnecesary complexity.
- Extreme caution should be taken when storing TaskCompletionSource as class members. Deadlocks are common in unexpected TaskCompletionSorce edge cases. 

As a rule of thumb, highly cohesive transactional implementations should be preferred. Or in other words, the flow for interacting with the TaskCompletionSource should be as simple as possible.

Example of a TaskCompletionSource integrating with a callback based method. This allows consumers call and await the method it is wrapping

```csharp
// Notice that the consumer of the method needn't know about the TaskCompletionSource within
public async Task DoSomeWorkAsync(CancellationToken ct)
{
    var tcs = new TaskCompletionSource();

    // Notice that we are disposing of the CancellationTokenRegistration
    using var _ = ct.Register(() => tcs.SetCancelled());

    DoSomeWorkCallback(() => {
        if(ct.IsCancellationRequested)
        {
            return;
        }

        tcs.SetResult();
    }

    await tcs.Task; 
}
```

Example of a TaskCompletionSource integrating with an event. This allows consumers to await for the next time the event is triggered.

```csharp
public event Action OnSomeEvent;

// Notice that the consumer of the method needn't know about the TaskCompletionSource within
public async Task WaitOnSomeEvent(CancellationToken ct)
{
    var tcs = new TaskCompletionSource();

    // Notice that we are disposing of the CancellationTokenRegistration
    using var _ = ct.Register(() => {
        OnSomeEvent -= Callback;
        tcs.SetCancelled()
    });

    void Callback()
    {
        OnSomeEvent -= Callback;
        tcs.SetResult();
    }

    OnSomeEvent += Callback;
    await tcs.Task;
}
```


### 7. Async Methods should cancel instantly
Exception: This rule can be broken when the cancellation process is asyncrhonous. This should be the exception rather than the rule.

For example, the following snippet below does not properly cancel instantly given it waits for the callback to run before it either completes or cancels.
See the snippet above for an example where it is finishing instantly

```csharp
// Attention: Do not do this in your codebase 
public Task MethodAsync(CancellationToken ct)
{
    var tcs = new TaskCompletionSource();

    SomeCallbackMethod(() => {
        if(ct.IsCancellationRequested)
        {
            // Cancellation happens late, this callback may run at any point in the future
            // After the cancellation was requested
            tcs.SetCancelled();
            return;
        }
        
        tcs.SetResult();
    })

    return tcs.Task;
}
```
