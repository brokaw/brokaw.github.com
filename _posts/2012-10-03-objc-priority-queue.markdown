---
layout: post
title: An Objective-C Priority Queue
description: I create an object-oriented priority queue for Objective-C objects using Core Foundation.
published: false
---

I went looking for an Objective-C priority queue recently and came up empty handed. 
There
were a couple of sites with code posted directly in a blog post, but I couldn't find a
source repository.

I did find that Core Foundation has a class _CFBinaryHeap_, which is pretty much a
priority queue by another name. Being Core Foundation it is a straight C API, with a C
function pointer for the comparator and void* pointers for the contents. I wanted a more
Objective-C approach, and in particular I wanted to use a block for the comparator
function. I decided to write an Objective-C wrapper around CFBinaryHeap.

The only tricky implementation is the comparator function. Cocoa defines an NSComparator
block that does exactly what I want. The problem is that I have to use it from a C
function, so I can't reference an NSComparator instance variable kept in the priority
queue object.

The solution I came up with was to:

1. Keep an instance variable in the priority queue to hold the NSComparator. 2. When an
object is added to the queue, I wrap it in a private _node_ object. 3. I also assign the
comparator from the priority queue to an instance variable in the node object. This is
kept as a weak reference. 4. Insert the node object into the CFBinaryHeap.

Since the node will be passed into the comparison function, I unwrap the objects there,
pull the NSComparator out of the enclosing node, use it inside the comparison callback,
and return the result.

So far, it seems to work. The coming to github soon.


