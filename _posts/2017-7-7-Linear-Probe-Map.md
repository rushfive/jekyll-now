---
layout: post
title: Linear Probe Map (Open Addressing Scheme)
---

**Open Addressing** is another method to handle collisions. One disadvantage of using *separate chaining* is that it requires the use of an auxiliary data structure, which costs more space. Using *open addressing*, we store entries directly into a slot. This requires the load factor is always 1 or less, as we can't store more entries than the number of slots that exist in the internal array.

The *open addressing* scheme is a general term that describes a collision handling approach that stores entries directly into the internal array. The simplest method is **linear probing**, which works as follows:

**Setting an entry**
- a *hash code* is obtained, then *compressed* as usual
- the entry is attempted to be stored at the compressed index 
- if the slot is available (null), store it there
- if it's taken, we look at the next slot. If available, store it. If not, go to the next. Rinse and repeat.

**Getting an entry**
- get the compressed index of the key
- walk through the internal array starting at the compressed index. If you find an entry with the key, it exists so return it. If you run into an empty slot first, it doesn't exist.

<!--more-->

**Removing an entry**
This is pretty similar to *getting* an entry but there's a gotcha that we have to keep in mind.
- get the compressed index of the key
- walk through the internal array starting at the compressed index. If you find an entry with the key, we **cannot** simply remove it and set it to null. That would break the other operations that need to walkthrough the internal array, causing it to hit a null slot before reaching the entry it needs to.
- for this, we insert a sentinel entry that marks it as *defunct*. A *defunct* entry marks the slot so we know to continue walking through if we stumble upon one while searching for an entry.

The use of a *defunct* entry also affects how we set a new entry. While walking through the internal array, we need to note the index of the first *defunct* entry found. After we're done walking through, if an entry with the key does not exist and there's no empty slot to place it into, we can place the new entry into the index of the first *defunct* entry.

### Load Factor
While the recommended load factor for *separate chaining* schemes was 0.9 or less, for *open addressing* schemes it sits at around 0.5! Studies have shown that as the load factor increases beyond 0.5 and reaches closer to 1, the chances of clusters appearing greatly increase (a cluster being entries placed in consecutive slots). Clusters cause similar issues to a separately chained bucket containing more than one entry. It requires walking through something in order to find an entry, which slows the run times of those operations.

And of course, the load factor can never be more than 1 for *open addressing* because we're throwing in an entry per slot!

### Implementing a Linear Probe Hash Map

Our map must implement the `IMap<TKey, TValue>` interface:

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

We'll start with the private members:

```c#
private Entry<TKey, TValue>[] internalArray { get; set; }
private int size { get; set; } = 0;
private int defunctCount { get; set; } = 0;
private int capacity { get; set; }
```

These should seem familiar, they're the same ones as with the previous map implementations. The one new member is `defunctCount`, which is a counter storing the number of defunct entries that are currently stored. We need to dynamically resize the internal array when the load factor hits ~0.5. The number of entries to determine the load factor value should be the size plus the defunct count, since a defunct entry takes up a slot in the array.

`Size()` and `IsEmpty()` is simple enough, making use of the internal members:

```c#
public int Size()
{
    return this.size;
}

public bool IsEmpty()
{
    return this.size == 0;
}
```

To continue with the other methods, we need to first define a defunct entry. Both the Entry and DefunctEntry is below for reference:

```c#
public class Entry<TKey, TValue>
{
    public TKey Key { get; set; }
    public TValue Value { get; set; }
}

// used as a sentinel placeholder for linear probe collision handling
public class DefunctEntry<TKey, TValue> : Entry<TKey, TValue>
{
}
```

The `DefunctEntry` simply derives from `Entry`. When we walk through and need to check for it, we can simply use C#'s `is` operator.

`Keys(), Values(), and Entries()` are all very simlar:

```c#
public TKey[] Keys()
{
    var keys = new TKey[this.size];

    int index = 0;
    for (int i = 0; i < this.internalArray.Length; i++)
    {
        if (this.internalArray[i] != null
            &&  !(this.internalArray[i] is DefunctEntry<TKey, TValue>))
        {
            keys[index] = this.internalArray[i].Key;
            index++;
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
        if (this.internalArray[i] != null
            && !(this.internalArray[i] is DefunctEntry<TKey, TValue>))
        {
            values[index] = this.internalArray[i].Value;
            index++;
        }
    }

    return values;
}

public Entry<TKey, TValue>[] Entries()
{
    var entries = new Entry<TKey, TValue>[this.size + this.defunctCount];

    int index = 0;
    for (int i = 0; i < this.internalArray.Length; i++)
    {
        if (this.internalArray[i] != null
            && !(this.internalArray[i] is DefunctEntry<TKey, TValue>))
        {
            entries[index] = this.internalArray[i];
            index++;
        }
    }

    return entries;
}
```

They walk through the internal array, and collect the relevant information so long as the slot is *not* null and is *not* a defunct entry.

The other operations is where things get a little more interesting. The next operations will make use of the same helper methods `KeyEquals` and `Compress` that were implemented in my previous
[Chain Hash Map post]({{ site.baseurl }}{% post_url 2017-7-6-Chain-Hash-Map %}).

Lets start with the **Get(TKey key)** method:

```c#
public TValue Get(TKey key)
{
    int startingIndex = this.Compress(key);

    // check range [compress(key), N - 1]
    for (int i = startingIndex; i < this.internalArray.Length; i++)
    {
        Entry<TKey, TValue> entry = this.internalArray[i];

        if (entry == null)
        {
            throw new Exception();
        }
        if (entry is DefunctEntry<TKey, TValue> || !KeyEquals(entry.Key, key))
        {
            continue;
        }

        return entry.Value;
    }

    // check range [0, compress(key) - 1]
    for (int i = 0; i < startingIndex; i++)
    {
        Entry<TKey, TValue> entry = this.internalArray[i];

        if (entry == null)
        {
            throw new Exception();
        }
        if (entry is DefunctEntry<TKey, TValue> || !KeyEquals(entry.Key, key))
        {
            continue;
        }

        return entry.Value;
    }

    throw new Exception();
}
```

We compress the key, then use that as a starting point to walk through the internal array. Note that a for loop is repeated, the first to cover the range [compressedIndex, N - 1], then another following immediately after that covers [0, compressedIndex - 1]. You could do this in a single loop using something like a do-while loop with modulus but we'll just roll with this.

If we ever run into a `null` value, it means the entry with the given key doesn't exist so we throw an exception. If an entry *does* exist but it's either defunct or has a mismatching key we continue walking through. Lastly, if an entry with a matching key is found, return the value.

We also throw an exception if a matching entry was not found after going through both for loops. This means that the entire internal array is full and does not contain a matching entry.

**Set(TKey key, TValue value)** has a lot more going on:

```c#
public void Set(TKey key, TValue value)
{
    // resize array, try to keep load factor roughly < 0.5
    if (this.size + this.defunctCount >= this.capacity / 2)
    {
        var resized = new Entry<TKey, TValue>[this.capacity * 2];
        this.defunctCount = 0;

        Entry<TKey, TValue>[] entries = this.Entries();
        for (int i = 0; i < entries.Length; i++)
        {
            setFirstAvailableSlot(entries[i].Key, entries[i].Value, resized);
        }

        this.internalArray = resized;
    }

    setFirstAvailableSlot(key, value, this.internalArray, true);

    // functions
    void setFirstAvailableSlot(TKey k, TValue v, Entry<TKey, TValue>[] array, 
        bool incrementSize = false)
    {
        int compressedSlot = this.Compress(k);
        int? firstAvailable = null;

        // range [compressedSlot, N - 1]
        for (int i = compressedSlot; i < array.Length; i++)
        {
            if (array[i] == null)
            {
                array[i] = new Entry<TKey, TValue>
                {
                    Key = k,
                    Value = v
                };
                if (incrementSize)
                {
                    this.size++;
                }
                return;
            }

            if (array[i] is DefunctEntry<TKey, TValue>)
            {
                if (!firstAvailable.HasValue)
                {
                    firstAvailable = compressedSlot;
                }
                continue;
            }

            // has entry
            if (KeyEquals(array[i].Key, k))
            {
                array[i].Value = v;
                return;
            }
            // has entry, different entry, continue
        }

        // range [0, compressedSlot - 1]
        for (int i = 0; i < compressedSlot - 1; i++)
        {
            if (array[i] == null)
            {
                array[i] = new Entry<TKey, TValue>
                {
                    Key = k,
                    Value = v
                };
                if (incrementSize)
                {
                    this.size++;
                }
                return;
            }

            if (array[i] is DefunctEntry<TKey, TValue>)
            {
                if (!firstAvailable.HasValue)
                {
                    firstAvailable = compressedSlot;
                }
                continue;
            }

            // has entry
            if (KeyEquals(array[i].Key, k))
            {
                array[i].Value = v;
                return;
            }
            // has entry, different entry, continue
        }

        // not yet set, set in first available defunct spot
        if (firstAvailable.HasValue)
        {
            array[firstAvailable.Value] = new Entry<TKey, TValue>
            {
                Key = k,
                Value = v
            };
            if (incrementSize)
            {
                this.size++;
            }
        }
        else
        {
            // ensure this doesnt reach here
            throw new Exception();
        }
    }
}
```

The first thing we need to look at is the local helper function:
`void setFirstAvailableSlot(TKey k, TValue v, Entry<TKey, TValue>[] array, bool incrementSize = false)`

This takes a key, value, and an array, and proceeds to walk through using the familiar two for-loop method. It keeps track of the first *DefunctEntry* index found so we can place the new entry there if an entry with the key doesn't already exist. The last optional argument `incrementSize` is used to determine whether the internal `size` counter should be incremented when a new entry is added.

This is required because the `setFirstAvailableSlot` method is also used when resizing the internal array and re-setting the entries in the new array. In this scenario, the size should never be incremented because we're simply spreading them out again in a new array, not adding new ones.

The helper method takes up most of the method. The actual code execution of the method starts with a check and resizes the internal array if the load factor hits 0.5. Lastly, it adds the new entry using the helper method.

The **Remove(TKey key)** method:

```c#
public bool Remove(TKey key)
{
    int compressedSlot = this.Compress(key);

    for (int i = compressedSlot; i < this.internalArray.Length; i++)
    {
        var slotEntry = this.internalArray[i];

        if (slotEntry == null)
        {
            return false;
        }

        if (slotEntry is DefunctEntry<TKey, TValue>)
        {
            continue;
        }

        // has entry
        if (KeyEquals(slotEntry.Key, key))
        {
            this.internalArray[i] = new DefunctEntry<TKey, TValue>();
            this.size--;
            this.defunctCount++;
            return true;
        }
        // has entry but different key, continue
    }

    for (int i = 0; i < compressedSlot - 1; i++)
    {
        var slotEntry = this.internalArray[i];

        if (slotEntry == null)
        {
            return false;
        }

        if (slotEntry is DefunctEntry<TKey, TValue>)
        {
            continue;
        }

        // has entry
        if (KeyEquals(slotEntry.Key, key))
        {
            this.internalArray[i] = new DefunctEntry<TKey, TValue>();
            this.size--;
            this.defunctCount++;
            return true;
        }
        // has entry but different key, continue
    }

    return false;
}
```

We use the same two for-loops approach to potentially check the entire range of the internal array. If we find a matching entry we remove it, decrement the `size` counter, and throw in a `DefunctEntry` object into the slot. Return true or false accordingly.

Here's the entire implemented class:

```c#
public class LinearProbeHashMap<TKey, TValue> : IMap<TKey, TValue>
{
    private Entry<TKey, TValue>[] internalArray { get; set; }

    private int size { get; set; } = 0;
    private int defunctCount { get; set; } = 0;
    private int capacity { get; set; }


    public LinearProbeHashMap(int capacity)
    {
        this.capacity = capacity;
        this.internalArray = new Entry<TKey, TValue>[capacity];
    }

    public LinearProbeHashMap() : this(5)
    {
    }

    public int Size()
    {
        return this.size;
    }

    public bool IsEmpty()
    {
        return this.size == 0;
    }

    public TValue Get(TKey key)
    {
        int startingIndex = this.Compress(key);

        // check range [compress(key), N]
        for (int i = startingIndex; i < this.internalArray.Length; i++)
        {
            Entry<TKey, TValue> entry = this.internalArray[i];

            if (entry == null)
            {
                throw new Exception();
            }
            if (entry is DefunctEntry<TKey, TValue> || !KeyEquals(entry.Key, key))
            {
                continue;
            }

            return entry.Value;
        }

        // check range [0, compress(key)]
        for (int i = 0; i < startingIndex; i++)
        {
            Entry<TKey, TValue> entry = this.internalArray[i];

            if (entry == null)
            {
                throw new Exception();
            }
            if (entry is DefunctEntry<TKey, TValue> || !KeyEquals(entry.Key, key))
            {
                continue;
            }

            return entry.Value;
        }

        // should never reach here, would throw before this point while iterating
        // the internal array
        throw new Exception();
    }

    public void Set(TKey key, TValue value)
    {
        // resize array, try to keep load factor roughly < 0.5
        if (this.size + this.defunctCount >= this.capacity / 2)
        {
            var resized = new Entry<TKey, TValue>[this.capacity * 2];
            this.defunctCount = 0;

            Entry<TKey, TValue>[] entries = this.Entries();
            for (int i = 0; i < entries.Length; i++)
            {
                setFirstAvailableSlot(entries[i].Key, entries[i].Value, resized);
            }

            this.internalArray = resized;
        }

        setFirstAvailableSlot(key, value, this.internalArray, true);

        // functions
        void setFirstAvailableSlot(TKey k, TValue v, Entry<TKey, TValue>[] array, 
            bool incrementSize = false)
        {
            int compressedSlot = this.Compress(k);
            int? firstAvailable = null;

            // range [compressedSlot, N - 1]
            for (int i = compressedSlot; i < array.Length; i++)
            {
                if (array[i] == null)
                {
                    array[i] = new Entry<TKey, TValue>
                    {
                        Key = k,
                        Value = v
                    };
                    if (incrementSize)
                    {
                        this.size++;
                    }
                    return;
                }

                if (array[i] is DefunctEntry<TKey, TValue>)
                {
                    if (!firstAvailable.HasValue)
                    {
                        firstAvailable = compressedSlot;
                    }
                    continue;
                }

                // has entry
                if (KeyEquals(array[i].Key, k))
                {
                    array[i].Value = v;
                    return;
                }
                // has entry, different entry, continue
            }

            // range [0, compressedSlot - 1]
            for (int i = 0; i < compressedSlot - 1; i++)
            {
                if (array[i] == null)
                {
                    array[i] = new Entry<TKey, TValue>
                    {
                        Key = k,
                        Value = v
                    };
                    if (incrementSize)
                    {
                        this.size++;
                    }
                    return;
                }

                if (array[i] is DefunctEntry<TKey, TValue>)
                {
                    if (!firstAvailable.HasValue)
                    {
                        firstAvailable = compressedSlot;
                    }
                    continue;
                }

                // has entry
                if (KeyEquals(array[i].Key, k))
                {
                    array[i].Value = v;
                    return;
                }
                // has entry, different entry, continue
            }

            // not yet set, set in first available defunct spot
            if (firstAvailable.HasValue)
            {
                array[firstAvailable.Value] = new Entry<TKey, TValue>
                {
                    Key = k,
                    Value = v
                };
                if (incrementSize)
                {
                    this.size++;
                }
            }
            else
            {
                // ensure this doesnt reach here
                throw new Exception();
            }
        }
    }

    public bool Remove(TKey key)
    {
        int compressedSlot = this.Compress(key);

        for (int i = compressedSlot; i < this.internalArray.Length; i++)
        {
            var slotEntry = this.internalArray[i];

            if (slotEntry == null)
            {
                return false;
            }

            if (slotEntry is DefunctEntry<TKey, TValue>)
            {
                continue;
            }

            // has entry
            if (KeyEquals(slotEntry.Key, key))
            {
                this.internalArray[i] = new DefunctEntry<TKey, TValue>();
                this.size--;
                this.defunctCount++;
                return true;
            }
            // has entry but different key, continue
        }

        for (int i = 0; i < compressedSlot - 1; i++)
        {
            var slotEntry = this.internalArray[i];

            if (slotEntry == null)
            {
                return false;
            }

            if (slotEntry is DefunctEntry<TKey, TValue>)
            {
                continue;
            }

            // has entry
            if (KeyEquals(slotEntry.Key, key))
            {
                this.internalArray[i] = new DefunctEntry<TKey, TValue>();
                this.size--;
                this.defunctCount++;
                return true;
            }
            // has entry but different key, continue
        }

        return false;
    }

    public TKey[] Keys()
    {
        var keys = new TKey[this.size];

        int index = 0;
        for (int i = 0; i < this.internalArray.Length; i++)
        {
            if (this.internalArray[i] != null
                &&  !(this.internalArray[i] is DefunctEntry<TKey, TValue>))
            {
                keys[index] = this.internalArray[i].Key;
                index++;
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
            if (this.internalArray[i] != null
                && !(this.internalArray[i] is DefunctEntry<TKey, TValue>))
            {
                values[index] = this.internalArray[i].Value;
                index++;
            }
        }

        return values;
    }

    public Entry<TKey, TValue>[] Entries()
    {
        var entries = new Entry<TKey, TValue>[this.size + this.defunctCount];

        int index = 0;
        for (int i = 0; i < this.internalArray.Length; i++)
        {
            if (this.internalArray[i] != null
                && !(this.internalArray[i] is DefunctEntry<TKey, TValue>))
            {
                entries[index] = this.internalArray[i];
                index++;
            }
        }

        return entries;
    }

    private static bool KeyEquals(TKey k1, TKey k2)
    {
        return Comparer<TKey>.Default.Compare(k1, k2) == 0;
    }

    // use long to prevent integer overflow
    private int Compress(TKey key)
    {
        long hashCode = Math.Abs(key.GetHashCode());

        int p = this.capacity;
        while (true)
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
}
```

If you'd like to mess around and visually see how the operations work, you can clone my [repository](https://github.com/rushfive/DataStructuresAlgorithms). I've built an interactive console tester that allows you to test all the different maps we've implemented thus far and visualize the changes.