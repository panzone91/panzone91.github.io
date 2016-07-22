---
layout: post
title:  "Recap of my GSOC 2016"
date:   2016-08-23 11:04:11 +0200
categories: gsoc chapel
---

In this post I want to collect all my work for Chapel during the summer for keeping it easily readable.<!--more--> This will not include all my commits or PRs to Chapel (you can find my commit list [here](https://github.com/chapel-lang/chapel/commits?author=panzone)), just a little presentation of what I've done.

* [Introduced guard pages in Chapel tasks](https://github.com/chapel-lang/chapel/pull/3729)

This feature is quite complex and can be better explained by this ([here]({% post_url 2016-05-31-stack-overflows-and-guard-pages %})) post. It introduces the concept of guard pages in Chapel tasks, avoiding the access and modification of some parts of the memory.

* [Float numbers precision bug](https://github.com/chapel-lang/chapel/pull/3870)

A little correction to a float precision bug in Chapel.

* [Stack traces on halt](https://github.com/chapel-lang/chapel/pull/3946)

This is my project for the GSOC. You can see [here]({% post_url 2016-05-13-my-GSOC-project %}) for more details, but the main purpose of this project was adding the ability to Chapel to produce its own stack trace. Chapel (currently) only uses an "halt" approach for error handling but, on complex programs, this approach didn't provide any indications about the state of the program at the moment of the error and it can be complex to understand. A stack trace is a valuable tool for developers during the debugging phase of complex application for better understand how an error is encountered.

* [Debug information in LLVM backend](https://github.com/chapel-lang/chapel/pull/4342)

Chapel compiler generates C code but it also possess a LLVM backend (you can activate it with CHPL_LLVM=llvm and using the --llvm compiler flag). However, this backend didn't generate debug informations on newer versions of LLVM. My job was to update it and making it working with LLVM 3.7, the version currently provided by Chapel.