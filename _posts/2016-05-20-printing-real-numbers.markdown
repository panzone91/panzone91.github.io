---
layout: post
title:  "Printing real numbers"
date:   2016-05-20 15:05:00 +0200
categories: gsoc chapel
---

I'm spending my Community Bonding period working on a few bugfixes on [Chapel](http://chapel.cray.com). This post wants to describe the bug I've solved during this week. <!--more-->

The purpose of the Community Bonding period during the [Google Summer Of Code](https://summerofcode.withgoogle.com) is to give the student some time for learn the project's rules and what is a best way to learn how you should work with your organization than actually work with them on some little patches ? The student this way can see a part of the codebase of the project and learn about the codestyle and the submission process used by the community.

Now, about the bug. Let's take this Chapel program

{% highlight chapel %}
var pi = 314159.2654;
writeln(pi);
{% endhighlight %}

and its output

{% highlight shell %}
314159.0
{% endhighlight %}

Mmm, not good. The entire decimal part is vanished from the output. Why ?

By default, Chapel doesn't print all the digits in a real number for avoiding numbers too big to read on the output: by default Chapel prints a decimal representation of the number if it can be written in 6 digits. This is our problem: in our example the integer part of the number gets all the digits. However, since every real number in Chapel is printed with the decimal part, for this kinds of numbers Chapel always prints the ".0".

This output can be confusing, since it seems that the number doesn't have a decimal part (the number is an integer). We are already printing a digit of the decimal part (the ".0"), so why not print the first decimal digit ? It seems logic. However, printing the first decimal digit in our case presents a little problem: now we are using 7 digits for some number and 6 for others. Not a great solution.

The solution is to use the exponential notation: if a number has 6 digits in its integer part, the patch will convert it in exponential form, remove the trailing zeroes and print the number. The most difficult part during development was the removing of the trailing zeroes. I love C, as you can see from my other projects, but its string manipulation is terrible.

This change brings a little side-effect: it can break the tests even when there isn't an error. Normally every project is tested with some "case input" and their expected result for being sure that a new patch doesn't reintroduce previously solved problem. But changing how some numbers are printed can change the expected output of some programs ! For example, take this little test

{% highlight chapel %}
var x :real = 159265.0;
writeln("x is: ", x);
{% endhighlight %}

{% highlight shell %}
cat expected-output
x is: 159265.0

./chapel-test
x is: 1.59265e+05
{% endhighlight %}

Both representations are correct and equivalent, but now the test is failing even if the application is correct because it expects the decimal representation of the number. This means that the expected output should be updated. I checked every failed test manually, seeing if their new output in exponential form is correct and updating the expected output. Sometimes it was easy (like in the example above), sometimes it was a little more difficult (like updating a script in perl, since I've never wrote perl code before). After this and the inclusion of a new test for avoiding regression the patch was ready.

You can see the patch [here](https://github.com/chapel-lang/chapel/commit/96428e146c55e36df61cbbac09c4d97d80a35509).