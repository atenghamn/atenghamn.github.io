---
layout: post
title: "The ref and out keywords in C#"
date: 2024-07-24 18:05:00 +0200
categories: csharp
---

Recently I've played around with Rust a bit after having listened to a really inspiring talk by [Sebastian Nilsson](https://sebnilsson.com/) on Rust. One of the things that sets Rust apart is the concept of ownership. I don't know enough to write about Rust, but I thought it can be nice to have a short explanation of the `ref` and `out` keywords in C# that have some similarities with the ownership concept.

# ref keyword

So let's have a short breakdown on how the `ref` keyword works. 

We start with an example:

```csharp
// starting value
var originalValue = 5;

// If we send it in as a value, we make a copy of the value. This will
// not affect our original value.
int ValueMethod(int number) => number + 1;

// ValueMethod will now return a new integer that will be 6, not affecting our original integer 
// which will still be 5
var valueReturnedFromMethod = ValueMethod(originalValue);

Console.WriteLine($"Our original value: {originalValue}"); // 5
Console.WriteLine($"Our value returned from method: {valueReturnedFromMethod}"); // 6

// Now we send in a reference to our original value, which will affect the original value.
void ValueByReference(ref int number)
{
    number--;
}

ValueByReference(ref originalValue);
Console.WriteLine($"Our original value: {originalValue}"); // 4
```
Here we start with a value that is set to 5. If we call a method by value like our ValueMethod, then we will not alter our original value as shown.

But what if we pass in a value by reference with the ref keyword? Then we will reference our original value. As you can see, our method doesn't return anything, but our original value is changed.

This works for both value types like integers and reference types like strings, as shown below.

```csharp
var originalString = "I'm a string!";

// This will make a copy of our original string and add some more text to it,
// but it will not alter our original string.
string GiveMeANewString(string someText) => $"{someText} and we added some stuff";
Console.WriteLine($"Our original string: {originalString}");

var ourNewString = GiveMeANewString(originalString);
Console.WriteLine($"Our new string: {ourNewString}");

void ChangeOurOriginalString(ref string someText)
{
    someText = "And like witchcraft we changed our original string instead";
}

ChangeOurOriginalString(ref originalString);
Console.WriteLine($"Our original string: {originalString}");
```

That's really cool, huh!

# out keyword
A relative to the ref keyword is the out keyword. Here we can pass in values that don't even have to be initialized and get them returned to the original variable.

```csharp
var anotherNumber = 1;

void OutMethod(out int number)
{
    number = 2;
}
Console.WriteLine($"Before calling method: {anotherNumber}");
OutMethod(out anotherNumber);
Console.WriteLine($"After calling the method: {anotherNumber}");
```
Here we can see that if we send in an out parameter, the variable used for the parameter changes. The same goes for reference types like strings.

```csharp
var anotherString = "First";

void OutStringMethod(out string text)
{
    text = "Second";
}
Console.WriteLine($"Before calling method: {anotherString}");
OutStringMethod(out anotherString);
Console.WriteLine($"After calling the method: {anotherString}");
```
However, our value must have a value before being returned.

```csharp
int unInitializedValue;

ThisWontWork(out unInitializedValue);

// Since we don't have a value, the compiler will complain that
// Out parameter 'number' might not be initialized before accessing it.
void ThisWontWork(out int number)
{
    number++;
}

ThisWillWork(out unInitializedValue);

// This will work since we assign a value before returning.
void ThisWillWork(out int number)
{
    number = 1;
    number++;
}
Console.WriteLine($"This is now 2: {unInitializedValue}");

```

This was a pretty short explanation of the workings of the ref and out keywords. Hope you can find some use for it :)