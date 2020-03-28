---
layout: post
title:  "C# Interlocked.Add"
date:   2019-03-23 21:03:36 +0530
categories: .NET Threading C# Interlocked.Add Interlocked.Increment Interlocked.Decrement
---

In this article, we're going to describe the problem we'll face when working with a shared variable from multiple threads and the effective way of solving it using **`System.Threading.Interlocked.Add`**.

Without much of a theory, let's jump right into the code

```csharp
class Worker
{
    public static int doneWorkCount;

    public static void DoWork()
    {
        // Simulate some time consuming work here
        Thread.Sleep(TimeSpan.FromSeconds(1));

        doneWorkCount = doneWorkCount + 1;
    }

    public static void Main()
    {
        for (var i = 0; i < 100; ++i)
        {
            DoWork();
        }

        Console.WriteLine(doneWorkCount);
    }
}
```

As you can see, the final value of variable **`doneWorkCount`** is **100**, but since all the invocations of **`DoWork`** method executes on a **same thread**, program takes at least 100 second to complete, pretty slooow. I bet we can fix that by using multiple threads, let's see how!

```csharp
....

public static void Main()
{
    var threads = new List<Thread>();

    for (var i = 0; i < 100; ++i)
    {
        var thread = new Thread(DoWork);
        thread.Start();
        threads.Add(thread);
    }

    foreach (var thread in threads)
    {
        thread.Join();
    }

    Console.WriteLine(doneWorkCount);
}
```

Well, yes, the program indeed executes way faster, the execution time is somewhere in between 1 and 2 seconds, but the final value of **`doneWorkCount`** variable is no longer stable, meaning it varies from execution to execution, 98, 96, 97, etc. Doesn't `DoWork` method still execute 100 times with each execution incrementing the value of `doneWorkCount` by 1? Well, yes (`DoWork` method still executes 100 time) and no (not all execution increments the vlaue of `doneWorkCount` by 1). To understand why, let's observe the increment statement

```csharp
doneWorkCount = doneWorkCount + 1; // doneWorkCount += 1 or doneWorkCount++ or ++doneWorkCount all are the same, obviously.
```

The increment consists of the following steps :

1. read the current value of **`doneWorkCount`** variable
2. add 1 to it
3. Write new value to **`doneWorkCount`** variable

As we see "Increment" is **not a single operation** and hence is not atomic

>  An operation is atomic if it is indivisible - in other words, nothing else can happen in the middle. So, with an atomic write, you can't have another thread reading the value half way through the write, and ending up "seeing" half of the old value and half of the new value. Similarly, with an atomic read, you can't have another thread changing the value half way through the read, ending up (again) with a value which is neither the old value nor the new value. [Source](https://jonskeet.uk/csharp/threads/volatility.html)

let's see how it can be interrupted

- **`thread1`** reads the value of **`doneWorkCount`** (0, the initial value)
- **`thread1`** adds 1 to 0
- **`thread2`** reads the value of **`doneWorkCount`** which is still 0 (**remember `thread1` has not written the new value (1) to `doneWorkCount` yet**)
- **`thread1`** writes calculated value **1** to **`doneWorkCount`**
- **`thread2`** adds 1 to 0
- **`thread2`** writes calculated value **1** to **`doneWorkCount`**

So, even though **`thread1`** and **`thread2`** both executed `DoWork` method, the **value of `doneWorkCount` was increased by 1 instead of 2**, due to lack of atomicity.

The simplest and fastest way to solve atomicity issue is to use `Interlocked.Add` method

```csharp
public static int Add (ref int location1, int value)
```
```
Adds two integers and replaces the first integer with the sum, as an atomic operation.

Parameters
location1:Int32
    A variable containing the first value to be added. The sum of the two values is stored in location1.
value:Int32
    The value to be added to the integer at location1.
Returns:Int32
    The new value stored at location1.
```

Looks like exactly what we want (Note that, there is `Interlocked.Increment/.Decrement` method which is simply [Interlocked.Add(ref location, 1/-1)](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/threading/interlocked.cs#L34)) respectively

Let's modify our `DoWork` method

```csharp
public static void DoWork()
{
    // Simulate some time consuming work here
    Thread.Sleep(TimeSpan.FromSeconds(1));

    Interlocked.Add(ref doneWorkCount, 1); //or Interlocked.Increment(ref doneWorkCount);
}
```

No matter how many thread executes `Interlocked.Add` method concurrently, it is guaranteed that increment operation will be atomic.