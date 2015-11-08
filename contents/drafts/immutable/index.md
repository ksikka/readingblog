---
title: "Immutable Data Structures"
author: ksikka
date: 2015-10-26
template: article.jade
---

In Scala and Clojure, data structures are immutable by default.
Everytime you modify a hashmap or arraylist, you get what appears to be a new modified copy
and the original one is unchanged.
While that sounds potentially expensive, **immutable
data structures can be just as efficient as their mutable counterparts**!
In this post I'll explain how.

## Structural Sharing

You want to avoid copying the entire data structure
by sharing some of the unchanged parts with the old
data structure.
But it's not as easy as it sounds.
Here are two naive strategies you can think of to implement
immutable hashmap that use structural sharing but aren't performant.

### Tradeoffs: Two Naive Strategies

#### Fast reads, slow writes

A typical imperative hashmap is an array of linked lists,
where each linked list represents a bucket of all the key/value pairs where the key hashes
to that bucket. You could do an immutable update like so:

1. Copy the array of linked lists.
2. Copy the node containing the key/value pair you wish to update.
3. Repair the bucket/linkedlist of that node by copying all the nodes
that lead to that node. This effectively creates a new bucket/linked-list.
4. Replace the old bucket list with the new bucket in the copied bucket array.
5. Perform the regular mutative hashmap operation.

Here we take advantage of Structural Sharing to avoid copying
the linked list nodes for most of the unchanged key/value pairs.
But we did an `array_size` amount work to copy all of the linked list pointers.

On the plus side, reads are as fast as in an imperative hashmap.

#### Fast writes, slow reads

`HashMap = (ChangeOperation, OldHashMap)`

In this structure, you record the modifications in a linked list,
so writes are extremely fast.
But in order to know the value associated with some key,
you may have to traverse the entire list of the modifications.
(TODO footnote: You can do slighly better in some cases if you move the key, value pair
to the front of the list when you access it, but
you'd still get `O(n)` cost for random accesses.)

We've seen two solutions and we want to reach a middle
ground between them. We don't want to copy the _entire_
bucket array in a traditional hash map as we did in Naive Solution 2,
but we want to do some copying to make reads faster
than in Naive Solution 1. One avenue we haven't explored yet
is to use trees, and it turns out, there's an efficient solution
if you go down that branch of thought.

### Let's "Trie" something else: Path Copying

Instead of an array of buckets, we could use a tree
so that we only have to copy a small portion of the tree
on update.

If we stored the key/value pairs at the leaf nodes of the tree,
we might be able to get away with only copying the path
from the root to that leaf. The rest of the tree could remain the same.

The nodes along that path could encode the hash of the key.
So to read, you hash the key and traverse the nodes
to either find the key/value pair or not. Then you can copy that path
do your modification, and the new root node is the reference
to your new hashmap.





