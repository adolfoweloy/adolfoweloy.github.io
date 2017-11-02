---
layout: post
title:  "Thinking about functional programming"
date:   2017-01-15 12:44:00 -0300
lang: en
ref: functional_java
comments: true
categories: functional java
description: some thoughts on functional programming with Java8
---

Nowadays functional programming is not a novelty to anyone. Many developers are talking about it today either because of trends or real requirement. The question is that when I have worked with state machines which fires lots of events that can be captured by *listeners* spreaded through the project, I thought that the lowest decoupling achieved by Observers can lead into problems sometimes. In my humble opinion, when too much thing start being written basically using the **Observer** pattern, it becomes hard to find things in software when someone wants to understand what is going on. By knowing that there is some reactive frameworks which improves legibility in such event based applications, I started wondering about functional reactive programming (FRP).

But this post are not about reactive programming. So why am I talking about reactive programming? Well, before starting going deeper about functional programming lets take a look at what wikipedia says about [Functional reactive programming](https://en.wikipedia.org/wiki/Functional_reactive_programming):

> FRP is a programming paradigm for reactive programming (asynchronous dataflow programming) using the building blocks of functional programming.

So we need blocks of functional programming which are nothing more than combinators which are functions `filter`, `map` and `fold` that we will see more about later (fold function is the foundation for describing reduce as we will see later). So now, you know that it's important to start learning functional programming before start using a reactive framework. That was the reason so I want to learn functional programming.

There is lots of other reasons to start learning functional programming which includes problems with concurrency between threads and to better design data structures for object oriented code.

But does functional programming excludes OOP? That's the point! It is possible to use many functional programming concepts when writing code with OOP language. In addition, many people are inquiring themselves if they really need to model everything as an object in their projects. When dealing with big data volumes, will be viable to have a complex object graph? There is many cases where data retrieved from database could be managed as simple collections which could be handled in a functional way. So we have one more thing to motivate us toward functional programming: OOP isn't the silver bullet to solve every problems we have in software.

# What is functional programming?

Although this question was also answered by many people, I wrote here what I think about functional programming to contextualize the subject here! Furthermore I think it's hard to find direct and simple explanation.

Functional programming tries to bring the bases of math functions to software development. So functions are the basic element for functional programming as objects are the basic element for object oriented programming. As well as in math, functions must return the same output given the same input were provided. Through simple functions we can combine them to solve complex problems.

Languages as Java uses methods to represent functions, but the difference is that methods execute within the object context which they belong to. Functions are independent units which does not need to execute within the object context. Theoretically every object oriented languages provides the object by itself as a method parameter either implicitly or explicitly. That's the way we can reference to the object through `this` in Java. Python do it explicitly by requiring the programmer to declare methods with the first parameter to be `self` which is a reference for the object that current method belongs to. As an example of `self` usage take a look to the forwarding code:

```python
class User:
    def __init__(self, firstName, lastName):
        self.firstName = firstName
        self.lastName = lastName

    def name(self):
        return self.firstName + ' ' + self.lastName
```

# Main features
Because of the math functions nature, some features are embedded in how to write code using functional paradigm as explained as follows.

## Immutability
As math functions which does not change the argument values, functions written in code are not allowed to change the parameter values. So objects sent as parameter to functions should be immutable (as not providing setters or any method with side effects). Immutability is the key to avoid concurrency problems in a multi-threaded environment.

## Without side effects
Functions should not create side effects through the application or even to the object which they may belong to. By avoiding side effects we can avoid to create hard to find bugs.

## Declarative instead of imperative
Instead of expressing the code intention as a sequence of steps, one after the other, when using functional programming we write our code in a declarative way. A function which figures out the factorial is a good example to use for comparation between declarative and imperative ways. Lets take a look for the following code which computes the factorial of a number using iteration.
 
```java
public static long fact(long n) {
    long result = 1;
    if (n == 0) return 1;

    for (int i = 1; i <= n; i++) {
        result *= i;
    }

    return result;
}
```

And now using recursion, we can express our code in a declarative way:

```java
public static long fact(long n) {
    if (n <= 1) return 1;
    return fact(n - 1) * n;
}
```
Trying to understand the code above as a sequence of steps is tottaly infeasible. This should be interpreted as a rule expressed in a declarative way. Using such approach makes clear to understand that a number's factorial is the product of all elements that are less or equal to `n`.

## Lazy evaluation
Languages which does support for lazy evaluation, allows for interpreting code withouth the need to evaluate everything that is declared in an expression. Such feature allows for working with infinite data structures as natural numbers for example. As an example, lets take a look at the following code which represents a lazy sequence with the natural numbers:

```java
private LazySeq<Integer> naturals(int from) {
    return LazySeq.cons(from, () -> naturals(from + 1));
}
```

The expression `() -> naturals(from + 1)` wont be performed when someone call the `naturals` method. Such a **function** will be performed only when it is required to and is allowed to. If functions are defined to work in a lazy way with infinite data structures, we can use such a construction as follows (this example take the double of the first 10 natural numbers):

```java
LazySeq<Integer> first10Doubles = naturals(2).map(n -> n * 2).take(10);
```

A nice articl to read about it and uses Java as the main language for examples can be accessed in the blog post [Lazy sequences implementation for Java 8](https://dzone.com/articles/lazy-sequences-implementation). 

# Problems which functional programming can help to solve
- Race conditions: concurrency problem where different threads try to access shared content. 

- Maintenance: imaginig some function which sometimes returns a value and sometimes returns a different value for the same input how we can warrant that our application as a whole works? If someone cannot know if at least a unit of code works for a given input she wont be able to know if the whole system works.
 
# Data structures using functional programming

It's interesting to think that functional programming is not just about functions and immutable objects. The question is that we also need to think about data structures that are immutable and are supported by functional programming. Before creating our first data structure lets search for some definitions about data types. When thinking about functional programming we can think about concrete data types, algebraic data types and abstract data types. I think it's important to know such definitions because having enough vocabulary about what we are learning helps to better communicate and better depicts any concept.

## Concrete data types
They are the most basic data types used to compose more complex data types such as algebraic and abstract data types.

## Algebraic data types
Algebraic data types (thinking in abstract way) are types which can be created from applying product and sum operations through other types. But, here I don't want to say product and sum as we know in arithmetic. In this case, the product and sum are better related to the set theory. We can create any type compounding attributes withing a class. With Python we can create a tuple which is composed by elements defined by other types. Such examples are how we build a type through the product of attributes and they are know as **product types**. Types defined by sum operation can be emulated in Java by just extending some class creating a subtype which aggregates the attributes from the parent class. **Sum types** can also be defined by Lazy structures which can use functions which aren't performed immediately.

An example of algebraic data type in Java, can be given as follows:

```java
Optional<String> name;
```

With `Optional` we can work with a well defined data type which might contain some value or not. When working directly with `String` in Java, the programmer ends to be aware that a variable can have a content or can be null. The problem with `null` is that it isn't a `String`. It's like there is another type defining the same variable or attribute. **Attention here:** I 'm not saying to start using `Optional` everywhere because I really think it's not good for some cases. I particularlly use it for methods which really might not have what to return as an output. As a good example to use `Optional` I like to show a find method as `Optional<User> findByName(String name);`.

Lists are also algebraic data types that can be composed by another data types well defined.

For more about algebraic data types, I suggest you to take a look in the blog post [Probably Done Before](http://merrigrove.blogspot.com.br/2011/12/another-introduction-to-algebraic-data.html) because that was where I found an easy explanation about this topic for not inclined to math programmers.

## Abstract data types

Abstract data types can be understood as more generic types which leaves undefined parts as can be read in the link about Abstract Data Type from [wiki.haskell](https://wiki.haskell.org/Abstract_data_type). Thinking about object oriented programming, abstract data types does not limit the possible subtypes.

Generics in Java allows for abstract data types definitions because it leaves undefined some data types used by another type declared as a class.

# That's enough theory

Now lets take some practice through examples that will show the beauty of functional programming.

## Recursive linked list

The examples presented here are some examples I saw when I read the book [Functional Programming For Java Developers](https://www.amazon.com/Functional-Programming-Java-Developers-Concurrency/dp/1449311032). This example shows the detinition of an immutable linked list to build a collection of elements held in sequence. To do this, each node of the list has the attribute `head` that contains the value of the current element, and the attribute `tail` which references another list. What we are handling here is a recursive data structure with finite elements so we wont need laziness. As this is a finite list, it requires to define the end of the list and it is achieved by defining an empty list. With an empty list there is nowhere to go when iterating through the elements. The next figure shows how this list will be defined:

![Recursive list]({{ "/assets/recursive-list.png" | absolute_url }})

We need to represent the empty list and not empty list, so the following classes which are `EmptyRecursiveList` and `NonEmptyRecursiveList` both implement `RecursiveList` as we can see:

### __RecursiveList__ interface

```java
public interface RecursiveList<T> {

    T head();

    RecursiveList<T> tail();

    boolean isEmpty();

}
```

### __EmptyRecursiveList__ class
```java
public class EmptyRecursiveList<T> implements RecursiveList<T> {

    public EmptyRecursiveList() {
    }

    public T head() {
        throw new RuntimeException("the list is empty");
    }

    public RecursiveList<T> tail() {
        throw new RuntimeException("the list is empty");
    }

    public boolean isEmpty() {
        return true;
    }

}
```

### And __NonEmptyRecursiveList__ class
```java
 public class NonEmptyRecursiveList<T> implements RecursiveList<T> {

    private final T head;

    private final RecursiveList<T> tail;

    public NonEmptyRecursiveList(T head, RecursiveList<T> tail) {
        super();
        this.head = head;
        this.tail = tail;
    }

    @Override
    public T head() {
        return head;
    }

    @Override
    public RecursiveList<T> tail() {
        return tail;
    }

    @Override
    public boolean isEmpty() {
        return false;
    }

}
```

Ok, but if you want to create a list with some numbers you need to do something like as follows:

```java
RecursiveList<Integer> list =
    new NonEmptyRecursiveList<Integer>(1,
        new NonEmptyRecursiveList<>(2,
            new EmptyRecursiveList<>()));

```

It seems a bit confusing isn't it? So lets create the class `ListModule` which will provide better ways to define recursive lists through `factory methods`.

```java
public class ListModule {

    private static final RecursiveList<? extends Object> EMPTY 
        = new EmptyRecursiveList<Object>();

    public static <T> RecursiveList<T> list(T head, RecursiveList<T> tail) {
        return new NonEmptyRecursiveList<T>(head, tail);
    }

    @SuppressWarnings("unchecked")
    public static <T> RecursiveList<T> empty() {
        return (RecursiveList<T>) ListModule.EMPTY;
    }

}
```
And now, to create a list we can use a cleaner approach:

```java
ListModule.list(1, ListModule.list(2, ListModule.empty()));
```

But it can be better by using `import static` for `ListModule` class:

```java
list(1, list(2, empty()));
```

Until now this recursive linked list does not allow you to find an element or to iterate through all the elements. Given such requirements lets create some functional code by defining the method `forEach` within the `RecursiveList` interface. This method will allow you to iterate through the list elements.

```java
public interface RecursiveList<T> {

    T head();

    RecursiveList<T> tail();

    boolean isEmpty();

    void forEach(Consumer<T> consumer);
}
```

To create such a `forEach` in an interactive way (and imperative) we could use a simple `for` and for each iteration invoke the method `accept` from `consumer` functional interface. But doing such thing will just mask the imperative code. Instead of this lets use recursive approach as follows:

```java
@Override
public void forEach(Consumer<T> consumer) {
    consumer.accept(head());
    tail().forEach(consumer);
}
```

The code above implements `forEach` for `NonEmptyRecursiveList`. To implement it for the `EmptyRecursiveList` just do nothing because there is nothing to handle.

```java
@Override
public void forEach(Consumer<T> consumer) {
}
```

To start using this `forEach` we can use **lambda** which are available by Java 8. Using previous versions of Java would require the usage of anonymous classes. Such anonymous class could be implemented using the **Consumer** interface but this is in the past now! Lets use our functional approach as follows.

```java
RecursiveList<Integer> nums = list(1, list(2, empty()));
nums.forEach(item -> System.out.println(item));
```

And now? How to find specific values from this list? We can create the `filter` method which will search for elements that satisfies the predicate we send as a parameter.

```java
@Override
public RecursiveList<T> filter(Function<T, Boolean> f) {
    if (f.apply(head())) {
        return list(head(), tail().filter(f));
    }
    return tail().filter(f);
}
```

Implementing the filter for `NonEmptyRecursiveList` as presented above there is no need to imperatively iterate through elements to apply the predicate. All thing is done recursivelly in a declarative way. That is easy to read this code and understand that a new list will be created from elements that satisfies the function provided as parameter. For `EmptyRecursiveList` we can just return an `empty` list:

```java
@Override
public RecursiveList<T> filter(Function<T, Boolean> f) {
    return empty();
}
```

Lets use this filter to find just even numbers from a given list:

```java
RecursiveList<Integer> even = nums.filter((item) -> item % 2 == 0);
```

Although the code above has created a filtered list, how to retrieve an specific element? It would be a good idea if the even list provides a method to find the first element as follows:

```java
Integer num = even.findFirst().orElse(0);
```

If the even list is empty, trying to use `findFirst` might return an optional element which can either contain some value or nothing. To support this, `findFirst` should retrieve an `Optional<Integer>` so when there is no element in the list of even numbers a zero is returned.

Lets add this `findFirst` method to the  `RecursiveList` interface as follows:

```java
Optional<T> findFirst();
```

And the implementations for non empty list and empty list respectively:

```java
@Override
public Optional<T> findFirst() {
    return Optional.of(head());
}
```

```java
@Override
public Optional<T> findFirst() {
    return Optional.ofNullable(null);
}
```

Another interesting method which is part of combinators functions is the function `map` which allows to derive another list of different data type. The following code is the `NonEmptyRecursiveList` implementation:

```java
@Override
public <U> RecursiveList<U> map(Function<T, U> f) {
    return list(f.apply(head()), tail().map(f));
}
```

And for the empty list just return an empty list:

```java
@Override
public <U> RecursiveList<U> map(Function<T, U> f) {
    return empty();
}
```

If you are writing this sample codes as presented here do not forget to declare them in the `RecursiveList` interface. If you have any problems executing this code don't worry because all the examples are available for download at GitHub through the link [funcjava](https://github.com/adolfoweloy/funcjava).

As an example of map usage, lets print into the console the double of values present into some list:

```java
RecursiveList<Integer> numbers = list(1, list(2, list(3, empty())));

numbers
    .map((item) -> item * 2)
    .forEach(item -> System.out.println(item));
```

Another combinator function is `fold`. `fold` is the base for `reduce` function that is available in most languages. But why `fold`? I think it will be easy to understand the purpose of this function when thinking about the real meaning of the word. When executing the fold function it's like the code were folding the list until all values was reduced to just one value as presented in the next image.

![Folding]({{ "/assets/fold.png" | absolute_url }})

It's interesting to note the relation between fold and reduce. We kept folding the list until reducing it to a single value. The complete code follows presenting the `foldLeft` and `foldRight` for both non empty and empty list.

```java
public class NonEmptyRecursiveList<T> implements RecursiveList<T> {

    private final T head;

    private final RecursiveList<T> tail;

    public NonEmptyRecursiveList(T head, RecursiveList<T> tail) {
        super();
        this.head = head;
        this.tail = tail;
    }

    @Override
    public T head() {
        return head;
    }

    @Override
    public RecursiveList<T> tail() {
        return tail;
    }

    @Override
    public boolean isEmpty() {
        return false;
    }

    @Override
    public void forEach(Consumer<T> consumer) {
        consumer.accept(head());
        tail().forEach(consumer);
    }

    @Override
    public RecursiveList<T> filter(Function<T, Boolean> f) {
        if (f.apply(head())) {
            return list(head(), tail().filter(f));
        }
        return tail().filter(f);
    }

    @Override
    public Optional<T> findFirst() {
        return Optional.of(head());
    }

    @Override
    public <U> RecursiveList<U> map(Function<T, U> f) {
        return list(f.apply(head()), tail().map(f));
    }

    @Override
    public <T2> T2 foldLeft(T2 seed, BiFunction<T2, T, T2> f) {
        return tail().foldLeft(f.apply(seed, head()), f);
    }

    @Override
    public <T2> T2 foldRight(T2 seed, BiFunction<T, T2, T2> f) {
        return f.apply(head(), tail().foldRight(seed, f));
    }

}
```

```java
public class EmptyRecursiveList<T> implements RecursiveList<T> {

    public EmptyRecursiveList() {
    }

    public T head() {
        throw new RuntimeException("the list is empty");
    }

    public RecursiveList<T> tail() {
        throw new RuntimeException("the list is empty");
    }

    public boolean isEmpty() {
        return true;
    }

    @Override
    public void forEach(Consumer<T> consumer) {
    }

    @Override
    public RecursiveList<T> filter(Function<T, Boolean> f) {
        return empty();
    }

    @Override
    public Optional<T> findFirst() {
        return Optional.ofNullable(null);
    }

    @Override
    public <U> RecursiveList<U> map(Function<T, U> f) {
        return empty();
    }

    @Override
    public <T2> T2 foldLeft(T2 seed, BiFunction<T2, T, T2> f) {
        return seed;
    }

    @Override
    public <T2> T2 foldRight(T2 seed, BiFunction<T, T2, T2> f) {
        return seed;
    }

}
```

Now, using this kind of list, we can perform different operations using basic combinators to build flexible code that is enough to change according to some specific rule.

Just to present the `fold` usage, take a look at the following code which retrieves all even numbers, double them and sum up them all at the end:

```java
RecursiveList<Integer> values = list(1, list(2, list(3, list(4, empty()))));

Integer total = values
    .filter((n) -> n % 2 == 0)
    .map((n) -> n * 2)
    .foldRight(0, (n, m) -> n + m);
```

# Summary

It's important to note that the most things implemented here is already provided by Java 8 through `streams` API. But there is an important difference. Theses combinators functions presented here are not `lazy`, because these functions executes some logic through the whole list as soon they are invoked. So if we chain the functions `map` and `filter` such operations will execute immediatelly when invoked losing some performance. The Java 8 streams API supports **laziness** which can improve performance depending on situation. If you want to know more about laziness in Java 8, I suggest you to read [Java 8 Streams API - Laziness and Performance Optimization](http://java.amitph.com/2014/01/java-8-streams-api-laziness.html#.WHuBHLGZORs).

That's all. This post is the result of some days of functional programming studying. Any problems found, you can send a suggestion by comments. In case of any problems of my english version I am waiting for suggestions to better write.
