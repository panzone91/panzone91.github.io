---
layout: post
title:  "ELF and DWARF"
date:   2016-07-15 10:35:01 +0200
categories: gsoc chapel
---

Yes, it's since May that I wanted to do this pun. This week I'll talk briefly about ELF (Executable Linking Format) and DWARF (Debugging With Attributed Record Formats).<!--more--> 

But why? The objective of the next part of my GSOC project is to improve the Chapel stack trace by adding line numbers, so I need some informations about the relation between addresses and line number. Let's take one step at the time.

For getting the line number I have to understand where, in each function, I am. Fortunately this is easy: since we have to return to the caller function, the machine has to save the return address. With this address we know where, in that function, we are: at the instruction precedent the return address! So we have our list of instructions and, with some simple operations, we can understand where we are. Easy, right?

{% highlight chapel %}
proc main(){
  fact(2);
}

proc fact(i : int) :int{
  if(i == 0) then halt();
  else return i*fact(i-1);
}
{% endhighlight %}

{% highlight shell %}
prova.chpl:6: error: halt reached
Stacktrace

halt() at $CHPL_HOME/modules/internal/ChapelIO.chpl:679 + 38 (0x423407)
fact() at prova.chpl:5 + 44 (0x426088)
fact() at prova.chpl:5 + 70 (0x4260a2)
fact() at prova.chpl:5 + 70 (0x4260a2)
{% endhighlight %}

As we can see, no. Our fact() doesn't even have 44 lines! Why we're getting this result? Because a single code line can be translated into several machine instructions. We can't match our addresses to line number this way but we need some extra informations if we want to provide a better mapping between the application and its source code. We've already done the same thing with the first prototype to the stack trace project: we had created a table with some debugging informations. And here it's where DWARF comes in play.

DWARF is a standard encoding for debugging information in ELF files. The standard is incredibly complex, but you can think to our debugging informations as a set of (compressed) tables. Between these tables there is a table named ".debug_lines" that, in our example, contains:

{% highlight shell %}
prova.chpl                                     1            0x426041
...
prova.chpl                                     6            0x426079
prova.chpl                                     7            0x42608a
...
{% endhighlight %}

Ah ah! This is a giant table which tells us at which addresses the line number x starts. Let's take our first example: remember the address 0x426088 (sorry if you aren't quite used to an hexadecimal representation)? Well, from this table we now know that we are at line 6 since 0x426079 < 0x426088 < 0x42608a!