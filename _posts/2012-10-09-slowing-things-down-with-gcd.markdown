---
layout: post
title: Slowing Things Down with Grand Central Dispatch
description: After updating my priority queue to use GCD, the timing test indicate it actually slowed down slightly.
---

Grand Central Dispatch (GCD) is a way to easily add concurrency to your project. So I
couldn't resist the urge to add concurrency to SBPriorityQueue. This is a situation where
I really can't speed anything up, but I can return from some calls faster.

For example, when you add to the queue, you're not expecting a return value back, so you
can just throw your object on the queue and continue on other work before the queue is
done prioritizing your new object. The heapify-ing can happen on a background thread, but
it still has to happen, and it has to take about the same amount of time.

If you add a bunch of objects, then pop the minimum, you'll have to wait for all pending
heapifying to finish before the queue can return the minimum. All the time you saved is
potentially lost. Depending on your use case, you might not get any speed benefits. The
timing tests I run are all in a tight loop of adding and removing objects, so I actually
didn't expect too much of an improvement. It turns out the overhead of GCD slowed things
down a bit for my tests.

Comparative times are below. The five test with the doubling input size fill the queue
with _N_ random objects and then empty it. Indicative of heapsort, perhaps, but not
exactly a real-world usage. The _Alternating add/remove_ test starts out with a queue of
1000 objects. Then adds another 1000, and removes 1000 minimums. Then adds another 1000
random objects, and removes 1000. It does this 21 times, and the number you see is the
total time for that sequence.

Before GCD, I was all happy that I had shaved a few milliseconds off what I could do with
SBBinaryHeapPriorityQueue. An example of that is here.

**Times before GCD:**

    ================================
    SBBinaryHeapPriorityQueue Timing
    ================================
    T(4000) = 0.091548	dT = 0.091548	ratio = 0.000000
    T(8000) = 0.173290	dT = 0.081742	ratio = 1.892888
    T(16000) = 0.487516	dT = 0.314226	ratio = 2.813295
    T(32000) = 0.888305	dT = 0.400789	ratio = 1.822104
    T(64000) = 1.903556	dT = 1.015251	ratio = 2.142908
    
    Alternating add/remove: 0.487578
    
    =====================
    SBPriortyQueue Timing
    =====================
    T(4000) = 0.040244	dT = 0.040244	ratio = 0.000000
    T(8000) = 0.067305	dT = 0.027061	ratio = 1.672425
    T(16000) = 0.155441	dT = 0.088136	ratio = 2.309500
    T(32000) = 0.314812	dT = 0.159371	ratio = 2.025283
    T(64000) = 0.668882	dT = 0.354070	ratio = 2.124703
    
    Alternating add/remove: 0.162185


After I added GCD to SBPriorityQueue, the times are for all practical purposes identical
to SBBinaryHeapPriorityQueue.

**Times after GCG:**

    ================================
    SBBinaryHeapPriorityQueue Timing
    ================================
    T(4000) = 0.094204	dT = 0.094204	ratio = 0.000000
    T(8000) = 0.212039	dT = 0.117835	ratio = 2.250849
    T(16000) = 0.414159	dT = 0.202120	ratio = 1.953221
    T(32000) = 0.853801	dT = 0.439642	ratio = 2.061530
    T(64000) = 1.833983	dT = 0.980182	ratio = 2.148022
    
    SBBinaryHeapQueue Alternating add/remove: 0.494764


    =====================
    SBPriortyQueue Timing
    =====================
    T(4000) = 0.112605	dT = 0.112605	ratio = 0.000000
    T(8000) = 0.196670	dT = 0.084065	ratio = 1.746548
    T(16000) = 0.398301	dT = 0.201631	ratio = 2.025225
    T(32000) = 0.806359	dT = 0.408058	ratio = 2.024497
    T(64000) = 1.679386	dT = 0.873027	ratio = 2.082678
    
    SBPriorityQueue Alternating add/remove: 0.457388

It's possible these test are still artificial, and the benefit of an asynchronous
`addObject:` and `removeFirstObject` will make up for the overhead in the real world. Also
possible is that I've added GCD inefficiently. In any case, I'll take this as an
illustration of the limits of concurrency.