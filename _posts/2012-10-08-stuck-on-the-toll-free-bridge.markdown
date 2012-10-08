---
layout: post
title: Stuck on the Toll-Free Bridge
description: Automatic Reference Counting is straightforward when casting from Core Foundation types to Foundation types, but dealing with C arrays can confuse things. 
published: true
---

Something that came up when I tried to create an Objective-C wrapper around CFBinaryHeap
was how to deal with Automatic Reference Counting (ARC) when passing `void *` pointers
back and forth across the Core Foundation bridge. Core Foundation types are easy enough to
use. The problem wasn't Core Foundation itself, it was the `void *` pointers they require.

I needed to get an array of objects from Core Foundation using the function

{% highlight c %}
void CFBinaryHeapGetValues (
   CFBinaryHeapRef heap,
   const void **values
);
{% endhighlight %}

which expects an empty array of `void *` types. Then I have to take the filled-in C array
and turn it back into an NSArray. I picked the NSArray method

{% highlight objc %}
- (id)initWithObjects:(const id[])objects count:(NSUInteger)count
{% endhighlight %}

I kept failing at my attempts get the array of objects across the bridge in a way that
satisfied the compiler. My first attempt was to create a C array of type `id`. That
approach gave me an error when I tried to pass it over to a Core Foundation function which
expected a pointer to an array of `void *`

{% highlight objc %}
CFIndex size = CFBinaryHeapGetCount(_heap);
id __unsafe_unretained *values = (id __unsafe_unretained *)calloc(size, sizeof(id));
CFBinaryHeapGetValues(_heap, (const void **)values); //error
NSArray *objects = [[NSArray alloc] initWithObjects:values count:size];
{% endhighlight %}

The commented line above resulted in this error:

    Cast of an indirect pointer to an Objective-C pointer to 'const void **' is disallowed with arc.

My next attempt was to create a C array of CFTypeRef. That got through the
CFBinaryHeapGetValues() call okay, but it didn't get through the call to NSArray.

{% highlight objc %}
CFIndex size = CFBinaryHeapGetCount(_heap);
CFTypeRef *cfValues = calloc(size, sizeof(CFTypeRef));    
CFBinaryHeapGetValues(_heap, (const void **)cfValues);    
NSArray *objects = [[NSArray alloc] initWithObjects:(id *)cfValues count:size]; //error
{% endhighlight %}

This resulted in the following error:

    Pointer to a non-constant type 'id' with no explicit ownership.

Maybe I can make the ownership explicit? I couldn't figure out any legitimate ownership
qualifier. The line

{% highlight objc %}
NSArray *objects = [[NSArray alloc] initWithObjects:(id __unsafe_unretained *)cfValues count:size];
{% endhighlight %}

resulted in this error:

    Cast of a non-Objective-C pointer typ 'CFTypeRef *' (aka 'const void **) to '__unsafe_unretained_id *' is disallowed with ARC.

It was starting to feel like _everything_ was disallowd with ARC. Finally, I realized I
was trying to cross the bridge at the wrong point. The solution was to stay in Core
Foundation as long as possible. Specifically, use the C array of CFTypeRef to create a
CFArray.

{% highlight objc %}
CFIndex size = CFBinaryHeapGetCount(_heap);
CFTypeRef *cfValues = calloc(size, sizeof(CFTypeRef));    
CFBinaryHeapGetValues(_heap, (const void **)cfValues);
CFArrayRef objects = CFArrayCreate(kCFAllocatorDefault, cfValues, size, &kCFTypeArrayCallBacks);
{% endhighlight %}

I know the array is full of Cocoa objects or CFArayRef objects, so I don't even
need to deal with the messay struct of function pointers. There's a handy constant
`kCFTypeArrayCallBacks` that provides necessary retain and release functions for the array
contents. 

Once I have a CFArrayRef, I can get an NSArray with the straightforwrad ARC bridging 
rules:

    NSArray *objArray = (__bridge_transfer NSArray *)objects

And with that, I finally get a file that compiles with ARC.