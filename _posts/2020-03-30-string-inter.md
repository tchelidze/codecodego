---
layout: post
title:  "C# String interning"
date:   2019-03-23 21:03:36 +0530
categories: .NET String C# MemoryOptimization
excerpt: In this artcile we're going to talk about string interning and how it can be used to improve a performance of our application
---

## Strings are immutable

well, it's hardly a news

```csharp
var str1 = "A";
str1 += "B";
```

Line `str1 += "B"` will allocate **new string** `"AB"` and set its reference to `str1`, while old string object (`"A"`) becomes garbage. The IL representation of above segment makes it clear.

```csharp
IL_0001: ldstr        "A"
IL_0006: stloc.0      // str1

IL_0007: ldloc.0      // str1
IL_0008: ldstr        "B"
IL_000d: call         string [mscorlib]System.String::Concat(string, string)
IL_0012: stloc.0      // str1
```

As you can see the **+=** is replaced with **String.Concat**, which does an allocation of new concatenated string.

*In fact, string immutability is **NOT** enforced on the runtime level, only that BCL does not expose an API to modify a string object, but we still can acquire a pointer to the memory where string object resides and change some bytes there in unsafe mode, but it can cause an hard-to-diagnose issues and you should probably use `StringBuilder` class instead*.

## The idea behind string interning

Since string object can't be changed once allocated, ideally we should have a **single copy** of all the **unique strings** of our program in memory, in other words, if we have **`str1`** and **`str2`** string variables and both have a **same value**, then both should have a **same reference** too (point to the same memory address), that is called **String interning**.

Believe it or not, you're already doing a string interning

```csharp
var str1 = "A";
var str2 = "A";

Console.WriteLine(object.ReferenceEquals(str1, str2)); // True
```

Even if `str1` and `str2` are declared seperately they still point to the same memory address, the reason is string interning done by CLR automatically for **string literals**.

This behaviour is well documented

> The Common Language Infrastructure (CLI) guarantees that the result of two ldstr instructions referring to two metadata tokens that have the same sequence of characters return precisely the same string object (a process known as "string interning"). [Source](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.ldstr?redirectedfrom=MSDN&view=netframework-4.8)

If we look at IL code for the first 2 lines of above c# code snippet

```csharp
IL_0001: ldstr        "A"
IL_0006: stloc.0      // str1

IL_0007: ldstr        "A"
IL_000c: stloc.1      // str2
```

we see 2 `ldstr` instruction that loads the exact same string value.

There is not much you can do to disable this behaviour for whatever reason, most you can do is

```csharp
[assembly: CompilationRelaxations(CompilationRelaxations.NoStringInterning)]
```

Which *Marks an assembly as not requiring string-literal interning*, keep in mind that string interning still can happen.

## How to intern a string manually

There are 2 methods for that

- `static string Intern (string str)` - interns `str` string and return a reference to interned string
- `static string IsInterned (string str)` - returns back an interned string if `str` string is interned, or `null` otherwise.

Let's see some examples

```csharp
// Literal string is going to be interned automatically
var str1 = "Hello Bond";

// Dynamic string will not be interned
var tmpStr = new StringBuilder().Append("Hello ").Append("Bond").ToString();

// string.Intern returned an existing interned string (remember, "Hello Bond" is already interned by CLR)
var str2 = string.Intern(tmpStr);

Console.WriteLine(object.ReferenceEquals(str1, str2)); // true
Console.WriteLine(object.ReferenceEquals(str1, tmpStr)); // false
```

The api for interning a string seems pretty straightforward.

## Saving a memory is not the only advantage

The idea of string interning is to reuse piece of memory to save some bytes but that's not all, string interning has even more important advantage, **faster string comparison**.

If `str1` and `str2` are interned strings and their references are equal then their values are equal too and vice versa (if `str1` and `str2` references are NOT equal then their values are NOT equal too). This is a really good news, because string comparison is frequent and expensive operation.

## How does it work internally

To make interning work a static hash table (`InternTable`) is maintained for keeping a track of all the interned strings in our application. The interning process is simple

- checks if such string is already interned
- If yes
    - return interned string
- If not
   - allocate new string in memory
   - copy the value to newly allocated string
   - add newly allocated string reference to hash table
   - return back the refernce to newly allocated string

## Drawbacks

We talked about the advantages, string deduplication and performant comparison, now it's time for disadvantages

### Ever living string object

String, once interned, stays in memory during the lifetime of our application, the hash table we mentioned above keeps a refernce to interned string which keeps GC from collecting it. So, for the strings which are not frequently used during the lifetime of our application, string interning most likely is not the right way of storing it, we may end up in a situation, where instead of saving up a memory we do an opposite.

### Temporary string

In order to get an interned string, we should have a "normal" string. As an example, say we want to intern a content of a file, first we should store the content into the string variable, `str` for exmaple and then call `str = string.Intern(str)`, which will internally create a **new string** and copy the value of `str` to it. As a result, we end up having 2 duplicate string objects in memory, one of which (guess which one) should be garbage collected.

## Conclusion

String interning can be used effectively for long living duplicate strings, specially if we're doing a bunch of equality comparisons.