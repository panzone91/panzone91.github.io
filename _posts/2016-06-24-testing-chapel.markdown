---
layout: post
title:  "Testing Chapel"
date:   2016-06-24 14:10:04 +0200
categories: gsoc chapel
---

This week was the GSOC midterm evaluation week and I've finished the first version of my GSOC project. However, before merging my new patch, I had to write some tests.<!--more--> Because this was basically my main work this week, I'll talk a little about software testing in this post.

A test is a simple program that try our program with a specific input. The main idea about tests is simple: we run the test and compare the results with a "good" (expected) output. If the output is different from the expected one, then the test failed and there is a problem.

Testing is essential for every software but it's especially important in open source applications since they are developed by thousand of people around the world without a central coordination. Almost all of the open source software possess some test suites (collections of tests) and, every time the software is builded, the new build is passed through these test suites.

Chapel obviously has it own test suite. In fact, an enormous part of the Chapel repository is the testing system! There are tests for every part of the language: runtime, compiler, standard modules... every Chapel feature possess tests. Since my patch introduces a new feature, I had to write some tests for avoiding that a future patch will break my project.

Chapel, for testing, uses a couple of python scripts for generate a "virtual testing environment", an isolate instance of Chapel with specific runtime settings. A Chapel test is typically composed by 4 files:

1. A Chapel program 
2. The expected output (in a .good file)
3. A .skipif file, which indicates the conditions to skip the test. Sometimes a test don't have a sense if the runtime is builded with a specific configuration (for example, it's useless to test the jemalloc allocations if the runtime uses cstdlib)
4. A .prediff file, which is a script that will be executed after the execution of the test but before the confrontation with the .good file. This is important, since sometimes we can have the necessity to modify the .good file based on the execution machine. 

Let's take one of the tests I've written during this week as an example. We start with the .skipif file: we obviously want to skip the test if the runtime isn't builded with unwind support or if the printing of the stack trace is disabled.

{% highlight shell %}
CHPL_RT_UNWIND==0
CHPL_UNWIND==none
{% endhighlight %}

A good input program in my case is a simple, recursive, factorization procedure. With it I can test nested procedure calls (thanks to recursion) and its output is extremely predictable. This is the program:

{% highlight chapel %}
proc fact(i : int) :int{
  if(i == 0) then halt();
  else return i*fact(i-1);
}

fact(2);
{% endhighlight %}

and this should be the output we expected (our .good file)

{% highlight shell %}
fact-stacktrace.chpl:2: error: halt reached

Stacktrace

halt (/Users/panzone/workspace/chapel/modules/internal/ChapelIO.chpl:679)
fact (fact-stacktrace.chpl:1)
fact (fact-stacktrace.chpl:1)
fact (fact-stacktrace.chpl:1)
{% endhighlight %}

Uh, there is a problem with our .good file. Some procedures (like halt) are part of the standard modules, which reside in the Chapel installation directory (the Chapel home). In my example, they are in /Users/panzone/workspace/chapel/modules/ but another machine will have another path! For solving this, we create the .good file dynamically with our .prediff script.

{% highlight shell %}
#! /bin/bash

echo "$1.chpl:2: error: halt reached" > $1.good
echo "" >> $1.good
echo "Stacktrace" >> $1.good
echo "" >> $1.good
echo "halt ($CHPL_HOME/modules/internal/ChapelIO.chpl:679)" >> $1.good
echo "fact ($1.chpl:1)" >> $1.good
echo "fact ($1.chpl:1)" >> $1.good
echo "fact ($1.chpl:1)" >> $1.good
echo "" >> $1.good
{% endhighlight %}

Now the test is ready and if we execute it with the Chapel testing suite...

{% highlight shell %}
[test: runtime/panzone/fact-stacktrace.chpl]
[Executing compiler /Users/panzone/workspace/chapel/bin/darwin/chpl -o fact-stacktrace --cc-warnings fact-stacktrace.chpl < /dev/null]
[Elapsed compilation time for "runtime/panzone/fact-stacktrace" - 6.641 seconds]
[Success compiling runtime/panzone/fact-stacktrace]
[Executing program ./fact-stacktrace  < /dev/null]
[Elapsed execution time for "runtime/panzone/fact-stacktrace" - 0.032 seconds]
[Executing prediff ./fact-stacktrace.prediff]
[Executing diff fact-stacktrace.good fact-stacktrace.exec.out.tmp]
[Success matching program output for runtime/panzone/fact-stacktrace]
[Elapsed time to compile and execute all versions of "runtime/panzone/fact-stacktrace" - 6.692 seconds]

[Test Summary]
[Summary: #Successes = 1 | #Failures = 0 | #Futures = 0 | #Warnings = 0 ]
[Summary: #Passing Suppressions = 0 | #Passing Futures = 0 ]
[END]
{% endhighlight %}

which means that our test is now in the Chapel testing system.

My project is now merged into Chapel's master branch ([here](https://github.com/chapel-lang/chapel/commit/b4fc70d80251e25496f52bbfb7aa2b6b72f95dfd)) and I think this is a good conclusion for the first period of this GSOC.
