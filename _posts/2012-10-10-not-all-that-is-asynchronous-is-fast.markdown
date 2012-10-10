---
layout: post
title: Not All That is Asynchronous is Fast
description: I discover and fix the issue causing the slowdown described in the last post.
date: 2012-10-10 11:00:00
---

In my [previous post][1] I noticed a slowdown after adding GCD. As soon as I pushed that
post, I looked at my code and saw the source of the slowdown.

The heap array and related variables (like the array size and content size) are modified
only in a GCD serial queue. That way I can be sure that things happen in the order in
which they were added to the queue, and that heap operations are complete before trying to
add or remove objects. I also set up a property to access the heap count which goes
through that same queue, like so:

{% highlight objc %}
- (NSUInteger)count { 
    __block NSUInteger c;
    dispatch_sync(heap_q, ^{
        c = contentSize;
    });
    return c;
}
{% endhighlight %}

So far, so good. My problem is using this property in addObject: on the main thread, like
so:

{% highlight objc %}
if (self.count == arraySize - 1) { ///This blocks until the queue is empty!!
    dispatch_async(heap_q, ^{
        __strong id *tmp = resized(_heap, arraySize * 2, contentSize);
        for (int i = 0; i < contentSize; i++) {
            _heap[i] = nil;
        }
        free(_heap);
        _heap = tmp;
        arraySize *= 2;
    });
}
dispatch_async(heap_q, ^{
    _heap[++contentSize] = object;
    swim(_heap, contentSize, _comparator);
});
{% endhighlight %}

By using self.count on the main thread, I lose all the benefits of GCD because caller has
to wait for the queue to empty for that call to return. That method can't proceed past the
first line until the dispatch queue finishes all pending requests. That makes the
following dispatch_async calls far less effective. Once those are called, the queue is
guaranteed to be empty.

The fix is as simple as moving the entire method into the queue.

{% highlight objc %}
- (void)addObject:(id<NSObject>)object {
    dispatch_async(heap_q, ^{
        if (contentSize == arraySize - 1) {
            __strong id *tmp = resized(_heap, arraySize * 2, contentSize);
            for (int i = 0; i < contentSize; i++) {
                _heap[i] = nil;
            }
            free(_heap);
            _heap = tmp;
            arraySize *= 2;
        }
        _heap[++contentSize] = object;
        swim(_heap, contentSize, _comparator);
    });
}
{% endhighlight %}

Now there's no need for addObject to wait around under any circumstances. I had made the
same mistake for removeFirstObject, so I fixed that too. The result? Better times:

    =====================
    SBBinaryHeapPriorityQueue Timing
    =====================
    T(4000) = 0.087680	dT = 0.087680	ratio = 0.000000
    T(8000) = 0.186985	dT = 0.099305	ratio = 2.132584
    T(16000) = 0.433459	dT = 0.246474	ratio = 2.318149
    T(32000) = 0.924107	dT = 0.490648	ratio = 2.131936
    T(64000) = 2.196998	dT = 1.272891	ratio = 2.377428
    
    SBBinaryHeapPriorityQueue Alternating add/remove: 0.616692
    
    =====================
    SBPriorityQueue Timing
    =====================
    T(4000) = 0.074011	dT = 0.074011	ratio = 0.000000
    T(8000) = 0.150301	dT = 0.076290	ratio = 2.030794
    T(16000) = 0.355165	dT = 0.204864	ratio = 2.363024
    T(32000) = 0.662540	dT = 0.307375	ratio = 1.865443
    T(64000) = 1.454035	dT = 0.791495	ratio = 2.194637
    
    SBPriorityQueue Alternating add/remove: 0.365514

I ended the last post about a statement that this illustrates the limits of concurrency. I
think it better illustrates the limits of my ability to deal with concurrency.

[1]: /2012/10/10/slowing-things-down-with-gcd.html