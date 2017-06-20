---
layout: post
title: Introduction to Maps and an implementation of an Unsorted Map
---

A *map*, also referred to as an *associative array*, is one of the most useful and commonly used data structures around. The items stored in a map are *Entries*, which simply store a *key* and a *value* (or in the context of a C# *Dictionary*, a `KeyValuePair<TKey, TValue>`).

The *keys* are required to be unique, such that a given key will always *map* to the same value. Unlike a standard array, which maps an *integer index* to some value, a *map's* keys are not restricted to integers. You'll find common use-cases of the keys being `strings`, `Guids`, and other data types. Consequently, a good implementation of a map will be generic, giving the programmer flexibility for the types they'd like to use in the key-value mappings.

### Use-Cases and Examples
As mentioned, a *key* must be unique, so any real world scenario where something can be uniquely identified by some thing (a key) would benefit from a map data type if needed.

- A **social security number** maps the number uniquely to a specific person
- A **car's license plate number** maps the alphanumeric key to a specific vehicle
- A **username or handle** in a social network maps that account to a specific person
- A **birthdate** does *NOT* map that particular date to a specific person, since surely more than one person has a birthdate on that day.

The simplest rule for when a map data type should be used is if you find yourself iterating through some `array` or `list` multiple times looking for something. When you're searching an `array` for something based on a *key* (as opposed to its' *integer index*), the operation runs in `O(n)` time because the thing you're looking for may be at the very end and you have to walk through the entire `array` one at a time to find it.

<!--more-->

Thus, if you're making multiple passes through the array searching for things, this becomes `xO(n)` where `x` is the number of things you're searching for in the array. Here's an example in code:

```c#
List<User> users = await this.dbContext.GetUsersAsync();

// find user with email 'bob@test.com'
User bob = users.SingleOrDefault(u => u.Email == "bob@test.com");
if (bob != null) 
{
    // do something with bob
}

// later in code after doing some other things, look for 'mary@test.com'
User mary = users.SingleOrDefault(u => u.Email == "mary@test.com");
if (mary != null) 
{
    // do something with mary
}
```

A pretty simplified example but it should drive the point home. What if the list of users had a million items and bob and mary just happened to be the last two users on the list? Our code would be walking through that entire gigantic list twice, just to find the two users. In a real world use-case, we'd most likely be needing many more users from that list. Clearly a simple `array` or `list` is not the correct data type here.

Lets modify the code above using a `Dictionary` to make things more efficient:

```c#
List<User> users = await this.dbContext.GetUsersAsync();
Dictionary<string, User> userEmailMap = users.ToDictionary(u => u.Email, u => u);

// find 'bob@test.com' and do things
if (userEmailMap.TryGetValue("bob@test.com", out User bob)) 
{
    // do something with bob
}

// find 'mary@test.com' and do things
if (userEmailMap.TryGetValue("mary@test.com", out User mary)) 
{
    // do something with mary
}
```

Using a `Dictionary`, on the average case we're able to `get` a *value* by *key* in `O(1)` time (and in the worst case, `O(n)`). So if you ever need to iterate through an `array` more than once to do something, using a `map data type` makes a lot of sense.

As a side note, if we only ever needed to find *bob* from the above example, it wouldn't make sense to create a `Dictionary` from the users list, because the process of creating a `map` from an `array` is an `O(n)` operation anyways, so we're not getting any benefits from doing so.

## An Unsorted Map Implementation

Now that we have a general idea of what a `map` is and the purpose it serves, lets roll out a basic, albeit inefficient, implementation. This `UnsortedMap` will simply store the `Entries` in an array, resizing it as needed.

Any real `map` implementation will be using a better implementation that gives us faster run times for its' core operations, but this will serve as a good starting place to get our hands wet with maps. Also, a map, by definition, really only needs to create a mapping from some *key* to a *value* which this `UnsortedMap` will implement. We'll explore some more efficient implementations that get us the `O(1)` running times in later posts.

We'll start with defining an `Entry`, which is simply a wrapper around a *key* and *value* pairing:

```c#
public class Entry<TKey, TValue>
{
    public TKey Key { get; set; }
    public TValue Value { get; set; }
}
```

Next, lets define an `Interface` for our maps:

```c#
public interface IMap<TKey, TValue>
{
    int Size();
    bool IsEmpty();
    TValue Get(TKey key);
    void Set(TKey key, TValue value);
    bool Remove(TKey key);
    IEnumerable<TKey> Keys();
    IEnumerable<TValue> Values();
    IEnumerable<Entry<TKey, TValue>> Entries();
}
```

Lets discuss some details of the interface above:

#### int Size()
This will simply return the current number of `Entries` stored in our map. We'd like this to run in `O(1)`, so this will need to be updated as follows:
- When setting, increment by 1 if an `Entry` with the key doesn't already exist. If it already exists, we simply update the value of the existing `Entry` without changing the internal size counter.
- When removing, decrement by 1 if the entry to be removed by key was found.

When I say that we'd like this method to run in `O(1)` time, what I mean is that we need to always be keeping track of this size, or in other words, the number of `Entries` the map currently has. If we don't keep track of this, we would have to walk through the `internal array` every time this method is called to get a count of `Entries` it has. It's far more efficient to update this counter as we go, such that we can immediately return the `size` right when it's requested.

#### bool IsEmpty()
It's pretty standard to include helper methods like this in custom data structures as it helps with the readability of your code. Internally, all it really does is:

`return this.Size() == 0;`

#### TValue Get(TKey key)
This method will search for an `Entry` using the given key, returning its' *value* if found, and throwing an `Exception` if not found.

#### void Set(TKey key, TValue value)
This method will add the new `Entry` with the given key if it doesn't yet exist, or update if it does. If we're adding, we must ensure the following:
- The internal size counter is incremented
- If adding causes more `Entries` to be stored than the internal array's capacity can handle, we'll first create a new internal array that's double in size, ensuring to properly copy over existing `Entries` before adding to avoid any *index out of bounds* errors. This operation is similar to how *dynamic arrays* handle resizing. The user of these data structures should never have to worry about resizing operations, these should be handled behind the scenes by our implementations

#### bool Remove(TKey key)
This method attempts to remove an `Entry` with the given *key* if found. It also returns a `bool` indicating whether the `Entry` was found and subsequently removed. If an `Entry` was removed, we also need to decrement the internal size counter.

#### IEnumerable<TKey> Keys()
Returns an enumerable list of *keys* for the `Entries` in the map.

#### IEnumerable<TValue> Values()
Returns an enumerable list of *values* for the `Entries` in the map.

#### IEnumerable<Entry<TKey, TValue>> Entries()
Returns an enumerable list of *entries* that the map contains.

Lets dive into the implementation now. We'll start with the constructors and some the internal fields:

```c#
public class UnsortedMap<TKey, TValue> : IMap<TKey, TValue>
{
    private Entry<TKey, TValue>[] internalArray { get; set; }

    private int size { get; set; } = 0;
    private int capacity { get; set; }

    public UnsortedMap(int capacity)
    {
        this.capacity = capacity;
        this.internalArray = new Entry<TKey, TValue>[capacity];
    }

    // default constructor with capacity of 10 entries
    public UnsortedMap() : this(10)
    {
    }
}
```

The constructor sets the internal `capacity`, which represents the `length` of the `internalArray`. It also initializes the `internalArray` using said `capacity`.

The map is also initialized with a `size` counter starting at 0, which is only ever changed when an `Entry` is removed or added from its `internalArray`.

Here's the first couple of methods:

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

Because we're keeping an internal counter on the map's size, we're able to immediately return it when `Size()` is called. Calling `IsEmpty()` does a simple comparison to see if it's zero. Both methods run in `O(1)` time.

Before implementing some of the other methods, we need to define this utility method that helps us compare keys:

```c#
private static bool KeyEquals(TKey k1, TKey k2)
{
    return Comparer<TKey>.Default.Compare(k1, k2) == 0;
}
```

How you compare two generic things may differ depending on the language you're using. Because our examples are in C#, and we cannot make comparisons such as `k1 == k2` in the example above, we create a small helper that wraps the logic in making the comparison.

We grab the default comparer for the type `TKey` and determine if they are equal by seeing if running `Compare(x, y)` returns `0`. This way, we don't have to remember how to do this comparison where we need it. Also, it looks much cleaner to be able to use `KeyEquals(k1, k2)` as opposed to the line of code it encapsulates.

Lets jump to the `Get(TKey key)` method. Given a key, we want to check our map to see if a matching `Entry` exists. If found, return that `Entry`'s value. If not, we throw an `Exception`.

```c#
public TValue Get(TKey key)
{
    for (int i = 0; i < this.size; i++)
    {
        if (KeyEquals(this.internalArray[i].Key, key))
        {
            return this.internalArray[i].Value;
        }
    }
    throw new Exception($"Entry with key '{key}' not found.");
}
```

We only have to enumerate from the range `[0, size - 1]` inclusive (all ranges shown in this manner will represent an *inclusive* range from this point on). We only need to check the indices of the `internalArray` where `Entries` exist, and we'll be ensuring that there aren't any *gaps* when an `Entry` is removed. This will help us keep the map more performant by only having to walkthrough `size` number of steps to find a matching `Entry`, in the worst case.

We use the `KeyEquals()` method specified earlier to run a quick comparison on each step. After we've checked all existing `Entries`, if no matching one has been found, we throw an `Exception`.

For the `Set()` method, we'll first start by walking through the existing `Entries` in `internalArray` to see if we're going to be updating an `Entry`:

```c#
public void Set(TKey key, TValue value)
{
    // check to see if Entry with key exists and update if so
    for (int i = 0; i < this.size; i++)
    {
        if (KeyEquals(this.internalArray[i].Key, key))
        {
            this.internalArray[i].Value = value;
            return;
        }
    }

    // ...
}
```

Remember that the internal `size` counter is not modified during an update, as we're neither adding nor removing an `Entry`.

If this is *not* an update operation, we know that we're *adding*. However, before adding, we need to first ensure that the `internalArray` is large enough, so we start by comparing the current `size` with the current `capacity`. If adding another `Entry` isn't possible due to the `capacity` constraint, we'll first run some copies and reset the `internalArray` to be double its' current size:

```c#
// check if internalArray is full and resize if needed
if (this.size == this.capacity)
{
    var clone = new Entry<TKey, TValue>[this.capacity * 2];
    for (int i = 0; i < this.capacity; i++)
    {
        clone[i] = this.internalArray[i];
    }
    this.internalArray = clone;
    this.capacity *= 2;
}
```

Afterwards, it's safe to proceed with the *add* operation:

```c#
this.internalArray[this.size] = new Entry<TKey, TValue>
{
    Key = key,
    Value = value
};
this.size++;
```

Don't forget to increment the `size` counter. We also use the `size` counter as the *index* in which we place the new `Entry`.

Removing an `Entry` is fairly straight-forward. We walk through the `internalArray`, and if we find an `Entry` with a matching *key*, we remove it. The one thing we want to ensure, however, is that if there are any `Entries` left after the removal, we want to move the endmost `Entry` into the *index* of the `Entry` we're removing. Because this map implementation is unsorted, it doesn't matter that we move `Entries` around as the ordering is already completely arbitrary.

```c#
public bool Remove(TKey key)
{
    for (int i = 0; i < this.size; i++)
    {
        if (KeyEquals(this.internalArray[i].Key, key))
        {
            if (this.size >= 2)
            {
                this.internalArray[i] = this.internalArray[this.size - 1];
                this.internalArray[this.size - 1] = null;
            }
            else
            {
                this.internalArray[i] = null;
            }
            this.size--;
            return true;
        }
    }
    return false;
}
```

Ensuring that the `internalArray` doesn't have any gaps makes some other operations easier to implement. For example, in the `Set()` method we implemented earlier, we can simply add a new `Entry` into the index `this.size` because that represents the next available spot in the `internalArray`, *but* if and *only* if it doesn't contain any gaps.

We of course decrement the `size` counter if an `Entry` was found to be removed, and also return a `bool` as well.

The remaining methods of the interface are all very similar. We simply iterate through `internalArray` and return a list of *keys*, *values*, or *entries*.

```c#
public IEnumerable<TKey> Keys()
{
    var keys = new TKey[this.size];
    for (int i = 0; i < this.size; i++)
    {
        keys[i] = this.internalArray[i].Key;
    }
    return keys;
}

public IEnumerable<TValue> Values()
{
    var values = new TValue[this.size];
    for (int i = 0; i < this.size; i++)
    {
        values[i] = this.internalArray[i].Value;
    }
    return values;
}

public IEnumerable<Entry<TKey, TValue>> Entries()
{
    var entries = new Entry<TKey, TValue>[this.size];
    for (int i = 0; i < this.size; i++)
    {
        entries[i] = this.internalArray[i];
    }
    return entries;
}
```

We create a new array with its `size` being equal to the map's current size. Next, we walkthrough the `internalArray`, adding the items to the new array, then return the result at the end.

Here's our `UnsortedMap<TKey, TValue>` in its' entirety:

```c#
public class UnsortedMap<TKey, TValue> : IMap<TKey, TValue>
{
    private Entry<TKey, TValue>[] internalArray { get; set; }

    private int size { get; set; }
    private int capacity { get; set; }

    public UnsortedMap(int capacity)
    {
        this.capacity = capacity;
        this.internalArray = new Entry<TKey, TValue>[capacity];
    }

    // default constructor with capacity of 10 entries
    public UnsortedMap() : this(10)
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
        for (int i = 0; i < this.size; i++)
        {
            if (KeyEquals(this.internalArray[i].Key, key))
            {
                return this.internalArray[i].Value;
            }
        }
        throw new Exception($"Entry with key '{key}' not found.");
    }

    public void Set(TKey key, TValue value)
    {
        // check to see if Entry with key exists and update if so
        for (int i = 0; i < this.size; i++)
        {
            if (KeyEquals(this.internalArray[i].Key, key))
            {
                this.internalArray[i].Value = value;
                return;
            }
        }

        // check if internalArray is full and resize if needed
        if (this.size == this.capacity)
        {
            var clone = new Entry<TKey, TValue>[this.capacity * 2];
            for (int i = 0; i < this.capacity; i++)
            {
                clone[i] = this.internalArray[i];
            }
            this.internalArray = clone;
            this.capacity *= 2;
        }

        // at this point it's safe to add
        this.internalArray[size] = new Entry<TKey, TValue>
        {
            Key = key,
            Value = value
        };
        this.size++;
    }

    public bool Remove(TKey key)
    {
        for (int i = 0; i < this.size; i++)
        {
            if (KeyEquals(this.internalArray[i].Key, key))
            {
                // if any Entries will exist after removal, swap the very last
                // one into the index of the removed Entry
                if (this.size >= 2)
                {
                    this.internalArray[i] = this.internalArray[this.size - 1];
                    this.internalArray[this.size - 1] = null;
                }
                else
                {
                    this.internalArray[i] = null;
                }
                this.size--;
                return true;
            }
        }
        return false;
    }

    public IEnumerable<TKey> Keys()
    {
        var keys = new TKey[this.size];
        for (int i = 0; i < this.size; i++)
        {
            keys[i] = this.internalArray[i].Key;
        }
        return keys;
    }

    public IEnumerable<TValue> Values()
    {
        var values = new TValue[this.size];
        for (int i = 0; i < this.size; i++)
        {
            values[i] = this.internalArray[i].Value;
        }
        return values;
    }

    public IEnumerable<Entry<TKey, TValue>> Entries()
    {
        var entries = new Entry<TKey, TValue>[this.size];
        for (int i = 0; i < this.size; i++)
        {
            entries[i] = this.internalArray[i];
        }
        return entries;
    }

    private static bool KeyEquals(TKey k1, TKey k2)
    {
        return Comparer<TKey>.Default.Compare(k1, k2) == 0;
    }
}
```

Sweet! We've implemented our first `map` data structure, but this is probably in the discussion for one of the worst implementations you can make. We don't get the fast `O(1)` reads that well-implemented maps are known for. In C#, we could probably get away with just using a `List<Entry<TKey, TValue>>` in most cases instead of using an unsorted map, as the reads would be comparable.

Here's a table illustrating the run times for the methods our `IMap` interface specifies:

{: .table .table-bordered .table-striped .grid-4}
Method | Runtime
--- | ---
Size | O(1)
IsEmpty | O(1)
Get | O(n)
Set | O(n)
Remove | O(n)
Keys | O(n)
Values | O(n)
Entries | O(n)

As you can see, almost all of the core operations defined by the `interface` run in `linear` time, which is pretty bad. An ideal implementation of a `map` should run most of the core operations in constant `O(1)` time.

The next couple of posts will walk through creating a `HashMap` (or HashTable, HashSet, etc). The `HashMap` is another map data structure, so it'll implement the same interface the `UnsortedMap` did. However, it can employ some clever and trickier strategies to help us achieve the desired `O(1)` run times for most of the core operations.