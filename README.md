# AwaitCheatSheat
## How exception handling works with async/await, task.WhenAll etc
0. Not even an async example: Exception caught properly.
```csharp
public class Exceptional
{
    public static void Main()
    {
        try
        {
            X();
        }
        catch (Exception)
        {
            Console.WriteLine("exception caught");
        }

        Console.ReadLine();
    }

    private static void X()
    {
        throw new Exception();
    }
}
```
---
1. Unhandled exception because `X` is executed somewhere else because `X` is marked as `async`.
```csharp
public class Exceptional
{
    public static void Main()
    {
        try
        {
            X();
        }
        catch (Exception)
        {
            Console.WriteLine("exception caught");
        }

        Console.ReadLine();
    }

    private static async void X()
    {
        throw new Exception();
    }
}
```
---
2. Unhandled exception because `X` is executed somewhere else because `X` is marked as `async`, marking `Main` as `async` doesn't change anything ... we're not awaiting anything.
```csharp
public class Exceptional
{
    public static async Task Main()
    {
        try
        {
            X();
        }
        catch (Exception)
        {
            Console.WriteLine("exception caught");
        }

        Console.ReadLine();
    }

    private static async void X()
    {
        throw new Exception();
    }
}
```
---
3. Wont compile: we try to `await` a `void`.
```csharp
public class Exceptional
{
    public static async Task Main()
    {
        try
        {
            await X();
        }
        catch (Exception)
        {
            Console.WriteLine("exception caught");
        }

        Console.ReadLine();
    }

    private static async void X()
    {
        throw new Exception();
    }
}
```
---
4. Exception caught, awaiting `X` will unwrap the `AggregateException` and throw the first (and single) one.
```csharp
public class Exceptional
{
    public static async Task Main()
    {
        try
        {
            await X();
        }
        catch (Exception)
        {
            Console.WriteLine("exception caught");
        }

        Console.ReadLine();
    }

    private static async Task X()
    {
        throw new Exception();
    }
}
```
---
5. Exception caught, awaiting `X` will `await` the first task that throws an exception, unwrap the exception, rethrow it, and `Main` will unwrap it to throw it again and finally catch it.
```csharp
public class Exceptional
{
    public static async Task Main()
    {
        try
        {
            await X();
        }
        catch (SystemException)
        {
            Console.WriteLine("SystemException caught");
        }
        catch (ApplicationException)
        {
            Console.WriteLine("ApplicationException caught");
        }

        Console.ReadLine();
    }

    private static async Task X()
    {
        var applicationExceptionTask = new Task(LogAndThrow<ApplicationException>);
        var systemExceptionTask = new Task(LogAndThrow<SystemException>);

        applicationExceptionTask.Start();
        systemExceptionTask.Start();

        await applicationExceptionTask;
        await systemExceptionTask; // never executed
    }

    private static void LogAndThrow<TException>() where TException : Exception, new()
    {
        Console.WriteLine($"Throwing {typeof(TException).Name}");
        throw new TException();
    }
}
```
---
6. Awaiting `WhenAll` will catch an `AggregatedException` with 2 exceptions, but will rethrow only the first one. Finally caught by the catch in main.
```csharp
public class Exceptional
{
    public static async Task Main()
    {
        try
        {
            await X();
        }
        catch (SystemException)
        {
            Console.WriteLine("SystemException caught");
        }
        catch (ApplicationException)
        {
            Console.WriteLine("ApplicationException caught");
        }

        Console.ReadLine();
    }

    private static async Task X()
    {
        var applicationExceptionTask = new Task(LogAndThrow<ApplicationException>);
        var systemExceptionTask = new Task(LogAndThrow<SystemException>);

        applicationExceptionTask.Start();
        systemExceptionTask.Start();

        await Task.WhenAll(applicationExceptionTask, systemExceptionTask);
    }

    private static void LogAndThrow<TException>() where TException : Exception, new()
    {
        Console.WriteLine($"Throwing {typeof(TException).Name}");
        throw new TException();
    }
}
```
---
7. Here `Main` will catch nothing because we're awaiting the continuation that doesn't throw. We can see in the continuation of `WhenAll` that the task `t` contains 2 exceptions.
```csharp
public class Exceptional
{
    public static async Task Main()
    {
        try
        {
            await X();
        }
        catch (SystemException)
        {
            Console.WriteLine("SystemException caught");
        }
        catch (ApplicationException)
        {
            Console.WriteLine("ApplicationException caught");
        }

        Console.ReadLine();
    }

    private static async Task X()
    {
        var applicationExceptionTask = new Task(LogAndThrow<ApplicationException>);
        var systemExceptionTask = new Task(LogAndThrow<SystemException>);

        applicationExceptionTask.Start();
        systemExceptionTask.Start();

        await Task.WhenAll(applicationExceptionTask, systemExceptionTask).ContinueWith(t =>
        {
            Console.WriteLine("Execution continuation");
            Console.WriteLine("Exceptions caught: " + t.Exception.InnerExceptions.Count);
            Console.WriteLine("Exceptions details: " + t.Exception.Flatten());
        });
    }

    private static void LogAndThrow<TException>() where TException : Exception, new()
    {
        Console.WriteLine($"Throwing {typeof(TException).Name}");
        throw new TException();
    }
}
```
---
8. Awating a custom awaitable that throws the actual `AggregateException` by calling `Task.Wait` instead of `GetAwaiter().GetResult()`, and then catching the `AggregateException` in `Main`.
```csharp
public class Exceptional
{
    public static async Task Main()
    {
        try
        {
            await X();
        }
        catch (SystemException)
        {
            Console.WriteLine("SystemException caught");
        }
        catch (ApplicationException)
        {
            Console.WriteLine("ApplicationException caught");
        }
        catch (AggregateException e)
        {
            Console.WriteLine($"AggregateException caught, {e}");
        }

        Console.ReadLine();
    }

    private static async Task X()
    {
        var applicationExceptionTask = new Task(LogAndThrow<ApplicationException>);
        var systemExceptionTask = new Task(LogAndThrow<SystemException>);

        applicationExceptionTask.Start();
        systemExceptionTask.Start();

        await new TaskExtensions.AggregatedExceptionAwaitable(Task.WhenAll(applicationExceptionTask, systemExceptionTask));
    }

    private static void LogAndThrow<TException>() where TException : Exception, new()
    {
        Console.WriteLine($"Throwing {typeof(TException).Name}");
        throw new TException();
    }

    public static class TaskExtensions // 100% stolen from Jon Skeet
    {
        public struct AggregatedExceptionAwaitable
        {
            private readonly Task _task;

            public AggregatedExceptionAwaitable(Task task)
            {
                _task = task;
            }

            public AggregatedExceptionAwaiter GetAwaiter() => new AggregatedExceptionAwaiter(_task);
        }

        public struct AggregatedExceptionAwaiter : ICriticalNotifyCompletion //INotifyCompletion is enough for awaiter
        {
            private readonly Task _task;

            internal AggregatedExceptionAwaiter(Task task)
            {
                _task = task;
            }

            public bool IsCompleted => _task.GetAwaiter().IsCompleted;

            public void UnsafeOnCompleted(Action continuation) => _task.GetAwaiter().UnsafeOnCompleted(continuation);

            public void OnCompleted(Action continuation) => _task.GetAwaiter().OnCompleted(continuation);

            public void GetResult()
            {
                // This will throw AggregateException directly on failure,
                // unlike task.GetAwaiter().GetResult()
                _task.Wait();
            }
        }
    }
}
```
## What's the point of `Task.Yield()` ? And why should I sometimes await it ?
`Task.Yield` works a little bit like posting a work item for later on the current `Dispatcher` or `SynchronizationContext` or whatever. Awaiting it makes sure the async method will not be blocking its caller. It's like await a fake task and postponing all work that is declared after the await to a continuation.

Here's an example:
```csharp
public class Yield
{
    public static async Task Main()
    {
        LogToConsole("Calling Yielding()");
        var t = Yielding(false); // pass `true` to call `Task.Yield` at the beginning of `Yielding` method
        LogToConsole("Yielding() called, control back to Main");

        LogToConsole("Awaiting task returned by Yielding");
        await t;
        LogToConsole("Yielding done");
    }

    private static async Task Yielding(bool isYielding)
    {
        if (isYielding)
            await Task.Yield();

        LogToConsole("Start some heavy blocking work", 1);
        Thread.Sleep(2000);
        LogToConsole("Heavy blocking work done", 1);

        LogToConsole("Awaiting some async heavy work", 1);
        await Task.Delay(1000);
        LogToConsole("Async heavy work done, control back to Yielding", 1);
    }


    private static void LogToConsole(string message, int indent = 0, [CallerMemberName] string caller = "")
    {
        Console.WriteLine($"{DateTime.UtcNow} - #{Thread.CurrentThread.ManagedThreadId} - {new string('\t', indent) } [{caller}] {message}");
    }
}
```
When `isYielding` is false the first part of `Yielding` is called synchronously by `Main`, making it block for 2 seconds because of the `Thread.Sleep`. The console output is:
```
11/23/2017 9:59:53 PM - #1 -  [Main] Calling Yielding()
11/23/2017 9:59:53 PM - #1 -     [Yielding] Start some heavy blocking work
11/23/2017 9:59:55 PM - #1 -     [Yielding] Heavy blocking work done
11/23/2017 9:59:55 PM - #1 -     [Yielding] Awaiting some async heavy work
11/23/2017 9:59:55 PM - #1 -  [Main] Yielding() called, control back to Main
11/23/2017 9:59:55 PM - #1 -  [Main] Awaiting task returned by Yielding
11/23/2017 9:59:56 PM - #4 -     [Yielding] Async heavy work done, control back to Yielding
11/23/2017 9:59:56 PM - #4 -  [Main] Yielding done
```
When `isYielding` is true the method returns immediately to `Main`. Then the rest of the method is executed in another task, blocking it, etc. The output looks like this:
```
11/23/2017 10:00:22 PM - #1 -  [Main] Calling Yielding()
11/23/2017 10:00:22 PM - #1 -  [Main] Yielding() called, control back to Main
11/23/2017 10:00:22 PM - #1 -  [Main] Awaiting task returned by Yielding
11/23/2017 10:00:22 PM - #3 -    [Yielding] Start some heavy blocking work
11/23/2017 10:00:24 PM - #3 -    [Yielding] Heavy blocking work done
11/23/2017 10:00:24 PM - #3 -    [Yielding] Awaiting some async heavy work
11/23/2017 10:00:25 PM - #4 -    [Yielding] Async heavy work done, control back to Yielding
11/23/2017 10:00:25 PM - #4 -  [Main] Yielding done
```
