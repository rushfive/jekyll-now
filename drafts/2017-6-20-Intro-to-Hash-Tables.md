---
layout: post
title: Introduction to Hash Tables
---

In the previous post, we discussed what an abstract `map` data structure is, and then implemented a simple unsorted version that exhibits poor running time for the core operations, such as `Get()`, `Set()`, etc.

We'll now explore `Hash Tables`, which happens to be one of the most efficient implementations for a map data structure. This post will dive into some issues that arise when trying to implement a faster map, and a couple of creative solutions to bypass them.

We'll be implementing a couple `Hash Tables` in the subsequent posts, but first we need to understand some of the inner workings of how it should be implemented. Lets dive in!

## What we need is an Array

An `array` exhibits many of the core features we'd like in a `HashTable`. It creates a mapping where an `integer index` *key* maps to a place in memory. One of the neater and fundamental features of an `array` is that you can immediately access any element it contains if you know its' index. This is because we already have a reference to the array itself, and so accessing any elements by an index is simply finding the offset from its head.

If our map was so specific in that we needed `integer keys` and `string values`, we might be able to simply utilize a `string[]` array.

For example, lets say we want to store these `Entries` in some map:

```javascript
{
    Key: 0,
    Value: 'a'
},
{
    Key: 3,
    Value: 'b'
},
{
    Key: 4,
    Value: 'c'
},
{
    Key: 9,
    Value: 'd'
}
```

If we were to use a `string[]` array, it would end up looking like this:

{: .table .table-bordered}
0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
---|---|---|---|---|---|---|---|---|---
a | | | b | c | | | | | d

And if you wanted to access an `Entry` (which is a `string` in this example) using an index as key, you can immediately get it in `O(1)` time simply by calling `array[i]`.

We'll be harnessing the efficiency of a simple ole `array` to create the `hash table`. In the previous post where we created an `UnsortedMap`, it too used an array internally to store `Entries`. However, the indices of the `array` were never really used in the core operations for faster run times. Being unsorted, we simply added a new `Entry` wherever the first open spot was, and then to find one, we'd walkthrough the entire array from start to finish until we found it.

If we can somehow take some generic key, which can be represented by any data type, and map it to an index in the *internal array* of `Entries`, we'll be able to find any item we'd like in the desired `O(1)` time.

## The Hash Function

The remainder of this post will revolve around a simple scenario. We have an array, `A`, that'll internally represent the `HashTable`:

{: .table .table-bordered}
0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
---|---|---|---|---|---|---|---|---|---
 | | |  |  | | | | | 

The items we'd like to store have `string` keys and `int` values:

```c#
public class Entry<string, int>
{
    public string Key { get; set; }
    public int Value { get; set; }
}
```

The objective is to somehow take a key, regardless of its' `data type`, and somehow map it to an index within the range of the *internal array* - which is precisely what the **hash function** does. A common implementation of a *hash function* is typically broken out into two steps or parts:

1. Obtaining the `hash code`, then
2. applying a `compression function` to the `hash code`

### The Hash Code

The process of *creating* a `hash code` is a fairly advanced topic and something we won't be taking a deep dive into. However, we do need to understand some things regarding it to ensure our `HashTable` implementation behaves correctly.

Given some key `k` and a function `h` that takes something and returns a `hash code`, the function should return the exact same `hash code` whenever `k` is used as the argument.

In our C# examples, we'll be making use of `GetHashCode()`, which is available on `object` to override as you'd like. Here's what happens when we run it on a `string`:

```c#
int hashCode1 = "test".GetHashCode(); // 87411476
int hashCode2 = "test".GetHashCode(); // 87411476

bool equals = "test".GetHashCode() == "test".GetHashCode(); // true
```

The two hash codes for `"test"` resulted in the same integer value being returned. In our example it happened to be `87411476`, but this value will likely be different every time a program is run because .NET relies on its' memory address to compute the value.

The `equals` test on the last line seems pretty straight-forward but stresses an important point. We need to ensure that if we're using `GetHashCode()`, it'll always return the same code for a given *key*. If .NET doesn't provide the right implementation for this then you'll need to override `GetHashCode()` to meet this requirement, or find another suitable method to obtain the codes.

As a side note, things get much trickier when trying to create a `GetHashCode()` implementation for some object, like lets say a `CustomList<T>`. We need to be able to able to use some property from the object to create it. If it has a uniquely identifying property, like a `Guid` id, then that's a suitable candidate. But if you don't, you may have to resort to using some combination of the values in that list to create it. This can cause issues because if the list is changed after the *key* has been created, then subsequent calls to `GetHashCode()` and `Equals()` will produce different results, and you may no longer be able to find your item from the map using the *key*.

The good news? It seems fairly uncommon to use these kinds of objects as keys in a map. I'm sure the use-cases exist, but I've yet to run into them. In my personal experience, the *keys* tend to be some value type such as a `Guid` or a `string`.

Alright, now that we've produced a `hash code`, what do we do with it? Lets use the code from the example above, `87411476`. Clearly this won't fit into our array's range, `[0, 9]` (inclusive). We need to transform (or *compress*) the value to fit within the range.

### The Compression Function