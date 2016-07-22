---
layout: post
title:  "Refactoring"
date:   2016-06-10 14:59:09 +0200
categories: gsoc chapel opensource
---

My main work this week was refactoring my prototype so if you expected some great work or article like the past weeks, you're going to be disappointed.<!--more-->

What's refactoring? It's the procedure to change some code for making it more easy to read or to extend in the future. In open source, refactoring is one of the most important and at the same time underrated aspect of development: since you aren't introduce new shiny features, the users usually doesn't even notice it. However, having a clear source code is important for the developer that one day will have to work on it: it will make his/her job more easier and bugs correction more efficient. It's the kind of work that nobody wants to do, since everyone wants to work on new and interesting features, but it's necessary for the entire project.

So, after this excessive long introduction to refactoring, let's talk about my work: other than some typo corrections and functions encapsulation, the main thing about this week's refactoring is a little improvement to our table of conversion between Chapel symbols and native symbols. Our table was a simple array with entry in this format:

{% highlight c %}
Chapel Symbol, Native Symbol, File name, Line number
{% endhighlight %}

Symbols are saved as a string (a sequence of characters), so this means that a symbol of length 5 takes 6 bytes (because of the string terminator) in our application. This can be acceptable for function names, but in our table we also save the complete file name! For example, this is a copy of an halt entry in a little example:

{% highlight c %}
"halt", "halt", 
"/Users/panzone/workspace/chapel/modules/internal/ChapelIO.chpl", "679"
{% endhighlight %}

The complete filename is 62 characters long and this entire entry takes 77 bytes! Also, a lot of file name entries are the same (think about a module, which contains a lot of functions). That's an incredible waste of memory.

The solution is actually simple: make a separate table with the file name (without repetitions) and put the correspondent index in our symbol table. Fortunately Chapel already did the filename table (for handling module dependencies) so with a relativity simple patch I've seen a reduction of almost 40% in executable size!

During this week I've also worked on the Chapel building system but I'll talk about it in another occasion in the future.