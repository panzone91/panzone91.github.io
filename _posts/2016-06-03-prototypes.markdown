---
layout: post
title:  "Prototypes"
date:   2016-06-03 16:43:10 +0200
categories: gsoc chapel
---

We leaved [last week]({% post_url 2016-05-27-stack-pointers-and-symbols %}) with the definition of the architecture for my GSOC project: this week I've implemented a prototype of that architecture.<!--more-->

You can find the prototype [here](https://github.com/chapel-lang/chapel/compare/master...panzone:prototype-stack-trace). Currently it works only on some specific occasions and after a manual change in the Chapel building system, but the results are encouraging. In this post I'll try to explain the two basic parts of this prototype.

# Chapel symbol table

As we said, we need a table for converting the native symbols into their Chapel names. The solution is using a table, here implemented as a simple array, in which we puts in order:

1. The native name
2. The Chapel name
3. The source file name
4. The position in the source file name

For every Chapel symbols that it's not a runtime symbol, we create an "entry" with these informations. It's needless to say this is far from an optimal solution for both performances (for find a symbol we have to search the entire array) and used space (we puts a lot of symbols in this table. In the fact example from last week, the table itself takes almost 30 KB!). It isn't acceptable, but the purpose of this prototype is to see if my strategy works.

# Getting the stack trace

Last week, I've presented my "three step" procedure for this project. Let's see how they were implemented:

* Getting the stack trace

This was easy, because [libunwind](http://www.nongnu.org/libunwind/index.html) did all the work. Thanks to this library, obtaining the call stack and their pointers was extremely easy.

* Getting the native symbols

Again, with libunwind it was easy to convert our pointer into a string with the name of the native function.

* Converting the symbols

This is probably one of the area that will need more work. At the moment the prototype simply scan linearly the table created for each symbols and prints the values it can find! Horrible, but it works for now.

The next step will be to improve this prototype on performances and stability. But that's a work for the next week.