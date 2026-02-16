
---

layout: post
title: "Railway oriented programming with Java"
date: 2026-02-16 22:00:00 +0100
categories: Java, Patterns
---

The term railway oriented programming have been around a while and you can (and should) check it out since there's a lot of great resources
out [there](https://fsharpforfunandprofit.com/posts/recipe-part2/). But many of these examples are written in functional languages so I thought it would be fun to give some examples in Java.

First thing first, what is railway oriented programming?  In short it's a way of handling the control flow in a more functional matter, think - less hard to follow control flow and a more conveyor belt like way of handling your applications flow. The railroad part is a great way of building a mental model, instead of
having our control flow like a busy main street with intersections and roundabouts we can think of it as a train track where we have our track
(our happy path) but it can be derailed leading to an exception. A close relative to this thought is a monad but lets not go there now ;)

### Result<T>

In the heart of our Java implementation we're going to start with a interface that will look like this

```Java
public sealed interface Result<T> permits Success, Failure {
}
```

Here we have a sealed interface that only permits Success and Failure. As you've probably guessed Success and Failure will be the two types that carries our result.

```Java
public record Success<T>(T value) implements Result<T> {
}

```

Our Success record holds a value of T (bare with me if you're allergic to Generics, you can implement a type but as you will see, there are huge benefits with sticking to Generics).

```Java
public record Failure<T>(String error) implements Result<T> {
}
```

Our Failure record only hold a error message. So this far it's really straight forward, but now we're going to add some functionality to our Result<T> interface so we can use it to implement the railway pattern.

We start of with some static helpers so we can return success or failure, this is nice to haves and we could live without it, but it will come quite handy as you will see.

```java
    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }

    static <T> Result<T> failure(String error) {
        return new Failure<>(error);
    }
```

The next step is to implement a fold method whose only job is to say:

- if I'm success -> return onSuccess
- if I'm failure -> return onFailure

To make it do this we create a method with a really shitty(or at least hard to read - sorry this was the best I could come up with) method signature.

```Java
<R> R fold(Function<? super T, ? extends R> onSuccess,
                       Function<? super String, ? extends R> onFailure)
```

It basically takes two functions, one for success and one for failure and they both return the same type R.
With that reading pain out of the way we can look at the whole function:

```Java
 default <R> R fold(Function<? super T, ? extends R> onSuccess,
                       Function<? super String, ? extends R> onFailure) {
        if (this instanceof Success<?> s) {
            T value = (T) s.value();
            return onSuccess.apply(value);
        }
        Failure<?> f = (Failure<?>) this;
        return onFailure.apply(f.error());
    }
```

Here we check if it's an ```instanceof``` Success (remember the record), then run the onSuccess function. Otherwise run the onFailure function.

I don't blame you if you're confused now, but when we combine this with a bind function (sometimes called a flatMap) then it gets useful

```java
  default <U> Result<U> bind(Function<? super T, ? extends Result<U>> next) {
        return fold(
                next,
                Result::failure
        );
    }
```

So bind takes a success value (T) and returns another result with the help of fold. This makes it possible to chain methods together to create a chain (or a track).

Our complete Result<T> now looks like this

```Java
public sealed interface Result<T> permits Success, Failure {

    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }

    static <T> Result<T> failure(String error) {
        return new Failure<>(error);
    }

    default <R> R fold(Function<? super T, ? extends R> onSuccess,
                       Function<? super String, ? extends R> onFailure) {
        if (this instanceof Success<?> s) {
            T value = (T) s.value();
            return onSuccess.apply(value);
        }
        Failure<?> f = (Failure<?>) this;
        return onFailure.apply(f.error());
    }

    default <U> Result<U> bind(Function<? super T, ? extends Result<U>> next) {
        return fold(
                next,
                Result::failure
        );
    }
}

```

### So what should I do with this then?

Sorry for my extremly bad imagination but lets say we have a really weird utility class that we use for checking if an Integer is positive. Instead of having a lot of if-checks we can simply use our Result<T> to create a railroad pattern

```Java
public class PositiveIntegerUtil {

    public static Result<Integer> validate(Integer candidate){
        return isNull(candidate)
            .bind(PositiveIntegerUtil::isPositive);
    }

    private static Result<Integer> isNull(Integer candidate){
        return candidate == null
            ? Result.failure("Number cannot be null")
            : Result.success(candidate);
    }

    private static Result<Integer> isPositive(Integer candidate){
        return candidate < 0
            ? Result.failure("Number must be positive")
            : Result.success(candidate);
    }

}
```

It's a stupid example but lets pretend that we had more checks to do, it's not that hard to see how this could be really useful to chain all the validation logic together with an easy short-circut if something breaks.

Just to get a clear sense of the usage we can write some tests to see

```Java
package util;

import org.example.result.Failure;
import org.example.result.Success;
import org.example.util.PositiveIntegerUtil;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertInstanceOf;

public class PositiveIntegerUtilTest {

    @Test
    public void givenANegativeInteger_whenValidating_thenFailure() {
        var result = PositiveIntegerUtil.validate(-1);
        assertInstanceOf(Failure.class, result);

        var failure = (Failure<Integer>) result;
        assert(failure.error().contains("must be positive"));
    }

    @Test
    public void givenANullInteger_whenValidating_thenFailure() {
        var result = PositiveIntegerUtil.validate(null);
        assertInstanceOf(Failure.class, result);

        var failure = (Failure<Integer>) result;
        assert(failure.error().contains("Number cannot be null"));
    }

    @Test
    public void givenAPositiveInteger_whenValidating_thenSuccess() {
        var result = PositiveIntegerUtil.validate(1);
        assertInstanceOf(result.getClass(), result);

        var success = (Success<Integer>) result;
        assert(success.value() == 1);
    }
}

```

A real win we get from having our Result<T> interface is that we can reuse it for any type so if we want a new equally stupid String utility class we can create one
without changing anything on Result<T>

```Java
package org.example.util;

import org.example.result.Failure;
import org.example.result.Result;
import org.example.result.Success;

public class StringUtil {

    public static Result<String> validate(String candidate){
        return isNull(candidate)
            .bind(StringUtil::isEmpty);
    }

    private static Result<String> isEmpty(String candidate){
        return candidate.isEmpty()
            ? new Failure<>("String cannot be empty")
            : new Success<>(candidate);
    }

    private static Result<String> isNull(String candidate){
        return candidate == null
            ? new Failure<>("String cannot be null")
            : new Success<>(candidate);
    }
}

```

```Java
package util;

import org.example.result.Failure;
import org.example.result.Success;
import org.example.util.StringUtil;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertInstanceOf;

public class StringUtilTest {

    @Test
    public void givenAString_whenValidating_thenSuccess() {

        var result = StringUtil.validate("racecar");
        assertInstanceOf(Success.class, result);

        var success = (Success<String>) result;
        assertEquals("racecar", success.value());
    }

    @Test
    void givenNull_whenValidating_thenFailureWithMessage() {
        var result = StringUtil.validate(null);

        assertInstanceOf(Failure.class, result);
        var failure = (Failure<String>) result;
        assertEquals("String cannot be null", failure.error());
    }

    @Test
    void givenEmptyString_whenValidating_thenFailureWithMessage() {
        var result = StringUtil.validate("");

        assertInstanceOf(Failure.class, result);
        var failure = (Failure<String>) result;
        assertEquals("String cannot be empty", failure.error());
    }
}

```

### Summary

So should I never write a if statement again? Yes you probably should but this pattern can come really handy when the control flow starts to get a bit to nested and the logic get hard to follow. That's all for this time, hope you found something useful!
