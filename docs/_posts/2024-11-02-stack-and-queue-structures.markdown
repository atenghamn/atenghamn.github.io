---
layout: post
title: "Stack and Queueu data structures"
date: 2024-11-02 11:40:00 +0200
categories: datastructures
---

So you've just started out your programming journey and you probably heard about the stack in some way or another, maybe you've visited [StackOverflow](https://stackoverflow.com/) and wondered about the name. Maybe you heard a term like call stack or even looked at one while debugging. In this post I thought it would be fun to look at two different data structures that are pretty foundational to how your computer handles memory, the stack and the queue data structures. I will have my examples in C# but the data structure itself isn't tied to any programming language.

## The stack data structure

If we imagine a stack of records (not a C# record but a vinyl record) and we have three vinyl records: at the bottom we have Miles Davis - Kind of blue, in the middle we have Nas - Illmatic and at the top we have Toots and the Maytals - In the dark. So far so good, but now we want to listen to Illamtic. Since it's in the middle we first have to remove In the dark to access Illmatic. If we instead want to listen to Kind of blue we have to remove both In the dark and Illmatic since it's at the bottom. This is the core principle of the stack! We can also put it as first in - first out **FILO**. Lets say we got a stack of numbers. Like 1, 2, 3, 4 and 5 and we want to access 3. Then we have to remove 5 and 4. so our new stack is 1, 2 and 3. The process of removing from a stack is called **poping** and if we add to the stack we call that **pushing**. 

If we return to our stack of three vinyl records, lets say that we have built ourself some kind of vinyl record storage. Since we only had three records and we're out of money we only build or storage for three records. One day a friend comes and gives us a fourth vinyl record. We try and put it in the storage but remember, we only built it for three vinyl records so when we add the fourht theres no room for it. Now we got a **stack overflow**. 

If we instead would sell all of our vinyl records and then forgets about it and we go to our storage to get a vinyl record but it's empty, then we got a **stack underflow**.

## Stack code example

For this example I worked with a linked list. A linked list is a list of nodes where each node has a value and a reference to the next node. If we have a linked list of three letters A, B and C.

The first node A has the value of A and a reference to B.

The second node has the value of B and a reference to C.

The third node has the value of C and a null reference.

```csharp
public class StackLinkedList
{
    private static LinkedListNode<int>? _first = null;
    public static LinkedList<int> List = [];
}
```

as you can see our stack class consists of a linked list and a linked list node called _first that will serve as our reference to where we are in the stack. Lets add methods for pop and push operations on the stack 

```csharp
    
    // our Push method takes a value
    public void Push(int value)
    {
        // we create a new node to hold our value
        var newNode = new LinkedListNode<int>(value);
        // we assign our _first node the value
        _first = newNode; 

        // and finally adds it to the stack
        List.AddLast(newNode);
    }
    

    public void Pop()
    {
        // we create a temporary node to print we value we remove (this is not neccessary if you dont care what is removed)
        var temp = _first;

        // if _first is null and we try to remove we get a stack underflow
        if (_first is not null) 
        { 
            // we make the node behind first node the new first node (ideally we would like to check so there's an actual node behind too but I tried to make it as concise as possible )
            _first = _first.Previous;

            // them we remove the last element of the stack
            List.RemoveLast();
        }
        // this is just for visualisation later on
        if (_first != null)
        {
          Console.WriteLine($"Removed element {temp.Value});  
        };
    }
    
```

Now we have implemented pop and push operations and our stack is pretty much complete. To make it easier to follow I will add a method to print our stack to the console.

```csharp
   public void PrintStack()
    {
        var currentNode = _first;
        while (currentNode != null)
        {
            Console.WriteLine(currentNode?.Value);
            currentNode = currentNode?.Previous;
        }
        
    }
```

So lets look at our stack class
```csharp
public class StackLinkedList
{
    private static LinkedListNode<int>? _first = null;
    public static LinkedList<int> List = [];
    
    
    public void Push(int value)
    {
        var newNode = new LinkedListNode<int>(value);
        _first = newNode; 
        List.AddLast(newNode);
    }
    
    public void Pop()
    {
        var temp = _first;
        if (_first is not null) 
        {
            _first = _first.Previous;
            List.RemoveLast();
        }
        if (_first != null) Console.WriteLine($"Removed element {temp.Value}");
    }
    
    public void PrintStack()
    {
        var currentNode = _first;
        while (currentNode != null)
        {
            Console.WriteLine(currentNode?.Value);
            currentNode = currentNode?.Previous;
        }
        
    }
}
```

We have a list that we can add (pop) and remove (push) from. Let's see it in action. 

```csharp
var stackLinkedList = new StackLinkedList();
stackLinkedList.Push(1);
stackLinkedList.Push(2);
stackLinkedList.Push(3);
stackLinkedList.PrintStack();
stackLinkedList.Pop();
stackLinkedList.PrintStack();
```

This will print

```
3
2
1
Removed element 3
2
1
```

As we see we always travel from front to back (or top to bottom). First in -> Last out. This is the **FILO** principle. 

## The queue data structure

What if we rather would like to have a first in -> first out order **FIFO**? Then we can use the queue structure. We can imagine a regular queue of people in a store. The first person in line will be the first to be served. If we have a queue of five people: Anna, Mohammed, Lisa, Shawn and Aila. For Lisa to be served we have to serve Anna and Mohammed first. If we add another person to our queue the we are doing an **enqueue** and if Anna has gotten served and is removed from the queue then have done a **dequeue**. Remember first in - first out. Just as the stack if our queue are full and we try to add another element then we get an overflow. If our queue are empty and we try to remove another item then we get an underflow.

## Queue code example 

Once again we will work with a linked list to make it easy to follow. 

```csharp
public class QueueLinkedList
{
    private static LinkedListNode<int>? _first = null;
    private static LinkedListNode<int>? _last = null;
    public static LinkedList<int> List = [];
}
```
The difference here comapared to the stack is that we now has two references. The first and the last node of the queue.

To peform enqueue and dequeue operations we add two methods.

```csharp
// we input the value we want to add
public void Enqueue(int value)
    {
        // we create a new node to hold our value
        var newNode = new LinkedListNode<int>(value);

        // if the first node is empty both first and last node will hold the same value
        _first ??= newNode;
        _last = newNode; 

        // we add the node at the end of our list
        List.AddLast(newNode);
    }
    
    public void Dequeue()
    {
        // we create a temporary node for printing only so this isn't anything you need to do
        var temp = _first;
        if (_first is not null) 
        {
            // here we also should check that we have a node second to first but to make it easier to follow I skiped that step instead we assign the second node as first
            _first = _first.Next;

            // we remove the first node
            List.RemoveFirst();
        }
        // we print out the result
        Console.WriteLine($"Removed element {temp.Value}");
    }
```

Finally we create a method to print out que.

```csharp
 public void PrintQueue()
    {
        var currentNode = _first;
        while (currentNode != null)
        {
            Console.WriteLine(currentNode?.Value);
            currentNode = currentNode?.Next;
        }
    }
```

So our complete class will look like:

```csharp
public class QueueLinkedList
{
    private static LinkedListNode<int>? _first = null;
    private static LinkedListNode<int>? _last = null;
    public static LinkedList<int> List = [];
    
    
    public void Enqueue(int value)
    {
        var newNode = new LinkedListNode<int>(value);
        _first ??= newNode;
        _last = newNode; 
        List.AddLast(newNode);
    }
    
    public void Dequeue()
    {
        var temp = _first;
        if (_first is not null) 
        {
            _first = _first.Next;
            List.RemoveFirst();
        }
        Console.WriteLine($"Removed element {temp.Value}");
    }
    
    public void PrintQueue()
    {
        var currentNode = _first;
        while (currentNode != null)
        {
            Console.WriteLine(currentNode?.Value);
            currentNode = currentNode?.Next;
        }
    }
}


```

Lets see it in action! 
```csharp
var queueLinkedList = new QueueLinkedList();
queueLinkedList.Enqueue(1);
queueLinkedList.Enqueue(2);
queueLinkedList.Enqueue(3);
queueLinkedList.PrintQueue();
queueLinkedList.Dequeue();
queueLinkedList.PrintQueue();
```
We set it up the same way as we did with the stack but now it will print 
```
1
2
3
Removed element 1
2
3
```

## Summary
So this was the stack and the queue structures. To summarize:

The stack works with **FILO** -> first in, last out. 

The Queue works with **FIFO** -> first in, first out! 

If we add to add stack we are **pushing**. 

If we remove from the stack we are **popping**.

If we add to the queue we are doing an **enqueue**.

If we remove the from the queue we are doing an **dequeue**.

Until next time, tack care!