---
layout: post
title: "Going faster with Just-In-Time compilation"
description: ""
category: 
tags: []
---

This post is about code optimization using [Just-In-Time compilation](https://en.wikipedia.org/wiki/Just-in-time_compilation). The main insight is that there is a tradeoff between generality and performance. In other words if you want to make things generic they will be slower than a specialized solution.

Let’s use an example to make things clearer. Say we’re working on an analytics DB and we want to sum 2 columns. If we code a generic solution we cannot make any assumption about the column types, the number of rows in the columns or the hardware characteristics. A generic code would then be:

    // Let’s assume the column are stored in arrays a and b
    // and we want to store the result in array c
    if (type == ‘double) {
        for (int i = 0; i < nrows; ++i) {
            c[i] = a[i] + b[i];
        }
    }

The compiler cannot do much to optimize this code. It has a conditional and we don’t know the value of `nrows` so we cannot optimize the loop.

However at runtime we’ll have more information (like number of rows, types, etc.) and we can leverage it to create more efficient code.  Let’s say we know the column type is double and we also know the hardware supports [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) instructions that can add 2 pairs of doubles in a single cycle. We can generate optimizing code on the fly like this:

    | mov eax, a
    | movapd xmm0, oword [eax]  // load array a using SSE
    | mov eax, b
    | movapd xmm1, oword [eax]  // load array b using SSE
    | addpd xmm0, xmm1              // add arrays a and b in a single cycle using SSE
    | mov eax, c
    | movapd oword [eax], xmm0  // store result in array c using SSE
    | ret

You can take a look at the full code (adapted from this excellent [example](http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html)) [here](https://github.com/jorgemarsal/jitdemo/blob/master/sse.dasc). We use `dynasm` to generate the machine code that performs the additions using SSE. Then we copy that machine code to a memory region marked as executable and jump to it.

With this optimization the performance improves dramatically. 1 million iterations of the first example run in 0.04 seconds while the second example runs in 0.002 seconds. That’s a 20X speedup!

This is a pretty contrived example but a generalization of this technique is extremely useful to improve performance in real world cases and companies like [Databricks](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html) use it in their products.

Pretty cool huh?

