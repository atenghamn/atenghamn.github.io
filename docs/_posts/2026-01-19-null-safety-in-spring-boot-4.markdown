---
layout: post
title: "Null safety in Spring Boot 4"
date: 2026-01-18 22:10:00 +0100
categories: Java, SpringBoot
---

I've heard somewhere that the invention of the null reference in Java was a billion dollar misstake.
The reason for it beeing a "billion dollar misstake" would then be refering to all the crashes, NullPointerExceptions and overall
nastyness that this has caused production systems all over the world. And yeah, we've all been there, the NullPointerException hits you
like a rock in the head. The reason for this is how Java works with null where nullness is implicit. Meaning something may or may not
be null.

There are of course patterns to deal with this but what if you could just stop worrying alltogether? Well now we got a big step on the way towards that can 🎉

With Spring Boot 4 we get an implementation of [JSpecify](https://jspecify.dev/docs/api/org/jspecify/annotations/package-summary.html#nullness)
we can now say thats it's not null by default with @NullMarked annotation.

So how do you get this sweetness into your life?

It's actualy pretty straight forward, add a new (or update your old) package-info.java file like:

```java
@NullMarked
package com.your.path.here;

import org.jspecify.annotations.NullMarked;

```

Something to remember here tough is that sub-packages doesn't inherit the package-info file so they need to be declared per package.

Then if you need stuff to still possible return null

```java
@NullMarked
@Service
public class CanReturnNullService{

    @Nullable
    public Todo findById(Long id) {
        return todoRepository.findById(id).orElse(null);
    }

}

```

You can even use this for input parameters and mark them with @Nullable.

### Optional

By now you're probably wondering why you just can't use Optional everywhere instad.
In my opinion it's not an either or, have them both in your toolbelt but I think @NullMarked is specially good
in those cases where you don't want to brake an existing contract. Changing a return type breaks it.

```java
@Service
public class CanReturnNullService{

    // Doesn't break any contract
    @Nullable
    public Todo findById(Long id) {
        return todoRepository.findById(id).orElse(null);
    }

    // Breaks contract
    // if existing contract is
    // public Todo findById(Long id)
    public Optional<Todo> findById(Long id) {
        return todoRepository.findById(id);

  }

}

```

So use them both, they both have there place, on a new service/api/project I would probably use Optional
but since reality often involves existing contracts I think @NullMarked is a really good tool.
