---
layout: post
published: false
title: A Cool LLVM Thing - CPP Backend
summary: The CPP backend generates that LLVM API calls that would yield the given IR
---

Have you ever seen some LLVM IR instruction and wondered, "How the heck would
I make that with the LLVM C++ API?"  Wonder no longer with the *llc* C++
backend!

For my research, I frequently want to insert some new code into a module.  If
the new code I want to add is simple enough I can just create it with the LLVM
APIs.  But for more complicated stuff one solution is writing the code that I
want to insert in C, compiling it to LLVM IR.  If I know what API calls would
create this IR, then I can just make those calls inside my pass.  Sometimes I
don't know which API call to make and using the *llc* C++ backend is one way of
figuring it out.


#With Great Power Comes Great Responsibilty

Don't go crazy and copy-paste a big chunk of code from the C++ backend into your pass.
