---
layout: post
title: "Rules for having a nice time with Tasks and Cancellation in Unity3d"
date: 2025-05-10 00:00:00 +1
---

Working with async and CancellationToken in Unity3d can feel like juggling knives—one wrong move and something crashes silently or never cancels when it should. After running into enough edge cases, crashes, and confusing bugs, I’ve settled on a set of simple, consistent rules that help keep async code predictable, debuggable, and safe. These are the conventions I follow in all my Unity3d projects. Hope they help you too.


### 1. There should be a single `async void` method in the codebase

Async void is the place where all async flows must start in order for them to be captured. When a root `async Task` or `Task` method and is called but not awaited and an exception happens inside it, the exception will be captured by that root Task and silently hidden. This avoids a potential application crash but at the same time hides information from the developer that something went wrong.

Implementing some methods with `async void` and other with `Task` or `async Task` also means that not all methods can be composed and that we need to manually update them when the need araises.

Thus implementing all methods with `Task` and then using a single `async void` method is the best way to make sure we get all the information and at the same.

I like to call this method `RunAsync`. The name also helps make it clear that the method is a branching async Task flow that is starting.

Exception: Event callbacks that need to trigger async flows can break this rule, otherwise they can't actually be called

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

```csharp
public async Task Method1Async(CancellationToken ct) { ... }
public Task Method3Async(Guid id, float time, CancellationToken ct) { ... }
```


### 3. Forward CancellationToken and avoid interaction with it

We can differentiate between (non leaf) methods that forward a cancellation token and those (leaf) that where waiting actually occurs and

```csharp
public async Task NonLeafMethod(CancellationToken ct)
{
    //Notice that the CancellationToken is only forwarded to the next method
    //Task.Delay will use the CancellationToken internally accordingly
    await Task.Delay(1000, ct); 
}
```


### 4. Stay clear of cancellation bolierplate

By embracing these two rules all cancellation boilerplate can be eliminated 
1. Cancellable Task methods should assume they are never started cancelled
2. Tasks should fail with `OperationCancelledException` when cancelled. 

For rule 1, by embracing the assumption, you can be freed of the need to check the status of the CancellationToken on all methods in the codebase. You will only need to make sure that async flows are never started with a token that is already cancelled. This is much easier and a good practice in any case.

For rule 2, `OperationCancelledException` comes from `CancellationToken.ThrowIfCancellationRequested` or `TaskCompletionSource.SetCancelled`. This is the preferred C# convention and the way all of the System and third party libraries work. Following the same pattern the language and third parties work will ease integration and avoids complexity.

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


### 5. Prefer wrapping callback based APIs into async ones using TaskCompletionSource

TaskCompletionSource offers the perfect tools to turn any callback based API as tasks. 

Make sure however to have TaskCompletionSource be an implementation. Exposing the TaskCompletionSource, is usually a source of unnecesary complexity.

Notice that when using the `CancellationToken.Register` call, we also dispose of the `CancellationTokenRegistration` returned, otherwise we could be leaking resource for long living CancellationTokens

```csharp
//Notice that the consumer of the method needn't know about the TaskCompletionSource within
public async Task DoSomeWorkAsync(CancellationToken ct)
{
    var tcs = new TaskCompletionSource();
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


### 6. When an async Method is cancelled it should finish execution instantly

Exception: This rule can be broken when the cancellation process is asyncrhonous. This should be the exception rather than the rule.

For example, the following snippet does not properly cancel instantly given it waits for the callback to run before it either completes or cancels

```csharp
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

This should instead be implemented by Registering the TaskCompletionSource to be finished instantly when the CancellationToken is cancelled

Notice that the callback will still run at a later time but it will detect it is cancelled and will early exit accordingly

```csharp
public async Task MethodAsync(CancellationToken ct)
{
    var tcs = new TaskCompletionSource();
    //Cancellation happens instantly together with the CancellationToken
    using var _ = ct.Register(() => tcs.SetCancelled());

    SomeCallbackMethod(() => {
        if(ct.IsCancellationRequested)
        {
            return;
        }
        
        tcs.SetResult();
    })

    await tcs.Task;
}
```