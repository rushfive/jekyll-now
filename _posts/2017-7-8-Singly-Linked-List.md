---
layout: post
title: Singly Linked List
---

A *Linked List* is a fundamental data structure that allows us to easily add new items at any point in the list. Like an array, a linked list is so fundamental that it's often used internally to hold state for other large data structures. For example, a hash table can be implemented using a linked list internally to store its' entries.

All various forms of a linked list make use of nodes, which we can simply define as:

```c#
public class LinkNode<TElement>
{
    public TElement Element { get; set;}
    public LinkNode<TElement> Next { get; set; }
}
```

Nodes for all the various forms of a linked list share the `Element` property. What differs is the *pointers*, such as `Next`. The simplest of linked lists, the **Singly Linked List**, keeps a single pointer per node that points to the next node in the list.

![]({{ site.url }}/images/LinkedLists/SinglyLinkedListStructure.png)

At the minimum, a reference to the *head* of the linked list must be kept since a linked list is composed of reference type objects. It's recommended to keep a reference to the *tail* as well such that we can easily add a new item to the very end of the list. Without a reference to the tail, we would need to traverse the entire list just to add it.

<!--more-->

If you're completely new to linked lists, there are some tricky things you need to keep in mind when modifying the list. It'll become clear as we go over some of the common operations and implementations next. Here's the `interface` for the Singly Linked List we'll be implementing:

```c#
public interface ISinglyLinkedList<TElement>
{
    int Size();
    bool IsEmpty();
    TElement First();
    TElement Last();
    void AddFirst(TElement element);
    void AddLast(TElement element);
    TElement RemoveFirst();
}
```

#### Inserting a node to the very front
When adding a node to the very front (becoming the new head), you need to ensure that the new node's `Next` reference is set before setting that node as the `Head`. Recalling that the `Head` is the main reference we keep of the entire list, if we were to lose the original `Head` reference, we lose the rest of the list as well to garbage collection!

The correct step to add to the front is:
1. Create the new Node
2. Set the new node's `Next` reference to be the current `Head` node
3. Set the `Head` reference to be the new Node

```c#
public void AddFirst(TElement element)
{
    var node = new LinkNode<TElement>
    {
        Element = element
    };

    if (this.head == null)
    {
        this.head = this.tail = node;
    }
    else
    {
        node.Next = this.head;
        this.head = node;
    }

    this.size++;
}
```

Notice that we're also checking if the list is empty (head == null). If so, we simply set the `head` and `tail` to the new Node. If the list *isn't* empty, then we proceed with the steps outlined earlier by first setting the new Node's `Next` reference to be the current head Node, then resetting the `head` reference to be that node.

#### Inserting a node at the very end
This operation is much simpler compared to adding to the front. Because we keep a reference to the `tail`, we can simply create the new Node and then set the current tail's `Next` reference to the new node. Lastly, we must also re-set the tail to point to the new node and increment the size.

```c#
public void AddLast(TElement element)
{
    var node = new LinkNode<TElement>
    {
        Element = element
    };

    if (tail == null)
    {
        this.head = this.tail = node;
    }
    else
    {
        this.tail.Next = node;
        this.tail = node;
    }
    
    this.size++;
}
```

As we did when adding to the front, we check to see if the list is empty. If so, we set the current node to be both the head and tail. If it's *not* empty, we proceed as described above the code snippet.

#### Removing (and returning) the very first item 
For those familiar with a *stack* data structure, this operation is analagous to its' `pop` method. We want to remove the very first item from the list and return it. Similar to inserting an item to the front, we must ensure that the proper steps are taken here such that we don't lose the reference to the list. We cannot simply set the current `head` to `null` or we lose the rest of the list!

1. Start by checking if the list is empty, then handle appropriately (this may differ based on the language or your implementation needs)
2. Save the current' head's `Element` value in a temporary variable so that we can return it.
3. Set the `head` reference to be the current head's `Next` Node.

```c#
public TElement RemoveFirst()
{
    if (this.head == null)
    {
        throw new Exception();
    }

    TElement first = this.head.Element;
    this.head = this.head.Next;
    this.size--;

    if (this.size == 0)
    {
        this.tail = null;
    }

    return first;
}
```

We also need to check the size after removing the first Node. If the list is empty, we'll need to set the `tail` reference to `null`. The `head` reference is already handled, because if there's only one Node to remove, our code sets the `head` reference to be the current head's `Next` reference which is `null`. Lastly, we return the `Element` value that we previously saved.

Implementations for the rest of the interface are simple enough without explanations, so here's the entire **SinglyLinkedList** class below:

```c#
public class SinglyLinkedList<TElement> : ISinglyLinkedList<TElement>
{
    private LinkNode<TElement> head { get; set; }
    private LinkNode<TElement> tail { get; set; }
    private int size { get; set; }

    public SinglyLinkedList()
    {
        this.head = null;
        this.tail = null;
        this.size = 0;
    }

    public SinglyLinkedList(LinkNode<TElement> node)
    {
        this.head = this.tail = node;
        this.size = 1;
    }

    public void AddFirst(TElement element)
    {
        var node = new LinkNode<TElement>
        {
            Element = element
        };

        if (this.head == null)
        {
            this.head = this.tail = node;
        }
        else
        {
            node.Next = this.head;
            this.head = node;
        }

        this.size++;
    }

    public void AddLast(TElement element)
    {
        var node = new LinkNode<TElement>
        {
            Element = element
        };

        if (tail == null)
        {
            this.head = this.tail = node;
        }
        else
        {
            this.tail.Next = node;
            this.tail = node;
        }
        
        this.size++;
    }

    public TElement First()
    {
        return this.head.Element;
    }

    public bool IsEmpty()
    {
        return this.size == 0;
    }

    public TElement Last()
    {
        return this.tail.Element;
    }

    public TElement RemoveFirst()
    {
        if (this.head == null)
        {
            throw new Exception();
        }

        TElement first = this.head.Element;
        this.head = this.head.Next;
        this.size--;

        if (this.size == 0)
        {
            this.tail = null;
        }

        return first;
    }

    public int Size()
    {
        return this.size;
    }
}
```

A Singly Linked List is just one form of a linked list data structure. It's benefits, compared to using an array, is that insertions at any point in the list is simplified. When you add a new item to the middle of an array (in between existing items), it requires us to shift all elements after that index to be shifted to the right. This kind of extra overhead cannot be resolved using an array because an array is stored as a contiguous block of memory.

When we use a linked list, we can easily add a new item anywhere within the list simply by managing the pointers of the new node and the preceding one. To clarify, here's the steps needed to add a new item in the middle of a linked list:

```c#
[A] -> [B] -> [Add Node Here between B and C] -> [C] -> [D] -> null
```

1. Create the new node
2. Traverse the linked list, starting from the head, until you have a reference to the Node that'll be *before* the new node after it's placed (`B` above)
3. Set the new node's `Next` pointer to be Node B's Next
4. Set B's Next pointer to be the new node

It might take some practice to get used to managing the pointers and really thinking about them the right way. What's undeniable though is that the above steps are far simpler and more efficient than the steps needed to do the same with an array.

In the next post, we're going to explore a variation of a linked list, the **Circularly Linked List**, where the tail points back to the head (no null references for any node's Next).

As always with this series, you can view code samples and mess around with the interactive console tester I have in the [github repository](https://github.com/rushfive/DataStructuresAlgorithms).