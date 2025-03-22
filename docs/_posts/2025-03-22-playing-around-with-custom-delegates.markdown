---
layout: post
title: "Playing around with custom delegates"
date: 2025-03-22 22:21:00 +0200
categories: C#
---

I recently played around with functional interfaces in Java. C# as you know, doesn't have functional interfaces, the closest thing being delegates and lambdas. So how can you create something similar to

```Java
@FunctionalInterface
interface CheckTwoObjects {
    boolean check(Object o1, Object o2);
}
```

In C# we can use Func<T, T, T>, Action<T> and lambdas to achive the same thing.

```csharp
    public static bool CompareTwoComparableObjects<T1, T2> (T1 t1, T2 t2,  Func<object, object, bool> comparison, Func<object, object, bool> typeCheck) => comparison(t1, t2) && typeCheck(t1, t2);
    
    public static bool AreEqual(object o1, object o2) => o1.Equals(o2);
    
    public static bool AreIntegers(object o1, object o2) => o1 is int && o2 is int;
```

Here we have three methods. The first one takes two generic objects and two functions that both takes two objects and return boleans. As you probably guessed we are going to put the last two methods in the first one like this:

```csharp
var result = CompareTwoComparableObjects("someValue", "someOtherValue", AreEqual, AreIntegers);
```

This simple method makes it possible for us to create something very similar to Javas functional interfaces and we can use the method like: 

```csharp
var a = "x";
var b = 5;
var checkTheseTwo = CompareTwoComparableObjects(a, b, AreEqual, AreIntegers);
Console.WriteLine($"{a} and {b} have the same value and are numbers : {checkTheseTwo}");
```

You can of course write any combination of methods, I choose simple predicates just to keep it simple in this example but hey, go wild :) 