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
