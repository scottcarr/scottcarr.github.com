---
layout: post
published: false
title: C is Fundamentally Memory Unsafe, or Why Rust Matters
summary: A point made not often enough -- until people stop writing new C code
---

My PhD thesis topic is essentially, "How to retroactively make C programs safe."  I spent three year working about this problem.  Now, I'm convinced we should stop writing C code all together, and instead write new code in a language designed for safety (ex: Rust).

To be clear, it's impossible for me to "prove" to you that C is insecure -- in the sense of mathemtical or logic proof.  Instead, I will make a qualitative argument that the easy path in C is writing buggy and insecure programs.  Secure C code is difficult and expensive to write.  Secure C code is written *inspite of* C's underlying insecurity, not *because of* any positive attribute of the language. 

# Secure Systems

I will make my argument concrete in the rest of the article using examples.  But before jumping into that, I'll give an informal definition of systems security.  We want the following properties in our systems:

* P1: unexpected input should not crash my system
* P2: attackers should not be able to craft inputs that allow them to hijack my system
* P3: my system should not leak unintended data
* P4: attackers should not able to corrupt my system's protected data

The following examples illustrate that, it's much easier to write C programs that violate these properties than it is to write secure C programs.

# Fundamentally Bad Idea 1: `gets`

The `gets` function reads characters from `stdin` into a buffer until a newline character is read.  It's signature is `char* gets(char* str)`.

Calling `gets` securely is impossible!  No matter the size of the `str` buffer, an attacker can overflow it with an arbitrary amount of non-newline characters.  Every program that uses `gets` has a buffer overflow.  Luckily the C standard authors were smart and removed this from the C standard way back in ... 2011!

Joking aside, the `gets` function is horribly designed and a historical mistake.  Modern systems (hopefully) warn you if you compile/run a program that calls gets, but because of the C community's obsession with backwards compatibility, I can write/compile/run/publish a C program in 2018 that uses a function that is known to be impossible to call securely.

# Fundamentally Bad Idea 2: Unsized Buffers

...