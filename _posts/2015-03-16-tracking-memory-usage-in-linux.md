---
layout: post
title: "Tracking memory usage in Linux"
description: ""
category:
tags: []
---

When working with big data optimizing the memory footprint is important.

In this example we're serializing a data frame with 50 million elements using R's native `serialize` function:

    df <- data.frame(runif(50e6,1,10))
    ser <- serialize(df,NULL)

Each element is a double that takes 8 bytes. If we do the math the data frame should be 400MB (50M elements, 8byte each). The serialized version should be around 400MB too. However if we run that code and check the memory usage we see the following:

    $ cat /proc/15155/status |grep Vm
    ...
    VmHWM:	 1207792 kB
    VmRSS:	  817272 kB

`VmRSS` is the resident memory and in this case it's around 800MB as we'd expect. However the peak memory usage (`VmHWM`) is 1.2GB. Let's fire up `GDB` and see what's going on. The relevant R code is:

    InitMemOutPStream(&out, &mbs, type, version, hook, fun);
    R_Serialize(object, &out);
    val =  CloseMemOutPStream(&out);
    ...
    return val;

If we set a breakpoint right after `R_serialize` we see that the memory usage is around 800MB, as we'd expect:

    VmRSS:	  816664 kB

However is we step into `CloseMemOutPStream` we see this:

    PROTECT(val = allocVector(RAWSXP, mb->count));
    memcpy(RAW(val), mb->buf, mb->count);
    free_mem_buffer(mb);

The code is allocating a whole new buffer and copying the serialized object there. If we set a breakpoint just before `free`, the memory usage at that point is 1.2GB.

    VmRSS:	 1207384 kB

### Optimizing the code

Using R's `serialize` we need 3x the amount of original memory (1.2GB for a 400MB) data frame, which is not acceptable.

Ideally we'd like to avoid the last copy and just serialize the object in a final buffer.

The other improvement would be to serialize the object in chunks. E.g. we could have a 10MB buffer and stream parts of the serialized object from that buffer. In this case the peak memory usage would be 410MB instead of 1.2GB!
