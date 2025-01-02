---
layout: post
title: "Immediate or deferred execution"
date: 2024-11-02 20:00:00 +0200
categories: C#, LINQ
---
When I first started using LINQ I was a bit confused why I sometimes didn't get back the result I expected. The thing I didn't now was that LINQ uses **deferred execution**. This means that the query is not executed until the result is actually needed. This is a good thing because it allows us to build up complex queries without executing them until we actually need the result.

For someone coming from a different background this might be a bit confusing so lets have a look and see whats going on. 
 

In our example we are creating something that can help us sort tarantulas in case they are poisonous or not. We start of with a class called Tarantulas that looks like this:

```csharp
public sealed class Tarantula
{
    public string Name { get; set; }
    public bool IsNewWorld { get; set; }
    public bool IsPoisonous { get; set; }
}
```
If we then create a list of various tarantulas, and filter out the non venomous ones before printing the result we get something like this: 

```csharp
List<Tarantula> tarantulas =
[
    new Tarantula
    {
        Name = "Poecilotheria regalis",
        IsNewWorld = false,
        IsPoisonous = true
    },
    new Tarantula()
    {
        Name = "Aphonopelma chalcodes",
        IsNewWorld = true,
        IsPoisonous = false
    },
    new Tarantula
    {
        Name = "Brachypelma hamorii",
        IsNewWorld = true,
        IsPoisonous = false
    },
    new Tarantula
    {
        Name = "Tliltocatl albopilosus",
        IsNewWorld = true,
        IsPoisonous = false
    }
];

var poisoniusTarantulas = tarantulas.Where(spider => spider.IsPoisonous);

Console.WriteLine("Dont pet: ");
foreach (var tarantula in poisoniusTarantulas)
{
    Console.WriteLine(tarantula.Name);
}
```

So far it's all pretty straight forward. But what if we add a new tarantula to the list after we have filtered out the non venomous ones? 

```csharp
List<Tarantula> tarantulas =
[
    new Tarantula
    {
        Name = "Poecilotheria regalis",
        IsNewWorld = false,
        IsPoisonous = true
    },
    new Tarantula()
    {
        Name = "Aphonopelma chalcodes",
        IsNewWorld = true,
        IsPoisonous = false
    },
    new Tarantula
    {
        Name = "Brachypelma hamorii",
        IsNewWorld = true,
        IsPoisonous = false
    },
    new Tarantula
    {
        Name = "Tliltocatl albopilosus",
        IsNewWorld = true,
        IsPoisonous = false
    }
];

var poisoniusTarantulas = tarantulas.Where(spider => spider.IsPoisonous);
var poisonius = tarantulas.GetPoisonousTarantulas();
tarantulas.Add(
    new Tarantula
    {
        Name = "Pelinobius muticus",
        IsNewWorld = false,
        IsPoisonous = true
    });

Console.WriteLine("Dont pet: ");
foreach (var tarantula in poisoniusTarantulas)
{
    Console.WriteLine(tarantula.Name);
}
```
This will print 
```
Dont pet: 
Poecilotheria regalis
Pelinobius muticus
```

This is because the query is not executed until we actually need the result. If we want to execute the query immediately we can use the `ToList()` method. 

```csharp
var poisoniusTarantulas = tarantulas.Where(spider => spider.IsPoisonous).ToList();
```
and then our output will be 

```csharp
Dont pet:
Poecilotheria regalis
```
Strange? Not really, just remember that LINQ uses deferred execution, think of it as lazy loading, query will not be called until needed.

To make it even more clear we can create an extension method that prints the name and then yield returns the poisonous tarantulas. 

```csharp
public static class SpiderExtensions
{
    public static IEnumerable<Tarantula> GetPoisonousTarantulas(this IEnumerable<Tarantula> tarantulas)
    {
        foreach (var tarantula in tarantulas)
        {
            Console.WriteLine($"{tarantula.Name} is sorted here");
            if (tarantula.IsPoisonous)
            {
                yield return tarantula;
            }
        }
    }
}
```

And then we can call it like this: 

```csharp
var poisonius = tarantulas.GetPoisonousTarantulas();
```

this will print 
```
Aphonopelma chalcodes
Brachypelma hamorii is sorted here
Brachypelma hamorii
Tliltocatl albopilosus is sorted here
Tliltocatl albopilosus
Pelinobius muticus is sorted here
Pelinobius muticus
```

If we change it to use the `ToList()` method to execute the query immediately. 

```csharp
var poisonius = tarantulas.GetPoisonousTarantulas().ToList();
```

this will output 
```
Poecilotheria regalis is sorted here
Aphonopelma chalcodes is sorted here
Brachypelma hamorii is sorted here
Tliltocatl albopilosus is sorted here
Dont pet: 
Poecilotheria regalis
```	

Finally we can use the both sorting methods but only call the list that didn't use the extension method. We see that the extension method is never called since it doesn't print anything. 

```csharp
public sealed class Tarantula
{
    public string Name { get; set; }
    public bool IsNewWorld { get; set; }
    public bool IsPoisonous { get; set; }
}

public static class SpiderExtensions
{
    public static IEnumerable<Tarantula> GetPoisonousTarantulas(this IEnumerable<Tarantula> tarantulas)
    {
        foreach (var tarantula in tarantulas)
        {
            Console.WriteLine($"{tarantula.Name} is sorted here");
            if (tarantula.IsPoisonous)
            {
                yield return tarantula;
            }
        }
    }
}

List<Tarantula> tarantulas =
[
    new Tarantula
    {
        Name = "Poecilotheria regalis",
        IsNewWorld = false,
        IsPoisonous = true
    },
    new Tarantula()
    {
        Name = "Aphonopelma chalcodes",
        IsNewWorld = true,
        IsPoisonous = false
    },
    new Tarantula
    {
        Name = "Brachypelma hamorii",
        IsNewWorld = true,
        IsPoisonous = false
    },
    new Tarantula
    {
        Name = "Tliltocatl albopilosus",
        IsNewWorld = true,
        IsPoisonous = false
    }
];

var poisoniusTarantulas = tarantulas.Where(spider => spider.IsPoisonous);
var poisonius = tarantulas.GetPoisonousTarantulas();
tarantulas.Add(
    new Tarantula
    {
        Name = "Pelinobius muticus",
        IsNewWorld = false,
        IsPoisonous = true
    });

Console.WriteLine("Dont pet: ");
foreach (var tarantula in poisoniusTarantulas)
{
    Console.WriteLine(tarantula.Name);
}

```

Which will output  
```
Dont pet: 
Poecilotheria regalis
Pelinobius muticus
```	
Hope use had some use of this post and just for the record, Pelinobius muticus isn't that poisonous, but really defensive (and big) and I couldn't think of any other tarantula that was poisonous :D 



