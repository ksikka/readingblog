---
title: "Immutable Data Structures"
author: ksikka
date: 2015-10-26
template: article.jade
---

In Scala and Clojure, data structures are immutable by default.
Everytime you modify a hashmap, you get what appears to be a new modified copy
and the original one remains unchanged.
While that sounds impractical, **immutable
data structures can be just as efficient as their mutable counterparts!**
In this post I'll explain how.

## Your typical hashmap

A typical imperative hashmap is an array of linked lists,
where each linked list represents a bucket of all the key/value pairs where the key hashes
to that bucket.

<section class="illustration">
  <table class="vertical-array">
    <tr><td>
      <div class="ll-wrapper">
        <div class="ll-pointer">
        </div><div class="ll-edge">
        </div><div class="ll-edge">
        </div><div class="ll-node">
        </div><div class="ll-edge">
        </div><div class="ll-node">
        </div><div class="ll-edge">
        </div><div class="ll-node"></div>
      </div>
    </td></tr>
    <tr><td>
      <div class="ll-wrapper">
        <div class="ll-pointer">
        </div><div class="ll-edge">
        </div><div class="ll-edge">
        </div><div class="ll-node"></div>
      </div>
    </td></tr>
    <tr><td></td></tr>
    <tr><td>
      <div class="ll-wrapper">
        <div class="ll-pointer">
        </div><div class="ll-edge">
        </div><div class="ll-edge">
        </div><div class="ll-node">
        </div><div class="ll-edge">
        </div><div class="ll-node"></div>
      </div>
    </td></tr>
    <tr><td>
      <div class="ll-wrapper">
        <div class="ll-pointer">
        </div><div class="ll-edge">
        </div><div class="ll-edge">
        </div><div class="ll-node"></div>
      </div>
    </td></tr>
  </table>
</section>

You could make it immutable by performing updates like so:

```
/                                                                           \




                               ( IMAGE HERE )




\                                                                           /
```

1. Access and copy the node you wish to update.
2. Copy all the nodes that lead to that node.
3. Copy the array and write the pointer to copied list of nodes into the right array cell.
4. Perform the regular mutative hashmap operation.

We didn't copy _all_ the data, but we did copy the _entire_ array.
The array size of a hash map is usually proportional to `n`
in order to guarantee constant time operations.
So this is still `O(n)`! (TODO footnote) On the plus side, reads are just as fast as in an imperative hashmap.

## Let's "Trie" something else

Instead of an array of buckets, we could use a tree.
Specifically, we can use a tree that uses the hash of
a key to _route_ us to the location of the key/value pair.

```
/                                                                           \




                               ( IMAGE HERE )




\                                                                           /
```

The hash of a key is 32 bits, and if you sliced the bit sequence into 4-bit segments,
you can get to your destination in at most 8 jumps.
First, you ask the root node, for the first slice of my hash,
does there exist a key in this hashmap?

To which the node replies in one of three ways:
1. No
2. Yes, go ask that node over there and provide your next bit-slice.
3. Yes, there's just one, and here it is.

The way the node knows this is that the node keeps an
array of size 2^4 (16)(TODO footnote on NULL POINTER OPTIM). It interprets the 4-bit slice
you give it as an int between 0 and 15, which it
can use to index into its array. If the array cell
at that index is null, it tells you No.
If it's Yes it can either be a Tree Node pointer,
or a key/value pair.

(TODO Footnote This structure is a clever use of the [Trie](https://en.wikipedia.org/wiki/Trie),
also known as a Prefix Tree. Instead of one bit per node,
we compressed the depth of the trie from 32 to 8 by using 4 bits per node.
This came at the cost of making our nodes larger: each has a 16-length array.)

Now if we want to update our hashmap immutably, we need only copy the path
to the node with the key/value pair we want to change. (TODO Footnote This general technique is called Path Copying.)
(TODO Footnote In small tries, this path will be far short of 8 nodes, since
we stop creating nodes once the prefix of the hash is unique in the trie.)

```
/                                                                           \




                               ( IMAGE HERE )




\                                                                           /
```

## But is it fast?

Now we know the basics of how immutable hashmaps work,
are they really performant enough to be used in place of
mutable ones? There are a few performance optimizations
to the structure I described above that make the answer: **Yes**.

The structure that Clojure and Scala use in production
is Hash Array Mapped Trie...

1. Access
2. Updates
3. Insertion

## The benefits of immutability

Anyone who's experienced purely functional programming
will intuitively understand the value of immutability.
It causes a fundamental shift in the way you think about programming.
But long story short, there are at least three practical benefits
that even an imperative programmer can empathize with:

1. You can safely pass objects as values to any other piece of code,
without worrying about undesired mutation.
    - Mutation to a data structure can get spread across many
    different files and actors, devolving into runaway complexity over time.

2. You can share data structures across threads without tricky locking code.
    - No longer any write-contention on a data structure.

3. Memory-efficient cloning.
    - Technically this is a benefit of the way immutable data structures
    are implemented rather than immutability itself.

## Why aren't we using these on a daily basis?


<!--
#### Balogna

`HashMap = (ChangeOperation, OldHashMap)`

In this structure, you record the modifications in a linked list,
so writes are extremely fast.
But in order to know the value associated with some key,
you may have to traverse the entire list of the modifications.
(TODO footnote: You can do slighly better in some cases if you move the key/value pair
to the front of the list when you access it, but
you'd still get `O(n)` cost for random accesses.)

We've seen two solutions and we want to reach a middle
ground between them. We don't want to copy the _entire_
bucket array in a traditional hash map as we did in Naive Solution 2,
but we want to do some copying to make reads faster
than in Naive Solution 1. One avenue we haven't explored yet
is to use trees, and it turns out, there's an efficient solution
if you go down that branch of thought.


-->


