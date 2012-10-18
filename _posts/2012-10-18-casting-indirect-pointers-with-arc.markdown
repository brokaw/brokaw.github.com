---
layout: post
title: Casting Indirect Pointers With ARC
description: With Automatic Reference Counting, the Objective-C compiler is particularly finicky about casting between indirect pointers. I finally discover a way to cast without warnings.
published: true
---

Last week I talked about the difficulty I had casting an `id *` pointer across the bridge
to Core Foundation. My solution in [that post][1] was to cross the bridge at a different
point. Instead of using my `void **` object to create an NSArray, I used it to create a
CFArray, which then easily transfers across the bridge.

Today I ran into a situation where that approach didn't work. I was trying to implement
NSFastEnumeration in [SBPriorityQueue][]. NSFastEnumeration is implemented in a single method
call:

{% highlight objc %}
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state
                                  objects:(id __unsafe_unretained *)stackbuf
                                    count:(NSUInteger)len
{% endhighlight %}

Since I have a C array holding my objects, the easiest and fastest thing to do is to store
a pointer to that array in the state structure and return the count of the objects in the
array. The enumeration will be truly fast--as fast as a C `for` loop. The problem I ran
into is that the field state->itemsPtr is of type `__unsafe_unretained id *`. I couldn't
figure out how to store my pointer of type `__strong id *` in that field.

First, the setup. My array of objects is declared and created like this:

{% highlight objc %}
__strong id *_heap; //ivar in @implementation
_heap = (__strong id *)calloc(MIN_SIZE + 1, sizeof(id)); //in initializer
{% endhighlight %}

I wanted to store my \_heap pointer some way similar to this:
 
{% highlight objc %}
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state
                                  objects:(id __unsafe_unretained *)stackbuf
                                    count:(NSUInteger)len
{
    if (state->state == 0) {
       NSUInteger count;
       state->mutationsPtr = &contentSize;
       state->itemsPtr = (__SOME_ARC_CAST)&_heap[1]; // ignoring first array element;
       state->state = 1;
       return contentSize;
    }
    return 0;
}
{% endhighlight %}

The $64,000 question is: What is the actual cast represented by \__SOME_ARC_CAST? First
I'll go over all the options that seemed reasonable, but didn't work.

{% highlight objc %}
state->itemsPtr = (__bridge __unsafe_unretained id *)&_heap[1];
//ERROR: Incompatible types casting '__strong id *' to '__unsafe_unretained id *' with a __bridge cast.
{% endhighlight %}

I could reduce the problem from an error to a warning with some questionable use of
`const`:

{% highlight objc %}
const id *heapPtr = &_heap[1]; // ignoring first array element;
state->itemsPtr = heapPtr;
//WARNING: Assigning to '__unsafe_unretained id *'from 'const id *' discards qualifiers.
{% endhighlight %}

I don't like compiler warnings, so I did some research on the Apple Developer Forum. I
read a suggestion to use a double cast, so I tried:

{% highlight objc %}
__unsafe_unretained id  *heapPtr = (__bridge __unsafe_unretained id *)(void **)&_heap[1];
//ERROR: cast of an indirect pointer to an Objective-C pointer to 'void **' is disallowed with ARC
{% endhighlight %}

Closer (though I didn't know it at the time), but it still produces a compiler error.

Finally, thanks to an Apple engineer, I learned that the secret door out of this maze of
qualifiers is to cast through `void *`, not `void **`, like so: 

{% highlight objc %}
state->itemsPtr = (__bridge __unsafe_unretained id *)(void *)&_heap[1];
{% endhighlight %}

And with that, fast enumeration is implemented.

[1]: /2012/10/08/stuck-on-the-toll-free-bridge.html
[SBPriorityQueue]: http://brokaw.github.com/SBDataStructures/Classes/SBPriorityQueue.html