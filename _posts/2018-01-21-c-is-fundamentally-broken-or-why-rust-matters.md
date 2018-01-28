---
layout: post
published: true
title: "Work in Progress: C is Fundamentally Insecure, or Why Rust Matters"
summary: A point made not often enough -- until people stop writing new C code
---

```
THIS POST IS A WORK IN PROGRESS
```

My PhD thesis topic was essentially, "How to retroactively make C programs safe."  I spent three years working on this problem.  Now, I'm convinced we should stop writing C code all together, and instead write new code in a language designed for safety (ex: Rust).

To be clear, it's impossible for me to "prove" to you that C is insecure -- in the sense of mathemtical or logic proof.  Instead, I will make a qualitative argument that the path of least resistance in C is writing buggy and insecure programs.  To write secure C code is difficult and expensive.  Secure C code is written *inspite of* C's underlying insecurity, not *because of* any positive attribute of the language.

But this article will not be all doom and gloom.  Many smart people have been working hard developing tools to build secure systems, and The Way Forward is to put these tools to the test and write production quality software. 

# Secure Systems

I will make my argument against C, and for a new safer language, concrete in the rest of the article using examples.  But before jumping into that, I'll give an informal definition of systems security.  We want the following properties in our systems:

* P1: unexpected input should not crash our system
* P2: attackers should not be able to craft inputs that allow them to hijack our system
* P3: my system should not leak unintended data
* P4: attackers should not able to corrupt our system's protected data

The following examples illustrate that it's much easier to write C programs, that violate these properties, than it is to write secure C programs.

# A standard library function that impossible to use securely: `gets`

The `gets` function reads characters from `stdin` into a buffer until a newline character is read.  Its signature is `char* gets(char* str)`.

Calling `gets` securely is impossible!  No matter the size of the `str` buffer, an attacker can overflow it with an arbitrary amount of non-newline characters.  Every program that uses `gets` has a buffer overflow.  Luckily the C standard authors were smart and removed this from the C standard way back in ... 2011!

Joking aside, the `gets` function is poorly designed and a historical mistake.  Modern compilers/libraries (hopefully) warn you if you compile/run a program that calls `gets`, but because of the C community's obsession with backwards compatibility, I can write/compile/run/publish a C program in 2018 that uses a function that is known to be impossible to call securely.

# The many implications of `char*`

The `gets` function is an easy target to pick on, and yes, it has been removed from the C standard, but the underlying reason why `gets` is insecure still exists in essentially every C program.  The problem is that the C type system simply isn't expressive enough to safely use and implement APIs.  For example, let's look at the function `strcpy`.  It has the signature `char* strcpy(char* dst, const char* src)`.  Its specified behavior is that `src` is copied to `dst`.  No possible implementation of `strcpy` can ensure that the security properties mentioned above hold.  For example, if either `src` or `dst` are null, the program crashes, violating P1.  If the `src` string is too long, it causes a buffer overflow, which can violate P1-4.  In fact, a call to strcpy is only safe if:

1. Both `src` and `dst` point to allocated memory
2. The data pointed to be `src` is a C string of length `n`
3. The memory `dst` points to is big enough to hold `n` characters

Yet, the implementer of `strcpy` has no guarantee any of those hold.

A programming language can help us write secure programs by making the conditions that must be hold for secure execution explicit, and by checking if those conditions hold statically or dynamically.  Rust, and even C++, have references which cannot be null, solving item 1 above.  Items 2 and 3 are hard to ensure statically, but we can simply give every string buffer a length field and check it at runtime.  This can be done with vectors in C++ or Rust for example.  Sure this has a little runtime overhead, but we mitigate a whole category of problems.

# Valid values as error codes

Every C programmer has seen and written code like:

```
if (pointer) {
   // some code that deferences pointer
}
```

The `if` ensures `pointer` is not `null`, which is a necessary condition, but not a sufficient condition for the `if` body to execute correctly.  In fact, inside the `if` body, `pointer` can have any possible value but `0`.  If `pointer` is 64 bits, we still have `2^64-1` possible values.  The *vast* majority of those values crash the program (or do something else bad).  In the end, `null` is just some arbitrary value we picked to respresent an invalid pointer.  C doesn't even automatically initialize pointers to 0.

A better example of 'valid values as error codes' is the standard library function `atoi`.  The signature of `atoi` is `int atoi(const char* str)` and its behavior is it converts `str` to an integer.  What happens when `str` doesn't represent any integer?  For example, what is `atoi("banana")`?   Should `atoi` return o (or `null`)?  Obviously not, because `atoi("0")` returns the same thing.  It's impossible for `atoi` to indicate to its caller that it failed.  We can solve this issue by having a special value that is truly never valid like Python's `None`.  In Rust we use `None` with `Option<T>`, which allows the type system to enforce that the caller checked for a bad result.  Let's say I want to write `atoi` in Rust.  I'll give it the signature `Option<int> atoi(s: &str)`.  Now, because of Rust's type system, I know that:

1. The parameter `s` will never be null
2. Callers of my `atoi` function are forced, by the Rust type system, to implement code that handles both of the following cases: 
  - My `atoi` returns `None` -- which obviously indicates an error
  - My `atoi` returns `Some(a_number)` -- which means the answer is `a_number`

`Option<T>` avoids the entire confusion about which values indicate success and which indicate an error, and as a bonus ensures that callers have to handle those cases.

# Whose memory is it anyway?

TODO: `getaddrinfo` example

a surprising number of libc functions allocate memory