---
layout: post
published: true
title: Data Confidentiality and Integrity
summary:
---

The new project I am working on is called Data Confidentiality and Integrity
(DCI).  The goal is to protect sensitive data such as private keys, password
lists, and authorization tokens in C/C++ programs.  A motivating example is the
HeartBleed Bug.  Attackers were able to use a buffer overflow to read a servers
private key.  Existing techniques like [stack cookies] [1], [Control Flow
Integrity] [2], and [Code Pointer Integrity] [3].

[1]:http://en.wikipedia.org/wiki/Buffer_overflow_protection#Canaries 
[2]:http://research.microsoft.com/apps/pubs/default.aspx?id=64250
[3]:https://www.usenix.org/conference/osdi14/technical-sessions/presentation/kuznetsov


