---
layout: post
published: true
title: "Problems with Writing Safe C and Rust Mitigates them"
summary: Even if you don't use Rust, understanding these issues will make you a better C programmer.
---

```
THIS POST IS A WORK IN PROGRESS
```

My PhD thesis topic was essentially, "How to retroactively make C programs safe."  I spent three years working on this problem.  The longer I worked on C tooling, the more I realized the C ecosystem is a bunch of interdependent hacks.  Did you know glibc can only be built with gcc, and no other C compiler?  I'll refer to the C language specification, standard library and compiler collectively as "C."

Unfortunately, decades of iteration on C tooling have not added up to secure C.  Some problems have existed since C's invention in the 1970s.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">All the amazing crypto attacks in the world, and real security still comes down to someone screwing up memcpy().</p>&mdash; Matthew Green (@matthew_d_green) <a href="https://twitter.com/matthew_d_green/status/835033668972326913?ref_src=twsrc%5Etfw">February 24, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

The improvements that have been made are mitigations like Stack Cookies, DEP, and ASLR. The thinking behind these tools could be summarized as, "We know there will be vunlerabilities in our C code, so let's try to make exploiting them harder."  

The state of the art in mitigations are sanitizers (AddressSaniziter, MemorySanitizer, ThreadSanitizer, etc).  My own PhD project could be called a [sanitizer.](https://github.com/hexhive/datashield)  Sanitizers add dynamic checks into C programs, then the user does exhaustive testing to find all the bugs -- or so the theory goes.  The lack of static tooling for finding vulnerabilities speaks to how incredibly difficult precise static analysis is for C.

With that in mind, the rest of this post will be examples of the fundamental problems with C that lead to vulnerabilies, and some hints at how Rust deals with them.

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

A quick refresher: C uses manual memory management.  The program requests a number of bytes of memory from the OS, usually by calling `malloc` or similar, and releases a whole chunk of bytes by calling `free` with a pointer to the start of the chunk of bytes.

At first glance, this seems straightforward, but manual memory management becomes very complicated quickly.  Each chunk of allocated memory should not overlap any other chunk and should be freed exactly once.   How does C help you ensure these properties across a large multi-threaded code base?  It doesn't.

If a given API function returns a pointer to its callers, there are 3 possible scenarios -- ignoring the previously discussed `null`/non-`null` issue.  That is, the returned pointer points to:

1. a contiguous chunk of heap memory that the API function allocated
2. a data structure of non-contigous memory
3. some longer lived data that the caller isn't supposed to free

Functions like `malloc`, including `strdup`, fall under 1.  An example of 2 is `getaddrinfo`.  It returns the information in a linked list, so calling `free` on the head of the linked list would leave the rest of the list stranded in memory.  The function `getenv` is an example of 3.

The problem is we cannot differentiate those scenarios based on C's type system.
Compare:

`char* strdup(const char* s1)`

and

`char* getenv(const char* name)`

The types are the same, but the caller is supposed to call `free` on the return value of `strdup` but not `getenv`.  Our C compiler can't help us when we confuse the two.  In contrast, the Rust compiler ensures every variable has only one owner.



