---
layout: post
title:  "Stack pointers and symbols"
date:   2016-05-27 13:30:00 +0200
categories: gsoc chapel
---

Another Friday, another update ! How I've spent the first official week of the coding period ? <!--more-->

As you can read from this [post]({% post_url 2016-05-13-my-GSOC-project %}), my work for the summer is to make Chapel prints a stack trace when an halt instruction is encountered (because halt assumes that a critic error occurred). 

We've already seen what a stack trace is, now let's see how to implement it ! During the Community Bonding period I've discussed with my mentor about the best strategy for implementing this project. There are some libraries, like [libunwind](http://www.nongnu.org/libunwind/index.html), that permit to get the current stack from the program itself but Chapel presents a problem with this approach. 

Chapel is currently compiled into native code, with the runtime added as a static library. Chapel code is translated into native code and uses several runtime functions during the execution. For example, take this Chapel program:

{% highlight chapel %}
proc main(){
  fact(2);
}

proc fact(i : int) :int{
  if(i == 0) then halt();
  else return i*fact(i-1);
}
{% endhighlight %}

and then we get a stack trace during the execution:

{% highlight shell %}
frame #0: halt
frame #1: fact_chpl
frame #2: fact_chpl
frame #3: fact_chpl
frame #4: chpl_user_main
frame #5: chpl_gen_main
frame #6: chpl_executable_init
{% endhighlight %}

Uh, the functions names are quite different from what we have defined and some functions aren't even present in our original Chapel source !

Obviously a stack trace with these informations isn't what we want: we want a "Chapel stack trace", with Chapel names and without the runtime functions. Basically, we have a way to get the stack trace, but even if we get it, it's not something usable for us. We have to find a way to convert this stack trace into a "Chapel stack trace".

The solution I've thought is doing this work modularly in these passages:

1. We get the current stack trace
2. We convert this stack trace into the "native" names (the ones from our precedent example)
3. We convert those names into their Chapel names using a "Chapel symbol table".

The third step adds to the application a table with the relation between a Chapel name and its "real name". This way, we can convert the stack trace with the real names in a Chapel stack trace ! Taking the source code of the precedent examples, we'll obtain

{% highlight shell %}
halt
fact
fact
fact
main
{% endhighlight %}

So, everything is done, right ? Well, no: there are several questions that need to be answered. For example, how this table should be implemented ? We want something that is quick to access and, at the same time, something that doesn't take a lot of space in the application. Also, the entire process should be as modular and machine independent as possible. This is only a beginning and a lot of work is needed before this feature is ready.