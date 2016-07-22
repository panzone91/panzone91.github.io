---
layout: post
title:  "Little pull requests"
date:   2016-07-22 10:15:01 +0200
categories: gsoc chapel
---

This week was a week of little pull requests with little improvements and cleanup.<!--more--> However, even this kind of little tweaks are useful in an open source project: it's necessary sometimes to stop and cleanup your code base for making future work easier to do. For this post I've took two PR that I've developed this week.

* [Print an error if CHPL_UNWIND = libunwind on darwin](https://github.com/chapel-lang/chapel/pull/4174)

If you followed my precedent posts ([here]({% post_url 2016-06-03-prototypes %}) and [here]({% post_url 2016-06-17-Build-and-extend-Chapel-runtime %})), you know that an user can activate stack tracing in Chapel using the variable CHPL_UNWIND. This variable has two settings: "libunwind" if you want to use the provided libunwind library or "system" if you want to use your system provided unwind library. The provided libunwind library doesn't support Mac OSX (or I should say macOS?) so CHPL_UNWIND=libunwind on macOS isn't valid, but there wasn't any specific error message about it. This PR solves this.

* [Add libunwind as a third party package](https://github.com/chapel-lang/chapel/pull/4183)

This PR simply adds [this](http://www.nongnu.org/libunwind/) implementation of libunwind as the libunwind library provided by Chapel. Until now Chapel didn't provided a libunwind and required the user to manually download and put the library in his/her Chapel installation, but this can be a little trick. This PR also includes some little corrections and changes to the building system (I've described [here]({% post_url 2016-06-17-Build-and-extend-Chapel-runtime %}) the difficulties to the Chapel building system) to support the new tarball.