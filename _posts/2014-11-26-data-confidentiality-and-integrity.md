---
layout: post
published: true
title: Data Confidentiality and Integrity
summary: My new project for preventing information leaks and data corruption in C/C++ programs
---

The new project I am working on is called Data Confidentiality and Integrity
(DCI).  The goal is to protect sensitive data such as private keys, password
lists, and authorization tokens in C/C++ programs.  A motivating example is the
HeartBleed Bug.  Attackers were able to use a buffer overflow to read a servers
private key.  Existing techniques like [stack cookies] [1], [Control Flow
Integrity] [2], and [Code Pointer Integrity] [3] would not prevent this type of
attack.

The root of most exploits in C/C++ programs is memory corruption.  Somehow the
attacker gets a pointer out of bounds and reads or writes addresses the
programmer never intended.  DCI's protection mechanism will protect these out of
bounds reads and writes for a programmer selected subset of all the variables in
the program.

[1]:http://en.wikipedia.org/wiki/Buffer_overflow_protection#Canaries 
[2]:http://research.microsoft.com/apps/pubs/default.aspx?id=64250
[3]:https://www.usenix.org/conference/osdi14/technical-sessions/presentation/kuznetsov


