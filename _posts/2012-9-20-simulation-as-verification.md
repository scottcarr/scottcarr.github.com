---
layout: post
published: true
title: Simulation as verification
summary: We need simulation as part of the research process. To what extent does it replace testing on a real system?
---

To get up to speed on the current trend of computer science research, I've
been reading 5-10 papers a week.  Most of these are related to embedded systems.

In my tiny sampling, simulation results dominate most papers' evaluation section.
(Off topic: evaluation sections make up a surprisingly small portion of the papers.)
Building real system and carrying out meaningful validation is very difficult.
Perhaps the trend towards simulation is inevitable.

The example I'm thinking of is energy consumption for embedded
devices.  Doing a real-world test requires physically building the circuit,
running some representative benchmark, and measuring the energy consumption
with some sensor.  That's a single data point.  If the test is rerun
will the result be the same?  Does the result depend on uncontrolled factors
(ambient temperature in the room, how long the system has been powered on, etc.?)
This is especially relevant in wireless sensor networks (WSN).  WSN may be
deployed in any range of temperature enviroments and components do degrade over
time. These uncontrolled factors will probably dampen the effect size of the proposed
improvement.

Maybe this is missing the point.  What really matters is: does the paper present a 
good, novel idea?  Researchers aren't test engineers, so of course they should
be focusing more effort on the idea phase than the testing phase.  But, if I were
going to develop a product based on some published research, I'd be sure
to mentally widen the error bars a little (or recreate my own validation).
