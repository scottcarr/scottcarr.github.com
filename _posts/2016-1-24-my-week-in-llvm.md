---
layout: post
published: true
title: My Week in LLVM - Clone a Function
summary: This blog post brought to you by the function - CloneFunctionInto
---

For my current [research project](data-confidentiality-and-integrity/), I'm
using LLVM to harden C programs.  One aspect of the project is to partition the
program data into two disjoint sets.  To enforce isolation between the disjoint
sets, I add some instrumentation to every function.  The instrumentation is
different depending on which set the function uses, but there are some cases
where I want two versions of a given function (one for each of the two sets).
This leads to my problem of the week: how to clone a function in LLVM.

In this post, I'm not just going to jump to the answer.  Instead I'll give some
of the steps followed to give an example the might help some people who are
newer to LLVM. 

#Step 1: Google It

Whenever I face a problem with LLVM, the first thing I do is Google it.  Most
of the time, there isn't a StackOverflow question or a blog post that perfectly
fits my problem, but sometimes there is.  Always try Google.  You might get
lucky!

Here is what I get when I Google "llvm clone function":

![a screen shot](https://github.com/scottcarr/scottcarr.github.com/raw/master/images/google_clone_function.png)

In the first result, I found a function called [CloneFunction](http://llvm.org/docs/doxygen/html/CloneFunction_8cpp_source.html#l00078).  It looks pretty promising:

{% highlight cpp%}
/// Return a copy of the specified function, but without
/// embedding the function into another module.  Also, any references specified
/// in the VMap are changed to refer to their mapped value instead of the
/// original one.  If any of the arguments to the function are in the VMap,
/// the arguments are deleted from the resultant function.  The VMap is
/// updated to include mappings from all of the instructions and basicblocks in
/// the function from their old to new values.
///
Function *llvm::CloneFunction(const Function *F, ValueToValueMapTy &VMap,
                              bool ModuleLevelChanges,
                              ClonedCodeInfo *CodeInfo)
{% endhighlight%}

There are a couple issues I see, though:

1. the comment says "... without embeding the function into another module."  but i definitely want my function to be embedded in this module.
2. What the heck is a *ValueToValueMapTy*?
3. What the heck is a *ClonedCodeInfo*?

Let's put 2 and 3 aside for now.  How do I get my function to be cloned into
this module?  I Googled and grepped the LLVM source, but couldn't find any
function that cloned a function and took a *Module* reference as a parameter.
Then I found
[CloneFunctionInto](http://llvm.org/docs/doxygen/html/Cloning_8h_source.html#l00142)
which takes a new function to copy-paste the original function's code into.

{% highlight cpp%}
/// Clone OldFunc into NewFunc, transforming the old arguments into references
/// to VMap values.  Note that if NewFunc already has basic blocks, the ones
/// cloned into it will be added to the end of the function.  This function
/// fills in a list of return instructions, and can optionally remap types
/// and/or append the specified suffix to all values cloned.
///
/// If ModuleLevelChanges is false, VMap contains no non-identity GlobalValue
/// mappings.
///
void CloneFunctionInto(Function *NewFunc, const Function *OldFunc,
                       ValueToValueMapTy &VMap, bool ModuleLevelChanges,
                       SmallVectorImpl<ReturnInst*> &Returns,
                       const char *NameSuffix = "",
                       ClonedCodeInfo *CodeInfo = nullptr,
                       ValueMapTypeRemapper *TypeMapper = nullptr,
                       ValueMaterializer *Materializer = nullptr);
{% endhighlight%}

Using this, I can clone one function into another, so I just need to make sure
*newFunc* is a function in the current module.

#Step 2: Grep for It

At this point I'm pretty sure I want to use
[CloneFunctionInto](http://llvm.org/docs/doxygen/html/CloneFunction_8cpp_source.html#l00078),
but I don't know what a *ValueToValueMapTy* is.  For *CloneCodeInfo* it takes a
default value of *nullptr*, so I'm just going with that.  To figure out how to
use any LLVM type, my go to tactic is to find somewhere in the LLVM that uses
that type and adapt it for my purposes.

Unfortunately *ValueToValueMap* is used in a lot of places.  I had the bright idea that cloning a function is kind of like inlining a function into an empty function, so I looked in InlineFunction.cpp.  I found this [snippet](http://llvm.org/docs/doxygen/html/InlineFunction_8cpp_source.html#l01434):

{% highlight cpp%}
ValueToValueMapTy VMap;
// Keep a list of pair (dst, src) to emit byval initializations.
SmallVector<std::pair<Value*, Value*>, 4> ByValInit;

auto &DL = Caller->getParent()->getDataLayout();

assert(CalledFunc->arg_size() == CS.arg_size() &&
       "No varargs calls can be inlined!");

// Calculate the vector of arguments to pass into the function cloner, which
// matches up the formal to the actual argument values.
CallSite::arg_iterator AI = CS.arg_begin();
unsigned ArgNo = 0;
for (Function::const_arg_iterator I = CalledFunc->arg_begin(),
     E = CalledFunc->arg_end(); I != E; ++I, ++AI, ++ArgNo) {
  Value *ActualArg = *AI;

  // When byval arguments actually inlined, we need to make the copy implied
  // by them explicit.  However, we don't do this if the callee is readonly
  // or readnone, because the copy would be unneeded: the callee doesn't
  // modify the struct.
  if (CS.isByValArgument(ArgNo)) {
    ActualArg = HandleByValArgument(ActualArg, TheCall, CalledFunc, IFI,
                                    CalledFunc->getParamAlignment(ArgNo+1));
    if (ActualArg != *AI)
      ByValInit.push_back(std::make_pair(ActualArg, (Value*) *AI));
  }

  VMap[&*I] = ActualArg;
}
{% endhighlight%}

This is what I'm looking for, but instead of mapping the arguments passed to
the function to the function's formal parameters, I want to map my new
function's formal parameters to my original function's formal parameters.

#Step 3: Write My Solution

Now I'm ready to write my solution.

I ensured that my new function was in this module by creating my a new function using
[Module::getOrInsertionFunction](http://llvm.org/docs/doxygen/html/Module_8cpp_source.html#l00141).
I gave the new function has the same type as the original with "_cloned" added to the end.

Then I have a for-loop other both my new function's arguments and the old
functions and I map them to each other with a *ValueToValueMapTy* instance.

I set *ModuleLevelChanges* to true, even thought I'm not 100% sure what that does.

For the *Returns* parameter I made a new vector of return instructions using:

{% highlight cpp%}
SmallVector<ReturnInst*, 8> returns;
{% endhighlight%}

Currently, I do not store this vector anywhere, but I should.  I'll need to
find any the return instructions later in my pass, but that's a job for another
day.

For the *NameSuffix* parameter, I passed "_cloned" just so I can always tell if the values are from a cloned function or not.  In the future I should use a smarter suffix.

For the other parameters, I just left the default value.

#Conclusion

That's how I clone functions in LLVM.  The full code will be open sourced as soon as my paper is accepted :).

As always, tips and corrections are welcome.  You can contact me on Twitter.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">My new blog post on how I solved a problem in LLVM this week: <a href="https://t.co/nt1a3T4DYQ">https://t.co/nt1a3T4DYQ</a></p>&mdash; Scott Carr (@ScottCarr) <a href="https://twitter.com/ScottCarr/status/691354475815112704">January 24, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>



