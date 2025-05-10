# Rules for having a nice time with Tasks and Cancellation in Unity3d

- The only `async void` that should be present in the codebase should be `RunAsync`
Exception: Event callbacks that need to trigger async flows can break this rule, otherwise they can't actually be called
Justification: Async void is the place where all async flows must start in order for them to be captured. Implementing some methods with `async void` and other with `Task` or `async Task` means that not all methods can be composed and that we need to manually update them when the need araises.

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
```


- In unity3d, at the root of execution of a Task method you should always call the `RunAsync` extension method
Justification: When a Task method is not awaited at the root level and the Task fails, the exception will not be captured and surfaced thus it is silently ignored. This happens because on each level of the Task awaiting state machine, the next task in the chain is the one that captures and forwards the exception to the next one by awaiting it there. Only Tasks awaited on `async void`  methods will actually capture and surface the surface the exception. When a Task method is run, the exception will end up captured within the last Task in the chain and then silently ignored and discarded.
RunAsync also helps make it clear that the method is a branching async Task flow that is starting.

```csharp
public void MethodSync()
{
    //This clearly shows that an async flow is started 
    MethodAsync(CancellationToken.None).RunAsync();
}

public async Task MethodAsync(CancellationToken ct) { ... }
```


- All async methods should receive a CancellationToken as the last paramter
Justification: When all methods implement cancellation properly, implementation complexity is nil. Providing cancellation everywhere as parameter means that we can implement any cancellation strategy at any of the layers.

```csharp
public async Task Method1Async(CancellationToken ct) { ... }
public Task Method3Async(Guid id, float time, CancellationToken ct) { ... }
```

- When an async Method is cancelled it should fail with an OperationCancelledException
Remarks: This is the task that is thrown when `CancellationToken.ThrowIfCancellationRequested` or `TaskCompletionSource.SetCancelled` is called
Justification: This is the preferred C# convention and the way all of the System and third party libraries work. Embracing this reduces complexity and friction.


- Cancellable async methods should take advantadge of C# OperationCancelledException and assume they are never started cancelled

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


- Cancellation token should only be interacted with by methods that need to actively implement some waiting
Justification: Leaf awaiting methods are the places where awaiting actually happens, those are the only places where the CancellationToken should be used, everywhere in the async chain should just forward the cancellation token
Example:

```csharp
public async Task NonLeafMethod(CancellationToken ct)
{
    //Notice that the CancellationToken is only forwarded to the next method
    await LeafTaskCompeletionSourceMethod(ct); 
}

public async Task LeafTaskCompeletionSourceMethod(CancellationToken ct)
{
    var tcs = new TaskCompletionSource();
    //Notice that we interact with the Cancellation token here
    using var _ = ct.Register(() => tcs.SetCancelled());

    DoSomeWorkThenRunCallback(() => {
        if(ct.IsCancellationRequested)
        {
            return;
        }

        tcs.SetResult();
    }

    await tcs.Task; 
}
```

- Encapsulate usages of TaskCompletionSource and expose them as Task
Justification: The usage of TaskCompletionSource should always be an implementation detail to aid transforming some callback based code into async Task based. Exposing the TaskCompletionSource, is usually a source of unnecesary complexity.
Exception: This rule may be broken when there are performance gains that warrant the extra complexity

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

- Prefer wrapping callback based APIs into async ones using TaskCompletionSource
Justification: Legacy callback based APIs if used direclty introduce complexity that is usually better hidden in reusuable wrapping methods 

```csharp
//Notice that the responsability of this method is just to wrap the other
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


- When an async Method is cancelled it should finish execution instantly
Exception: This rule can be broken when the cancellation process is asyncrhonous. This should be the exception rather than the rule.
For example, the following snippet does not properly cancel instantly given it waits for the callback to run before it either completes or cancels

```csharp
public Task MethodAsync(CancellationToken ct)
{
    var tcs = new TaskCompletionSource();

    SomeCallbackMethod(() => {
        if(ct.IsCancellationRequested)
        {
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