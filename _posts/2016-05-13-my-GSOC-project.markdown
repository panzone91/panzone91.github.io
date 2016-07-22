---
layout: post
title:  "My GSOC project"
date:   2016-05-13 15:15:00 +0200
categories: gsoc chapel
---

This will be the second time I'm partecipating at the [Google Summer Of Code](https://summerofcode.withgoogle.com). This time I'll work on [Chapel](http://chapel.cray.com), a programming language with a focus on parallel development.<!--more--> Citing [Chapel's github page](https://github.com/chapel-lang/chapel), Chapel is "a Productive Parallel Programming Language".

More precisely, I'll spend the summer working on the Chapel's runtime. [My project](https://summerofcode.withgoogle.com/projects/#6738560711393280) is to implement some sort of backtrace when an unrecoverable error is encountered. A backtrace is a list of functions in the call stack at the moment of the error: this can help developers during the debugging phase. For example, this is the backtrace of a simple program:

{% highlight shell %}
test`foo  + 22
test`bar  + 19
test`main + 18
{% endhighlight %}

Looking at the backtrace we can know where the error is encountered (in the example, we now know that the application crash in the function foo called by bar).

I didn't know anything about Chapel before March 2016 but during this semester I'm following a parallel algorithms class: since Chapel is thought for parallel programming, it seems a good idea learn it for implement some of the algorithms I was going to see in class. Take that, my passion for low-level development and the result is clear: I have to try to apply under Chapel for this GSOC.

My first contribution to Chapel ([PR](https://github.com/chapel-lang/chapel/pull/3729)) it's something I want to explain in more details in a separate post. It needed a lot of work and refinements before it was ready but it was great to see my work merged.

I'm currently in the "Community Bonding period", working with my mentor to a couple of bugfixes and deciding the best way to implement my "main" GSOC project. We already had a couple of good ideas that need more improvements, but I think it's coming good.

Also I'm learning Chapel as a language. It's a nice language, even if a little rough around the edges (like every young language). I've experience with multithread programming but Chapel permits to ignore almost all the implementation details and to concentrate on the application itself. The syntax is like the PRAM pseudocode we're using in my parallel algorithms class. For example, this is the pseudocode for the summation of the values in an array:

{% highlight chapel %}
for( j = 1; j < log(n))
	for( k = 1; k < n/2) par do
		M[(2^j)k] = M[(2^j)k] + M[(2^j)k - (2^j-1)]
{% endhighlight %}

And this is my implementation of it in Chapel:

{% highlight chapel %}
proc summation(M){
var n = M.numElements;
	
for j in 1 .. log2(n) do {
	coforall k in 1 .. n/(2**j) {
		var ind = 2**(j) * k;
		M[ind] = M[ind] + M[ind - 2**(j-1)];
	}
}
return M[n];
}
{% endhighlight %}

As you can see, to implement the pseudocode in Chapel is quite easy.