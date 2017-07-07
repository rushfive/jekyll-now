---
layout: post
title: Chain Hash Map (Separate Chaining Scheme)
---

**Separate Chaining** is a method of collision handling where entries are stored in a *bucket*. Rather than having a slot in the internal array store just a single entry, we can now store a variable amount. This handles collisions by allowing different hashed keys to be stored in the same slot, although we still want to minimize the number of entries in a bucket. Ideally we want to store, at most, a single entry per bucket.

### The Load Factor

The *load factor*, a significantly important statistic for hash tables, is defined as a simple ratio:

`n / k`

where `n` is the number of entries, and `k` is the number of buckets. As the value of the load factor increases, so does the number of collisions, which ultimately results in slower run times for the map operations. Many experiments and studies have been done that suggest these load factors:

- For **Separate Chaining** schemes, a load factor of 0.9 or lower
- For **Open Addressing** schemes, a load factor of 0.5 or lower (explored in the next post)

### What's a bucket?

A *bucket* is simply an auxiliary data structure that is used to store multiple entries in that slot. Linked lists and simple maps are commonly used, but it could really be anything that can store multiple items.

<!--more-->

Having a bucket allows us to handle collisions, but ideally we want to only store one item per bucket on average. This is because if there's more, finding an item by key requires us to walkthrough the bucket to find it, and we're potentially back to linear run times for those kinds of operations.

### Implementing the ChainHashMap

We'll use the previously implemented `UnsortedMap` data structure to represent the buckets. A `ChainHashMap` is obviously a map as well, so we'll have it implement the `IMap<TKey, TValue>` interface, copied below for reference:

```c#
public interface IMap<TKey, TValue>
{
    int Size();
    bool IsEmpty();
    TValue Get(TKey key);
    void Set(TKey key, TValue value);
    bool Remove(TKey key);
    TKey[] Keys();
    TValue[] Values();
    Entry<TKey, TValue>[] Entries();
}
```

The map has three private members:

```c#
private UnsortedMap<TKey, TValue>[] internalArray { get; set; }
private int size { get; set; } = 0;
private int capacity { get; set; }
```

We keep track of the current size so we know when to resize the `internalArray`, which is an array of `UnsortedMap`s.

Constructors are as follows:

```c#
public ChainHashMap(int capacity)
{
    this.capacity = capacity;
    this.internalArray = new UnsortedMap<TKey, TValue>[capacity];
}

public ChainHashMap() : this(5)
{
}
```

I have the default constructor set the capacity to `5` for demo purposes. I've built an interactive console tester that goes along with my implementations (GITHUB_LINK_HERE), so it's easier to visualize the map with fewer entries.

Here's `Entries()`:

```c#
public Entry<TKey, TValue>[] Entries()
{
    var entries = new Entry<TKey, TValue>[this.size];

    int index = 0;
    for (int i = 0; i < this.size; i++)
    {
        var map = this.internalArray[i];
        if (map != null)
        {
            var mapEntries = map.Entries();
            for (int j = 0; j < mapEntries.Length; j++)
            {
                entries[index] = mapEntries[j];
                index++;
            }
        }
    }

    return entries;
}
```

The method walks through the internalArray, and if a bucket exists, walks through it and grabs the entries.

The `Keys()` and `Values()` methods are very similar:

```c#
public TKey[] Keys()
{
    var keys = new TKey[this.size];

    int index = 0;
    for (int i = 0; i < this.internalArray.Length; i++)
    {
        var map = this.internalArray[i];
        if (map != null)
        {
            var mapEntries = map.Entries();
            for (int j = 0; j < mapEntries.Length; j++)
            {
                keys[index] = mapEntries[j].Key;
                index++;
            }
        }
    }

    return keys;
}

public TValue[] Values()
{
    var values = new TValue[this.size];

    int index = 0;
    for (int i = 0; i < this.internalArray.Length; i++)
    {
        var map = this.internalArray[i];
        if (map != null)
        {
            var mapEntries = map.Entries();
            for (int j = 0; j < mapEntries.Length; j++)
            {
                values[index] = mapEntries[j].Value;
                index++;
            }
        }
    }

    return values;
}
```

The `Get(TKey key)` method is also similar, but instead of collecting a list of items, we simply walkthrough as before to find an item with a matching `key`. An `Exception` is thrown if not found.

```c#
public TValue Get(TKey key)
{
    for (int i = 0; i < this.size; i++)
    {
        var map = this.internalArray[i];
        if (map != null)
        {
            var entries = map.Entries();
            for (int j = 0; j < entries.Length; j++)
            {
                if (KeyEquals(entries[j].Key, key))
                {
                    return entries[j].Value;
                }
            }
        }
    }

    throw new Exception();
}
```

Notice the use of a custom `KeyEquals(TKey, TKey)` method that compares keys.

```c#
private static bool KeyEquals(TKey k1, TKey k2)
{
    return Comparer<TKey>.Default.Compare(k1, k2) == 0;
}
```

In C#, we can't directly compare generic variables using the `==` operator, so we use `Comparer<T>` to give us a valid comparer for the type of `TKey`. If we choose `TKey` to be a pretty standard type such as `string` or `int`, this method will suffice. If we are using some custom class as the `TKey`, you'll need to ensure that you provide proper comparison overrides in order for this to work.

In order to implement the `Set(TKey key, TValue value)` method, we'll need to define a *compression function* as outlined in the previous post. We use a rough implmentation of the **MAD (Multiply-Add-Divide)** method:

```c#
private int Compress(TKey key)
{
    long hashCode = Math.Abs(key.GetHashCode());

    int p = this.capacity;
    while(true)
    {
        if (isPrime(++p))
        {
            break;
        }
    }
    
    var random = new Random();
    int a = random.Next(1, p);
    int b = random.Next(0, p);

    return (int)(((a * hashCode + b) % p) % this.capacity);

    // functions
    bool isPrime(int number)
    {
        if (number == 1) return false;
        if (number == 2) return true;

        var boundary = (int)Math.Floor(Math.Sqrt(number));

        for (int i = 2; i <= boundary; ++i)
        {
            if (number % i == 0) return false;
        }

        return true;
    }
}
```

First, we hash using `GetHashCode()` and store its' absolute value as a `long`. This is required because we multiply the hash code by a random number, which could cause integer overflow. Note that this implementation is still brittle, because if the computed value of `a` is big enough (which may happen if `p` is very large), then we'd still overflow using a `long` as well. However, for our simple implementation this will suffice.

We then find a prime number as `p` that is larger than the capacity. The `a` and `b` values are drawn at random within the ranges [1, p - 1] and [0, p - 1] respectively. The returned int value is computed using the `MAD` formula described in the previous post. A local helper function is also implemented to help us find the prime numer for `p`.

The `Set(TKey key, TValue value)` method is then implemented as follows:

```c#
public void Set(TKey key, TValue value)
{
    int compressedSlot;

    // resize internal array if full
    if (this.size == this.capacity)
    {
        this.capacity *= 2;

        var resized = new UnsortedMap<TKey, TValue>[this.capacity * 2];
        Entry<TKey, TValue>[] entries = this.Entries();

        for (int i = 0; i < entries.Length; i++)
        {
            Entry<TKey, TValue> entry = entries[i];
            if (entry == null)
            {
                continue;
            }
            compressedSlot = this.Compress(entry.Key);
            if (resized[compressedSlot] == null)
            {
                resized[compressedSlot] = new UnsortedMap<TKey, TValue>();
            }
            resized[compressedSlot].Set(entry.Key, entry.Value);
        }

        this.internalArray = resized;
    }

    compressedSlot = this.Compress(key);
    if (this.internalArray[compressedSlot] == null)
    {
        this.internalArray[compressedSlot] = new UnsortedMap<TKey, TValue>();
    }

    this.internalArray[compressedSlot].Set(key, value);
    this.size++;
}
```

We first start by checking if the `internalArray` is full. If so, we need to dynamically resize it so we'll double it here. We grab the list of current entries, and then we re-add them into the larger array. It's important to note here that the keys must be re-compressed so we can get a nice spread across the range of the new internal array.

Another thing to keep in mind is that the internal array stores null values until *at least* one entry is present in the slot. At which, it stores a bucket. Thus, before adding an entry into a slot, we need to check for `null` and add in an `UnsortedMap` if it is before adding. We end by setting the new value and then incrementing the `size` counter.

`Remove(TKey key)` is fairly simple:

```c#
public bool Remove(TKey key)
{
    for (int i = 0; i < this.internalArray.Length; i++)
    {
        var map = this.internalArray[i];
        if (map != null)
        {
            bool removed = map.Remove(key);
            if (removed)
            {
                return true;
            }
        }
    }

    return false;
}
```

We walkthrough the internal array, and use the `UnsortedMap.Remove(TKey key)` method to try removing from existing buckets. Be sure to return the `bool` accordingly.

And that's it, we've implemented a hash map using separate chaining that handles collisions. In the next post, we'll explore a different collision handling scheme - linear probing.