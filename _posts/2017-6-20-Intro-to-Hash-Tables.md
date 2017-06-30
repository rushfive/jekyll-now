---
layout: post
title: Hash Tables, Hash Codes and Compression Functions
---

**Hash tables** are an implementation of a map data structure, and they happen to be one of the most efficient around. Whereas the **unsorted map** I implemented previously logs mostly *linear* run times for most of a map's core operations, **hash tables** are able to reliably achieve *constant* run times by cleverly making use of the simple, yet fundamental, `array` data structure.

An array is always able to *get* an element in constant time if you know the index. This is because you'll already have a reference to the array in memory, and so the index is used as an offset from the head to grab the value in a snap. So, in order to use an array to implement a **hash table**, we need to ensure that

1. the *key* we use is an integer
2. and that it fits within the bounds of the array

However, a map is generally implemented *generically*, such that any `data type` can be used as a key. This causes an issue with *point 1* above if we need a map with a non-integer key. 

Well, lets say we are using integer keys. We still have an issue of utilizing an array, because we could potentially require a *huge* array to hold only a small number of entries. If the integer keys we need to store are 0 and 1000, we would need to allocate enough memory to hold an array of size 1001, just to store two entries. What a waste!

## Hash Codes and the Compression Function

In order to tackle the two issues above, **hash tables** require the use of a **hash function**. This function itself is split into two components that tackle the problems:

1. obtaining a **hash code**, and
2. applying a **compression function**

The **hash code** is obtained by computing some logic on the key. For example, if we have a string key in .NET, we can get a useable hash code using `GetHashCode()`, which is overriden to ensure that the same code is created for a string with the same sequence of characters. If you create some custom data structure, you'll want to override `GetHashCode()` and `Equals()` so that it conforms to the needs of a hash table. 

A general guideline for an `Equals()` override is that two objects that should be considered logically equal should always compute the same hash code. In other words, if `a.Equals(b) == true`, then `a.GetHashCode() == b.GetHashCode()` should also be true.

If you have two random objects, `a` and `b`, it's possible that our hash code function ends up computing the same hash code for them. Ultimately, they'll end up being placed in the same spot (index) of the array. This is known as a **collision**, and it's something that we must try to minimize in our hash table implementations.

I'll get into the methods to resolve collisions shortly. For now, it's enough to know that multiple entries can be stored in an array because we store an auxiliary data structure such as a linked list as the array values.

If we end up having to search through multiple entries stored in an array's index to see if it contains a given key, that's where `Equals()` comes in. Lets assume that the string keys `"hello"` and `"other"` produced the same hash code so they were thrown into the same spot in the array (we're ommitting the compression function until a bit later). To determine whether the key `"hello"` exists in this spot, we'll go through the keys and run `Equals()`. At some point, we end up running `"hello".Equals("hello")` which returns true, so we know that an entry with that key exists in this spot. Calling string's `Equals()` returns true in that case because that's how the C# team implemented the override. The strings "hello" and "hello" are logically the same, so they should be considered equal. Makes sense.

To recap, two objects may compute the same hash code but not be considered equal. However, two objects that *are* considered equal should *always* have the same hash code.

If you have your own custom data structure, it's up to you to implement efficient and bug-free overrides of `GetHashCode()` and `Equals()`. For common value types such as `string` or `int` and others, they already come with those methods overriden and are immediately useable in a hash table. Below, we'll explore some common methods employed to obtain hash codes from these kinds of value data types.

Next we'll explore how hashes of some common data types are computed.

**Bytes and Shorts**
`byte`s are represented as an 8-bit unsigned integer, while `short`s are represented as a 16-bit signed integer. The objective is to map the byte/short to an `int` (hash code), which is represented as a 32-bit signed integer. They all represent essentially a different range of integers, with the `int` type having the space to encompass both `byte` and `short` values.

For example, the value `7` in a byte is represented in bits as 

`00000111`

while the same number `7` is represented in an `int` as 

`000000000000000000000000000111`

We can simply *cast* the `byte` and `short` to an `int` and we'd end up with the exact same thing:

`(int) 00000111 --> 000000000000000000000000000111`

It essentially adds more padding (think space) while keeping the same value. This is perfect for passing to the compression function because it represents the same value and is of the data type required by the function (int).

**Longs and Doubles**
Some other data types such as `long` and `double`s can hold bigger numbers. These types take 64 bits of space, so we can't simply cast to an `int`. Doing so would not only chop off 32 of the bits, but would also cause bugs in where two different `long` values are now casted to the same `int` (the lower 32 bits happened to be the same).

In these scenarios, we can split the 64 bit representation into two 32 bit components and **sum** them up. Here's a C# example:

```c#
long bigValue = 12345678901234;
int lowerPart = (int)bigValue;
int higherPart = (int)(bigValue >> 32);
int summedHashCode = lowerPart + higherPart;
```

**Strings and Polynomial hash codes**
Hashing `string`s is a little tricker. A string is just a sequence of characters, each of which is represented by an integer (for example, ASCII codes). You might think the solution would be to take the sum of all the character's integer representations. However, that leads to issues where two distinct and different strings end up computing the same hash code.

For example, the characters in the string `"hash"` is represented by the numbers 104, 97, 115, and 104 in that order. Taking the sum gives us 420. But using this method, the strings "`ashh`" and "`shha`" also compute to 420, when those strings are clearly different and should *NOT* be considered equal to `"hash"`. To correctly hash a string, we need to account for the specific order the characters appear in the sequence. A common function used for this is:

 (x<sub>k-1</sub> * a<sup>k-1</sup>) + (x<sub>k-2</sub> * a<sup>k-2</sup>) + ... +  (x<sub>0</sub> * a<sup>0</sup>)

 Where `k` is the number of characters in the string, `x` is the integer representation of the character, and `a` is a random prime number. Studies and tons of testing have shown that a prime number is necessary for the value of `a` to get a good spread of the hash values, but in particular, 33, 37, 39, and 41 have been found to work really well.

------------------

Once a hash code is obtained, we then apply a **compression function** to the code. The objective of a good compression function is not only to fit a hash code within the given range of an array, but also to minimize the number of collisions.

Below are two common implementations of a compression function.

### The Division Method

This function is defined as

`i mod N`

where `i` is the integer (hash code) and `N` is the size of the array. It's highly recommended to choose a prime number as the size of the array as this minimizes the number of collisions. For example, if we have the hash codes 100, 200, 300, 400, and 500 that we'd like to store in an array of size 10, we'd wind up with the same value of `0` for all the codes. But by changing `N` to be a prime number, say `11`, we end up having no collisions.

Selecting a prime number for `N` isn't enough though. It's still very much possible to end up with collisions if the hash codes you need to compress are multiples of the `N`. For example, if `N` is still `11` and we need to compress the hash codes 11, 22, 33, etc. then we'd still end up with `0` as the compressed values.

### The MAD Method

This is a more elaborate function that accounts for repeated patterns in a set of hash codes we need to compress. **MAD** is an acronym standing for *Multiply-Add-Divide* and is defined as follows:

`((ai + b) mod p) mod N`

`N` is the size of the array, and must be in the end of the order of operations to fit within the array's bounds. `p` is any prime number greater than `N`. Both `a` and `b` are random integers chosen at random from the range [0, p-1] inclusive, with `a` having to be greater than `0`. Using the MAD method as our compression function results in the keys being spread out more evenly across the array (less collisions).

---

Now that we better understand how hashing works, we move to the next problem: how do we handle collisions?

As noted earlier, a *collision* happens when a hash function (hash code + compression function) computes the same key for two different objects. We try to minimize collisions using methods like the ones discussed in this post, but we'll inevitably run into them given enough inputs.

There are some clever strategies used to resolve this issue, and that'll be the focus of the next couple of posts.

- Separate Chaining
- Open Addressing