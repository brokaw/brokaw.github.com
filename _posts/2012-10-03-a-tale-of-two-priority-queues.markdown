---
layout: post
title: An Tale of Two Priority Queues
description: I compare two object-oriented priority queues for Objective-C objects. One using using Core Foundation, another using a C array.
published: true
---

The Cocoa Foundation framework doesn't have a priority queue built-in. When I went looking
for an Objective-C priority queue, I found a few examples, but nothing that jumped out as
the canonical implementation. Maybe if you know enough to know you need a priority queue,
you also know how to make one from scratch. In any case, I decided to take this as an
invitation to build my own implementation.

I liked the idea of building it on top of CFBinaryHeap in Core Foundation, but I wanted my
implementation to be a little more general than the version [here][1]. In particular, I
wanted to pass a comparator function (or better: comparator block) to the object instance
so that I could have multiple instances of the same priority queue class deal with
different types of content objects. The implementation I found requires a new subclass for
every different content object type. That approach has merit in that you can statically
type your contents. My approach holds content of type `id`, and it is up to the client
programmer to make sure the comparator block deals with the contents of the queue properly.

Cocoa defines an NSComparator block that does exactly what I want. The problem is that
CFBinaryHeap takes a function pointer for the comparator. To use NSComparator, I have to
write a C function which calls my comparator. Since I'm calling it from inside a C
function and not from inside an object method, I can't simply store an NSComparator
reference in an instance variable and call it whenever I need it.

The solution I came up with was to:

1. Keep an instance variable in the priority queue to hold the NSComparator.
2. When an object is added to the queue, I wrap it in a private _node_ object.
3. I also assign the comparator from the priority queue to an instance variable in the
node object. This is kept as a weak reference.
4. Insert the node object into the CFBinaryHeap.

Since the node will be passed into the comparison function, I unwrap the objects there,
pull the NSComparator out of the enclosing node, use it on the objects I unwrapped inside
the comparison callback function, and return the result.

This seemed to work well, but I wondered how I would do if I created my own priority queue
from scratch, using a straight C array for the heap. This would have a couple of
additional benefits:

1. Since I'm heapifying the array myself, I can use a block directly to compare the content
objects, instead of having to wrap the block inside a function.
1. I can store the content objects directly in my array, instead of wrapping them inside a
node.
1. I can easily support fast enumeration on the raw heap contents.

I experimented with moving some of the most used methods to C functions for performance. I
could detect a slight speed increase each time I moved a method to a function, and I got a
little carried away. By the end of it, my block objects were called from C functions
again. The difference was that I had defined my functions to accept an NSComparator
pointer, so the function had easy access to the comparator (remember the original problem
was that the function had no easy way to get the comparator block).

Here are some speed results. I start with an empty queue, fill it with random numbers, and
remove the min until empty. _T(n)_ is the time in seconds it took to process _n_ random
numbers. dT is the difference between the previous trial an this trial. Ratio is the ratio
of the previous run time to the current run time. 

First, the queue based on CFBinaryHeap:

    ================================
    SBBinaryHeapPriorityQueue Timing
    ================================
    T(4000) = 0.040881	dT = 0.040881		ratio = 0.000000
    T(8000) = 0.083901	dT = 0.043020		ratio = 2.052323
    T(16000) = 0.164073	dT = 0.080172		ratio = 1.955555
    T(32000) = 0.377770	dT = 0.213697		ratio = 2.302451
    T(64000) = 0.863485	dT = 0.485715		ratio = 2.285743

Next my home-grown priority queue:

    =====================
    SBPriortyQueue Timing
    =====================
    T(4000) = 0.016533	dT = 0.016533		ratio = 0.000000
    T(8000) = 0.034372	dT = 0.017839		ratio = 2.078997
    T(16000) = 0.086743	dT = 0.052371		ratio = 2.523655
    T(32000) = 0.161758	dT = 0.075015		ratio = 1.864796
    T(64000) = 0.350667	dT = 0.188909		ratio = 2.167849

There doesn't seem to be enough trials to say that SBBinaryHeapPriorityQueue is going to
converge on a fixed ratio, but I'm going to go out on a limb and say that both
implementations have the same order of growth. The home-grown version is about twice as
fast in this test though, so the constant factors seem significantly lower. This test
isn't the only, or even the best, indicator of overall performance, though. It would be
interesting to fill up the priority queue, then time successive insertions and deletions
of increasingly greater number of objects.

Code repository [available on Github](https://github.com/brokaw/SBDataStructures).

[1]: http://three20.pypt.lt/cocoa-objective-c-priority-queue