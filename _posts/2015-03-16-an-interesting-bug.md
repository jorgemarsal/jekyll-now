---
layout: post
title: "An interesting bug"
description: ""
category:
tags: []
---

The other day I hit an interesting bug and I thought it would be fun to share it.

The interesting part is that it only happens with some compiler versions.  E.g. when compiled with GCC 4.8.2 on Ubuntu the software hangs. When compiled with GCC 4.6.4 it doesn't. Let's see what's going on:

    library(distributedR)
    distributedR_start(inst=1, log=3)
    Workers registered - 1/1.
    All 1 workers are registered.
    Master address:port - 127.0.0.1:50002
    <hung>

The prompt is not coming back. If we run `top`, `R-executor-bin` is stuck using 100% of a core.


## Debugging

Let's see where is stuck. First we need the pid of `R-executor-bin` which in this case is `57725`:

    $ gdb bin/R-executor-bin 57725
    ...
    0x0000000000443e55 in dataptr (x=0x6509e60) at  /usr/local/lib/R/site-library/Rcpp/include/Rcpp/routines.h:197
    197	inline void* dataptr(SEXP x){
    (gdb) bt
    #0  0x0000000000443e55 in dataptr (x=0x6509e60) at /usr/local/lib/R/site-library/Rcpp/include/Rcpp/routines.h:197
    #1  0x000000000043aee0 in r_vector_start<24> (x=0x6509e60) at /usr/local/lib/R/site-library/Rcpp/include/Rcpp/internal/r_vector.h:32
    #2  update (v=<synthetic pointer>, this=<synthetic pointer>) at /usr/local/lib/R/site-library/Rcpp/include/Rcpp/vector/traits.h:40
    #3  update (this=<synthetic pointer>) at /usr/local/lib/R/site-library/Rcpp/include/Rcpp/vector/Vector.h:428
    #4  set__ (x=<optimized out>, this=<synthetic pointer>) at /usr/local/lib/R/site-library/Rcpp/include/Rcpp/storage/PreserveStorage.h:22
    #5  Vector (size=@0x7fffb4f70bc0: 38, this=<synthetic pointer>) at /usr/local/lib/R/site-library/Rcpp/include/Rcpp/vector/Vector.h:107
    #6  ReadRawArgs (R=...) at platform/executor/src/executor.cpp:474
    #7  0x0000000000436659 in main (argc=<optimized out>, argv=<optimized out>) at platform/executor/src/executor.cpp:709

The process is stuck in the `dataptr` function. GDB tell me is this one:

`Rcpp/routines.h:197`:

    197	inline void* dataptr(SEXP x){
    198	    typedef void* (*Fun)(SEXP) ;
    199	    static Fun fun = GET_CALLABLE("dataptr") ;
    200	    return fun(x) ;
    201	}

The function is calling itself in an infinite loop! This code doesn't make any sense. The function above is used to call `dataptr` at runtime.

`GET_CALLABLE` is a macro pointing to `R_GetCCallable`:

    #define GET_CALLABLE(__FUN__) (Fun) R_GetCCallable( "Rcpp", __FUN__ )

Let's see what's `R_GetCCallable`. It's defined in `Rdynload.h` in R:

    /* Experimental interface for exporting and importing functions from
    one package for use from C code in a package.  The registration
    part probably ought to be integrated with the other registrations.
    The naming of these routines may be less than ideal. */

    void R_RegisterCCallable(const char *package, const char *name, DL_FUNC fptr);
    DL_FUNC R_GetCCallable(const char *package, const char *name);

The corresponding registration part is in `Rcpp_init.cpp:109`:

    RCPP_REGISTER(dataptr)
    #define RCPP_REGISTER(__FUN__) R_RegisterCCallable( "Rcpp", #__FUN__ , (DL_FUNC)__FUN__ );

But wait. That registration is not pointing to `dataptr` in `Rcpp/routines.h:197`. It's pointing to another `dataptr` in `Rcpp/src/barrier.cpp:71`:

    // [[Rcpp::register]]
    void* dataptr(SEXP x){
        return DATAPTR(x);
    }

That's bad. There are 2 functions with the same name and we're registering one but getting the other one at runtime! And to add insult to the injury the latter is calling itself in an infinite loop. But there are 2 weird things here:

1.  This only fails in some compiler versions!
2.  The process is hung but is should segfault. If we keep recursing indefinitely at some time we'll run out of stack space and the process will be killed.


Let's try to find out the reason for #1. Let's compile with both GCC 4.6.4 and GCC 4.8.2 and see the differences in the symbol tables:

    $ nm --dynamic R-executor-bin-4.8 |grep dataptr
    0000000000443e50 W _Z7dataptrP7SEXPREC
    00000000006c6060 u _ZGVZ7dataptrP7SEXPRECE3fun
    00000000006c5fe8 u _ZZ7dataptrP7SEXPRECE3fun

    $ nm --dynamic  R-executor-bin-4.6 |grep dataptr
    00000000006cafa0 u _ZGVZ7dataptrP7SEXPRECE3fun
    00000000006cafa8 u _ZZ7dataptrP7SEXPRECE3fun

Aha! GCC 4.8 has removed one of the `dataptr` definitions (the wrong one) so we're getting what we expect at runtime. Unfortunately GCC 4.6 has not.

## Reproducing the hang with a small program

Let's try to reproduce it with a small example:

    #include <Rcpp.h>
    #include <RInside.h>
    #include <iostream>

    int main() {
        RInside R(0, 0);
        int size = 38;
        Rcpp::RawVector raw(size);
        std::cout << "done" << std::endl;
        return 0;
    }

    $ g++-4.6 -I/usr/local/lib/R/site-library/RInside/include/ -I/usr/local/lib/R/site-library/Rcpp/include/ -I/usr/share/R/include/ -O0 -g  -o rcpphang-4.6 rcpphang.cpp -lR -lRInside -L/usr/local/lib/R/site-library/RInside/lib/
    $ g++-4.8 -I/usr/local/lib/R/site-library/RInside/include/ -I/usr/local/lib/R/site-library/Rcpp/include/ -I/usr/share/R/include/ -O0 -g  -o rcpphang-4.8 rcpphang.cpp -lR -lRInside -L/usr/local/lib/R/site-library/RInside/lib/

Let's run it:

    $ ./rcpphang-4.8
    done
    $ ./rcpphang-4.6
    done

Wait, what? This is working! No hang. Let's take a look at the symbol tables:

    $ nm --dynamic rcpphang-4.6 |grep dataptr
    <no output>

    $ nm --dynamic rcpphang-4.8 |grep dataptr
    <no output>

Ok, that's why it's working. `dataptr` is not in the dynamic symbol tables. Looking at the original SW (that hung) I see we compile with `-rdynamic`:

  **-rdynamic**: Pass the flag -export-dynamic to the ELF linker, on targets that support it. This instructs the linker to add all symbols, not only used ones, to the dynamic symbol table. This option is needed for some uses of dlopen or to allow obtaining backtraces from within a program.

Let's compile with `-rdynamic`:

    $ g++-4.6 -I/usr/local/lib/R/site-library/RInside/include/ -I/usr/local/lib/R/site-library/Rcpp/include/ -I/usr/share/R/include/ -O0 -g -rdynamic -o rcpphang-4.6 rcpphang.cpp -lR -lRInside -L/usr/local/lib/R/site-library/RInside/lib/
    $ g++-4.8 -I/usr/local/lib/R/site-library/RInside/include/ -I/usr/local/lib/R/site-library/Rcpp/include/ -I/usr/share/R/include/ -O0 -g  -o rcpphang-4.8 rcpphang.cpp -lR -lRInside -L/usr/local/lib/R/site-library/RInside/lib/

The symbols are there now:

    $ nm --dynamic rcpphang-4.6-rdyn-noopt|grep dataptr
    00000000004039f8 W _Z7dataptrP7SEXPREC
    0000000000606668 u _ZGVZ7dataptrP7SEXPRECE3fun
    0000000000606670 u _ZZ7dataptrP7SEXPRECE3fun

    $ nm --dynamic rcpphang-4.8-rdyn-noopt|grep dataptr
    0000000000403a08 W _Z7dataptrP7SEXPREC
    0000000000606648 u _ZGVZ7dataptrP7SEXPRECE3fun
    0000000000606650 u _ZZ7dataptrP7SEXPRECE3fun

    $ ./rcpphang-4.8
    <segfault>

    $ ./rcpphang-4.6
    <segfault>

Ok, now both have the symbol and therefore both segfault! The other difference is that we're compiling the original SW with `-O2`:

    $ g++-4.6 -I/usr/local/lib/R/site-library/RInside/include/ -I/usr/local/lib/R/site-library/Rcpp/include/ -I/usr/share/R/include/ -O2 -g -rdynamic -o rcpphang-4.6 rcpphang.cpp -lR -lRInside -L/usr/local/lib/R/site-library/RInside/lib/
    $ g++-4.8 -I/usr/local/lib/R/site-library/RInside/include/ -I/usr/local/lib/R/site-library/Rcpp/include/ -I/usr/share/R/include/ -O2 -g  -o rcpphang-4.8 rcpphang.cpp -lR -lRInside -L/usr/local/lib/R/site-library/RInside/lib/


    $ nm --dynamic rcpphang-4.8|grep dataptr
    00000000006046a8 u _ZGVZ7dataptrP7SEXPRECE3fun
    00000000006046a0 u _ZZ7dataptrP7SEXPRECE3fun
    $ nm --dynamic rcpphang-4.6|grep dataptr
    0000000000402b50 W _Z7dataptrP7SEXPREC
    00000000006046b8 u _ZGVZ7dataptrP7SEXPRECE3fun
    00000000006046c0 u _ZZ7dataptrP7SEXPRECE3fun

Now we have the same behavior as with the original SW! GCC 4.8 has removed the symbol while 4.6 hasn't. That's why 4.8 works and 4.6 hangs:

    $ ./rcpphang-4.8
    done

    $ ./rcpphang-4.6
    <hung>


Now regarding weirdness #2, the explanation for the hang instead of the segfault is probably that the compiler is doing [tail call optimization](http://c2.com/cgi/wiki?TailCallOptimization).

## Fixing the problem

One way to fix this issue is to make sure the conflicting function is not exported in the shared library. We can achieve that adding the following attribute:

    inline __attribute__ ((visibility ("hidden"))) void* dataptr(SEXP x){

In the end we reported the bug to the `Rcpp` author and he applied the fix in `Rcpp 0.11.5`.

That was fun :)
