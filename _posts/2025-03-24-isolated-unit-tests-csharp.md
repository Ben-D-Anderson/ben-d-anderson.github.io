---
layout: post
title: Isolating C# Unit Tests Using Application Domains [.NET Framework]
description: Running unit tests in isolated application domains, thus providing the ability to test a singleton-oriented code base as if each unit test was running in a new instance of the application.
readtime: 6 minute
toc: true
tags: programming
---

Code that executes in a new [AppDomain](https://learn.microsoft.com/en-us/dotnet/api/system.appdomain?view=net-9.0) has a completely detached state from the originating application domain - even all static attributes take their default values. This behaviour is only really comparible to stopping the application and re-running it.

In this blog post, I will demonstrate how to use this functionality to unit test singleton classes which can only be initialised once, and don't provide any sort of clean-up or teardown methods. The techniques used to achieve this behavior are **only** available in .NET Framework.<!--excerpt-->

## Executing Code In A New Application Domain

### Executing Fixed Code

Microsoft's [documentation](https://learn.microsoft.com/en-us/dotnet/api/system.appdomain.docallback?view=netframework-4.8.1) on using application domains only demonstrates using the feature to execute static code in another domain, not consuming any existing state from the current domain. Other [articles](https://web.archive.org/web/20160911103936/http://www.superstarcoders.com/blogs/posts/executing-code-in-a-separate-application-domain-using-c-sharp.aspx) built on this example by introducing a clean class for running isolated work, and even demonstrating passing parameters to the isolated application domain.

The `Isolated` class provides a means of executing code in another `AppDomain`:
```cs
public class Isolated<T> : IDisposable where T : MarshalByRefObject
{
    private AppDomain _domain;
    public T CrossDomainObject { get; private set; }

    public Isolated()
    {
        _domain = AppDomain.CreateDomain("Isolated:" + Guid.NewGuid(), 
            null, AppDomain.CurrentDomain.SetupInformation);

        Type type = typeof(T);
 
        CrossDomainObject = (T)_domain.CreateInstanceAndUnwrap(type.Assembly.FullName, type.FullName);
    }
 
    /// <exception cref="CannotUnloadAppDomainException">
    /// When the code running in the AppDomain refuses to shutdown/finalise.
    /// </exception>
    public void Dispose()
    {
        if (_domain != null)
        {
            AppDomain.Unload(_domain);
            _domain = null;
        }
    }
}
```

Here's how to use that class to run fixed code with parameters in an isolated environment:
```cs
class Program
{
    static void Main(string[] args)
    {
        using (Isolated<Worker> isolated = new Isolated<Worker>())
        {
            isolated.CrossDomainObject.DoWork(new Parameters { Data = 1 });
        }
    }
}

[Serializable]
public class Parameters
{
    public int Data { get; set; }
}

public class Worker : MarshalByRefObject
{
    public void DoWork(Parameters parameters)
    {
        //this code runs in an isolated AppDomain, with the supplied parameters
    }
} 
```

### Executing Arbitrary Actions

Parameters passed across app domains must be serializable, so in order to execute arbitrary [Action](https://learn.microsoft.com/en-us/dotnet/api/system.action-1?view=net-9.0)(s) in isolated app domains, we must first serialize the `Action`.

#### Serializing Actions

```cs
public class ActionSerialization
{
    public static byte[] Serialize(Action action)
    {
        using (MemoryStream ms = new MemoryStream())
        {
            BinaryFormatter bf = new BinaryFormatter();
            bf.Serialize(ms, action);
            return ms.ToArray();
        }
    }

    public static Action Deserialize(byte[] serializedAction)
    {
        using (MemoryStream ms = new MemoryStream())
        {
            BinaryFormatter bf = new BinaryFormatter();
            ms.Write(serializedAction, 0, serializedAction.Length);
            ms.Seek(0, SeekOrigin.Begin);
            return (Action) bf.Deserialize(ms);
        }
    }
}
```

It should be noted that Actions should not be serialized and then stored for later use, whether that be by using the file system or other means. This is because [BinaryFormatter](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.serialization.formatters.binary.binaryformatter?view=net-9.0) is actually [insecure](https://learn.microsoft.com/en-us/dotnet/standard/serialization/binaryformatter-security-guide). Furthermore, if the code base changes at all then the deserialization [could fail](https://stackoverflow.com/a/49139733/14844306).

#### Sending Actions To Isolated AppDomains

By combining the `Isolated` class and the action serialization code, we can execute Actions in an isolated AppDomain.
Below I have provided an example which will instantiate a singleton in two different AppDomains:
```cs
public class AnnoyingSingleton
{
    private static AnnoyingSingleton _instance;

    public static AnnoyingSingleton GetInstance()
    {
        if (_instance == null)
        {
            Console.WriteLine("Instantiated AnnoyingSingleton");
            _instance = new AnnoyingSingleton();
        }
        return _instance;
    }
}
```

```cs
class Program
{
    static void Main(string[] args)
    {
        AnnoyingSingleton.GetInstance();
        // Output: "Instantiated AnnoyingSingleton"
        AnnoyingSingleton.GetInstance();
        // Output: nothing - singleton is already instantiated

        using (Isolated<Worker> isolated = new Isolated<Worker>())
        {
            Action actionToRun = () => AnnoyingSingleton.GetInstance();
            byte[] serializedAction = ActionSerialization.Serialize(actionToRun);

            isolated.CrossDomainObject.ExecuteAction(serializedAction);
            // Output: "Instantiated AnnoyingSingleton"
            isolated.CrossDomainObject.ExecuteAction(serializedAction);
            // Output: nothing - singleton is already instantiated in this isolated AppDomain
        }
    }
}

public class Worker : MarshalByRefObject
{
    public void ExecuteAction(byte[] serializedAction)
    {
        Action action = ActionSerialization.Deserialize(serializedAction);
        action.Invoke();
    }
}
```

Awesome! Now we are running arbitrary Actions in isolated AppDomains, letting us re-instantiate the singleton.

### Cross-Domain Exceptions

One drawback of the current implementation, is that code will silently fail if the executed Action throws an exception. This can be resolved by wrapping up the exception information and returning it from the cross-domain method.

```cs
[Serializable]
public class CrossDomainException : Exception
{
    public CrossDomainException() {}

    public CrossDomainException(string message) : base(message) {}

    public CrossDomainException(string message, Exception innerException) : base(message, innerException) {}

    protected CrossDomainException(SerializationInfo info, StreamingContext context) : base(info, context) {}
}
```

```cs
static void Main(string[] args)
{
    using (Isolated<Worker> isolated = new Isolated<Worker>())
    {
        Action actionToRun = () => AnnoyingSingleton.GetInstance();
        byte[] serializedAction = ActionSerialization.Serialize(actionToRun);

        CrossDomainException? exception =
            isolated.CrossDomainObject.ExecuteAction(serializedAction);

        if (exception != null)
        {
            throw exception;
        }
    }
}

public class Worker : MarshalByRefObject
{
    public CrossDomainException? ExecuteAction(byte[] serializedAction)
    {
        try
        {
            Action action = ActionSerialization.Deserialize(serializedAction);
            action.Invoke();
        }
        catch (Exception e)
        {
            return new CrossDomainException("[" + e.GetType().Name + "] "
                + e.Message
                + Environment.NewLine + Environment.NewLine
                + "Real Stack Trace:" + Environment.NewLine
                + e.StackTrace);
        }
        return null;  
    }
}
```

## Assembling The IsolatedAction Class

Time to put everything together in one convenient class that can be used to execute `Action`(s) in isolated `AppDomain`(s).

```cs
/// <summary>
/// Provides the capability to execute arbitrary Action(s) in isolated application domains.
/// </summary>
public sealed class IsolatedAction
{
    private readonly byte[] _serializedAction;

    /// <summary>
    /// Runs the provided action in an isolated AppDomain.
    /// </summary>
    /// <param name="action">The action to run in an isolated environment.</param>
    public static void Run(Action action)
    {
        Wrap(action).Invoke();
    }

    /// <summary>
    /// Convert a standard action into an isolated action.
    /// When the returned action is invoked, it will execute in an isolated AppDomain.
    /// </summary>
    /// <param name="action">The action to wrap in an isolator.</param>
    /// <returns>
    /// A standard Action type, wrapped with code to run the provided action in an isolated AppDomain.
    /// </returns>
    public static Action Wrap(Action action)
    {
        return () => new IsolatedAction(action).Invoke();
    }

    /// <summary>
    /// Instantiate a new IsolatedAction by wrapping around an existing Action.
    /// </summary>
    public IsolatedAction(Action action)
    {
        _serializedAction = SerializeAction(action);
    }

    /// <summary>
    /// Every time this method is invoked, the action is ran in a new application domain.
    /// </summary>
    /// <exception cref="CannotUnloadAppDomainException">
    /// When the code running in the isolated AppDomain refuses to shutdown/finalise.
    /// </exception>
    public void Invoke()
    {
        AppDomain domain = AppDomain.CreateDomain("Isolated:" + Guid.NewGuid(),
            null, AppDomain.CurrentDomain.SetupInformation);
        Type workerType = typeof(Worker);
        Worker crossDomainWorker = (Worker)
            domain.CreateInstanceAndUnwrap(workerType.Assembly.FullName, workerType.FullName);

        CrossDomainException? exception =
            crossDomainWorker.ExecuteAction(_serializedAction);

        AppDomain.Unload(domain);

        if (exception != null)
        {
            throw exception;
        }
    }

    private static byte[] SerializeAction(Action action)
    {
        using (MemoryStream ms = new MemoryStream())
        {
            BinaryFormatter bf = new BinaryFormatter();
            bf.Serialize(ms, action);
            return ms.ToArray();
        }
    }

    private static Action DeserializeAction(byte[] serializedAction)
    {
        using (MemoryStream ms = new MemoryStream())
        {
            BinaryFormatter bf = new BinaryFormatter();
            ms.Write(serializedAction, 0, serializedAction.Length);
            ms.Seek(0, SeekOrigin.Begin);
            return (Action)bf.Deserialize(ms);
        }
    }

    private class Worker : MarshalByRefObject
    {
        internal CrossDomainException? ExecuteAction(byte[] serializedAction)
        {
            try
            {
                DeserializeAction(serializedAction).Invoke();
            }
            catch (Exception e)
            {
                return new CrossDomainException("[" + e.GetType().Name + "] " + e.Message
                    + Environment.NewLine + Environment.NewLine
                    + "Real Stack Trace:" + Environment.NewLine
                    + e.StackTrace);
            }
            return null;
        }
    }

    [Serializable]
    public class CrossDomainException : Exception
    {
        public CrossDomainException() {}

        public CrossDomainException(string message) : base(message) {}

        public CrossDomainException(string message, Exception innerException) : base(message, innerException) {}

        protected CrossDomainException(SerializationInfo info, StreamingContext context) : base(info, context) {}
    }
}
```

## Using It In Unit Tests [MSTest]

We now have a few different ways of using the `IsolatedAction` class:

### Statically Isolating & Running An Action

```cs
[TestMethod]
public void RunIsolatedAction()
{
    IsolatedAction.Run(() =>
    {
        //code in here will run in an isolated AppDomain

        //this assertion will throw an AssertFailedException, which will then be
        // wrapped in a CrossDomainException and thrown on the unit testing thread.
        Assert.IsTrue(false);
    });
}
```

### Wrapping An Action With Isolation Code

```cs
[TestMethod]
public void RunUnisolatedThenIsolatedAction()
{
    //the original action
    Action existingAction = ...;
    //this will run in the current AppDomain
    existingAction.Invoke();
    
    //same as the original action but wrapped in isolation code
    Action isolatedAction = IsolatedAction.Wrap(existingAction);
    //this will run in an isolated AppDomain
    isolatedAction.Invoke();
}
```

### Instantiating & Invoking An IsolatedAction

```cs
[TestMethod]
public void RunActionTwiceInDifferentDomains()
{
    IsolatedAction isolatedAction = new IsolatedAction(() =>
    {
        //code in here will run in an isolated AppDomain
    });
    //run it for the first time, in AppDomain "Isolated:13d6..."
    isolatedAction.Invoke();
    //run it for the second time, in AppDomain "Isolated:a274..."
    isolatedAction.Invoke();
}
```

## Isolating Other Delegate Types

Understandably, you may want to isolate other delegate types, such as [Func\<TResult\>](https://learn.microsoft.com/en-us/dotnet/api/system.func-1?view=net-9.0). I won't go into written examples for doing that here, but the general idea is replacing instances of `Action` in the above examples with whatever delegate type you're looking to isolate.
