---
layout: post
title:  "Regex and stack trace format"
date:   2016-07-08 14:15:42 +0200
categories: gsoc chapel
---

This week I've implemented the new stack trace format (discussed with the community in the Chapel developer mailing list) and, with it, I finally updated the test suite with the support to stack traces.<!--more-->

The second point is important: a lot of existing tests uses halt(), but their .good files doesn't present the stack trace. Also, not always the runtime is builded with the stack trace support. I also couldn't simply add a stack trace in every test that use halt(): there are a lot of tests to update! The solution is only one: write a script!

In this post I'll use the notation that I've used in [this post about the testing system in Chapel]({% post_url 2016-06-24-testing-chapel %}), so if you didn't read it I suggest to spend some time on it. The objective of this script is easy:

1. See if the test is a "special one", a test that possess a stack trace in its .good file. If the test is one of these, do nothing and exit (since in this case we want to test the stack trace). 
1. Execute the test and get its output
2. If the output possess a stack trace, remove it from the output.

The first and third point require to basically understand if a string (the output of the test or the content of the .good file) possess a stack trace and, in case, delete it. But how we can understand that? 

I used regular expressions. There is a lot to write about regular expressions (or regex), so I prefer to let [Wikipedia](https://en.wikipedia.org/wiki/Regular_expression) explains them. The main idea is that we have a string that can be matched, following some specific rules, to some parts of a general text. It seems perfect for us: we "match" the stack trace in the output/.good file and, if we find it, we remove it. Now we just have to find the regular expression (the pattern string) that matches all our Chapel stack traces.

Let's take an example of a stack trace:

{% highlight shell %}
Stacktrace

halt() at $CHPL_HOME/modules/internal/ChapelIO.chpl:679
fact() at prova.chpl:5
fact() at prova.chpl:5
fact() at prova.chpl:5
{% endhighlight %}

Our objective is to find a regex that can match this text. It's better to divide our work into two different phases:

1. The "prologue" (with the "Stacktrace"). This is a simple regex , since it's simple "Stacktrace"
2. The "function lines" (for example "fact() at prova.chpl:5"). The patter seems to be something like the procedures's name, the double parenthesis, a space, the string "at", another space, a string (the path), the character ":" and a number. A regex that can match this is:

{% highlight shell %}
[[a-zA-Z0-9]+\(\) at .*:.*]*
{% endhighlight %}

(Yes, I can probably do a little better, but for the moment it's seems enough).

With these regex, now it's easy to write a little script to remove from the output of a test a stack trace:

{% highlight shell %}
# This will remove all the function lines
grep -a -v -E "[[a-zA-Z0-9]+\(\) at .*:.*]*" $outfile > $outfile.tmp
# This will remove the stack trace prologue
perl -i -pe 'BEGIN{undef $/;} s/\nStacktrace\n\n//smg' $outfile.tmp
mv $outfile.tmp $outfile
{% endhighlight %}

You can find the complete script in my [pull request](https://github.com/chapel-lang/chapel/pull/4125).

# The future

These two last weeks was pretty light, especially considerate my first few weeks during this GSOC. Now however I've to start the next phase of my project: extend the stack trace with line numbers. This seems easy, however this problem presents a lot of difficulties. For example:

1. Even if we obtain the current PC at the moment of the procedure (doing a stack traversal), we don't have a link between number of instructions and compiled lines.
2. A single Chapel line is compiled into several native lines. Also, using the debugging informations for obtain the native line number doesn't give us a lot of informations about Chapel lines
3. Chapel doesn't (always) control how the native compiler does function inlining.

And these are only the first things I can think of. Next week I'll necessary do some brain-storming for seeing all the possible options for this project.