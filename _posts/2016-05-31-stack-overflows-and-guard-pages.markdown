---
layout: post
title:  "Stack overflows and guard pages"
date:   2016-05-31 16:19:12 +0200
categories: gsoc chapel
---

I finally had some time to write about my first contribution to Chapel, during the selection phase of the GSOC.<!--more-->

Before starting to explain this patch, I've to spend a couple of words about Chapel's memory allocators. Chapel is an extremely modular language, making it easily portable to different operating systems and architecture. Even memory allocation (when a program try to obtain a memory area to use) is done modularly: Chapel can use several memory allocator algorithms. [Here](http://chapel.cray.com/docs/latest/technotes/allocators.html) there are a list of the supported allocator and more informations about them.

Now, about this patch. I had to continue an [old pull request](https://github.com/chapel-lang/chapel/pull/2149). Chapel, being modular, can use different multithreading libraries. One of the supported library is [pthread](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/pthread.h.html), which is a part of a bigger standard (POSIX). A thread can be seen as a little separate application (that's not completely true, but for our purpose it's sufficient), so a thread has its own call stack. pthread usually creates this stack, however there are situations in which the developer wants that a thread uses a specific memory area as its own stack. Chapel is in one of those situations: we want that all memory is allocated using a specific memory allocator. 

The first pull request implemented the allocation of the stack using the Chapel allocator. My work was to finish this pull request, which means:

1. Update the original pull request
2. Add guard page support
3. Handle the stack deallocation

The first point wasn't difficult: the original pull request was almost a year old and some of its code caused some integration problems with the current Chapel runtime. A couple of manual fixes later and it was ready. The second and third points are more interesting.

# Guard pages

Memory is finite, so what happens when a program fills the call stack ? We have an "overflow" of the stack: the application tries to write on the memory adjacent the stack. This is a problem known as a *stack overflow*. Let's take for example this simple Java program and try to execute it:

{% highlight java %}
public class MainClass{
  public static int fact (int x){
    return x * fact(x+1);
  }

  public static void main(String args[]){
    fact(2);
  }
}
{% endhighlight %}

{% highlight shell %}
java MainClass
Exception in thread "main" java.lang.StackOverflowError
{% endhighlight %}

This program continues to call the same function, filling the stack until at some point the program crashes: the stack was full and the program cannot made another function call.

My work was to implement a way to raise an error if a stack overflow happens in our new pthread stacks. This is important: often the memory adjacent our allocated stack is owned by the application but used for different things. If we don't prevent this, a stack overflow could accidentally modify some important data!

For doing it, I've used the concept of *guard page*. A guard page is a little memory area with the special propriety that cannot be accessed. If we put a guard page at the end of the stack, when the application tries to access it (we have a stack overflow), the guard page will cause an error.

Easy, right ? Well, this is where I've found a major problem. On OSX, when Chapel uses the cstdlib memory allocator, the entire runtime crashed during the creation of some guard pages. It took me a lot of testing and debugging but I've discovered the cause of it: [mprotect](http://pubs.opengroup.org/onlinepubs/9699919799/functions/mprotect.html) (which sets the guard page) requires to work only on mmap allocated memory. This isn't usually a problem, since a lot of implementations uses mmap for implements malloc. Guess what ? Starting from some version of OSX, the system allocator doesn't use mmap!

This is a problem for our stack allocator: the solution in the patch is to doesn't allocate the stack when, on OSX, the runtime uses the cstlib allocator (it uses the pthread provided stack).

# Stack deallocation

As a program can allocate a memory area, it can also deallocate it. Memory deallocation is important: if a memory area is deallocated, it means that that memory area it can be used by another application. Since when a thread finishes the execution the stack isn't used anymore, we should deallocate it. 

The first problem is to know which memory areas are used as stacks. Chapel keeps a linked list with the informations of every thread so it seems a logical idea trying to put this information into that list. However, the thread list is updated by the threads themselves: each thread registered itself on this list as its first operation. As pthread as no (standard) way to obtain the stack address from the thread itself, there wasn't an easy way to add our stack pointer into the thread list without some major changes to the thread implementation.

My solution was to create a new linked list, a stack list, with each stack allocated. When a thread finishes its work, the runtime can search this list, obtain the stack pointer and deallocate the area.

If you look at the patch, you can see that the stacks are freed at the end of the execution, when a general join of all threads is executed (since pthread creation is expensive, Chapel created them only one time). However, think about this situation:

- Thread A starts the thread join
- Thread B allocates the call stack, registers it in the stack list and calls pthread_create, creating thread C.
- Thread A finish the thread join (thread B stops) and start the stack deallocation.

This presents the possibility of a race condition: the stack for C is in the stack list, but it's possible that C isn't started yet (so it isn't registered in the thread list and it can't be joined) because, for specification, you can't assume that a thread is immediately created after the call to pthread_create. This means that the stack for C can be deallocated before C even starts! For this reason, the final version of the patch deallocate a stack only of the joined thread.

This patch took me a couple of weeks, but it is now merged with in the main branch ([here](https://github.com/chapel-lang/chapel/commit/937f539185bb17d763d56c0bce9d1de40575a10d)). I've learned so much during its development (like the race condition in the stack deallocation) and I'm proud of my little contribution.