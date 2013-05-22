---
published: true
layout: post
title: Emacs vs. Vim
summary: And the art of text editing
---

Around the middle of 2011, I started learning Vim.  Right away I noticed productivity gains simply from keeping my fingers on the home row (and off the mouse).  I followed what I imagine is the inevitable path.  At first I installed every plugin I could find, but eventually their allure wore off.  I migrated back to default Vim.  Most plugins offer something that is in default Vim but in a nicer way.  But the plugins go against the main selling point of Vim -- it's installed on every server.  Reconfiguring the plugins became a pain because I ended working on many different machines, so I dropped them.

My honeymoon phase had ended and I decided to try Emacs, mostly for fun (and at the suggestion of [Matt Might](http://matt.might.net/articles/grad-student-resolutions/ ) 

## To Mode or not to Mode ##

The big differentiator between Vim and pretty much every other text editor (including Emacs) is modes.  There's a whole bunch of modes in Vim, some of them I don't even understand after using it for two years.  It basically comes down to: in Vim sometimes pushing 'j' types the letter into my file, but in Emacs it always does.  This has profound implications.  In Emacs, every time I want to do a command I need to hold down a modifier key (control, alt, etc).  In Vim I only need modifier keys to switch modes.

There is a preoccupation with "how many keys do I need to press to accomplish x," but I'll indulge that idea briefly.  Often times Vim might result in fewer total key strokes.  I can switch to a mode, do what I want to do, and switch back.  In Emacs I might have to hold down some crazy key combination like "Control-Alt-Meta-whatever" and do that a bunch of times.  Other times I've found Emacs requires fewer key strokes, like if the operation I want to do in Vim would require me to leave a mode, switch to another, exit that mode, and then save.  The keystrokes changing modes don't manifest changes in the file, so they're wasted.

## Emacs Pinky ##

[Emacs Pinky](http://en.wikipedia.org/wiki/GNU_Emacs#Emacs_pinky ) is a special type of repetitive strain injury caused by holding down the Control or Meta key (often pressed by the pinky finger) for extended periods.  Having experienced RSI (mildly, thank $DEITY) I can imagine any severe case must be life changing for a programmer.  Our job is to type, our hands and keyboards our instrument.  

That said, Emacs pinky can be mitigated if not eliminated.  Besides, no matter if I use Vim or Emacs, I'm going to end up hitting modifier keys at some point.  There are two factors:

1. Frequency a key is pressed with the pinky
2. Duration key is held down

[Stick Keys](http://en.wikipedia.org/wiki/StickyKeys ) can almost eliminate #2. For #1, I think it's unavoidable that I'd end up pressing more modifier keys in Emacs.  Remapping the most used keys to be under my thumbs seems like a decent mitigation.

## Too Much of a Good Thing ##

As my comments in the beginning about plugins hinted at, there's a pitfall of spending too much time improving your productivity and never actually producing anything.  I think it's worth trying out different editors once and a while but it can be a time sink.

One of the main reasons I decided to try Emacs was I was too into Vim.  I was obsessed with editing really fast.  I'd gotten to the point where most Vim commands were in my muscle memory and I could just bang out code.  There was a side effect though.  I noticed I was thinking less and just churning the compile-test-edit loop faster and faster.  Getting through the loop faster is nice but it can border on "coding by randomly changing things until it works."

So as of this moment, I'm trying Emacs for a change of pace and I haven't used it long enough to pick a side in the Vim vs. Emacs holy war.  Preliminarily, it seems like more powerful plugins are possible in Emacs, and I've found its customizability to be nicer.
