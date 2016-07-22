---
layout: post
title:  "Build and extend the Chapel runtime"
date:   2016-06-17 19:12:31 +0200
categories: gsoc chapel
---

This week I've worked on adding the support to libunwind into the Chapel building system: it seems easy until you remember that the Chapel's runtime wants to be extremely modular and to only use standard NIX tools for keeping it portable.<!--more--> These two requirements are often in antithesis between them! So, I've brushed up my make-fu and started this work.

Chapel can uses several third party libraries if the developer sets the runtime in an appropriate way. An user can select which library should be used in the runtime using special environment variables. For example, the CHPL_MEM variable indicates the memory allocator Chapel should use (so CHPL_MEM=jemalloc means that the runtime must use jemalloc as the memory allocator). [This](http://chapel.cray.com/docs/latest/usingchapel/chplenv.html) is a list of a lot of those variables. If we want to use libunwind in our project, we must add libunwind as a third party library with a new variable for using it.

When libunwind should be linked to the runtime? And how? The first step was to introduce a new environment variable for "activate" the library. This new variable is (at the moment) CHPL_UNWIND. This means that, if for example we set CHPL_UNWIND to 'libunwind', we want to use libunwind for stack tracing, so it has to be linked to the runtime.

Without going into too much details, the main idea is that the runtime is builded with a really long name and this names includes all the variables and their values (after been sanitized by a python script). Then the main makefile for the runtime parses this name and call all the secondary makefiles that prepare all the settings needed by the compiler.

Let's make an example. We start the building of the runtime: the makefile takes all the Chapel's environment variable (using a special script that sanitize them and avoid conflicts) and generate a name like this:

{% highlight shell %}
darwin.clang.arch-native.unwind-system.mem-jemalloc
{% endhighlight %}

Each variable has a format of "name-value", separated by a ".". The general makefile parses this string for each name-value entry and call a makefile with the name of Makefile-name-value in the name. For example, when it parse 'unwind-system', it will call 'Makefile-unwind-system'. This makefile will include to a list present in the main makefile all the informations necessary to uses that library: the path to the necessary headers, compiled libraries and all the linking options. This is an extract of the Makefile-unwind-system file:

{% highlight shell %}

RUNTIME_DEFS += -DCHPL_UNWIND_LIBUNWIND

{% endhighlight %}

Which adds a define (CHPL_UNWIND_LIBUNWIND) that will be used in the source code for activate some specific paths. After this pass, the compiler now has all the options/libraries/defines needed for building the runtime as we want.

The reality is actually a lot more complex but this should give the main idea on how the Chapel runtime handles its modularity.
