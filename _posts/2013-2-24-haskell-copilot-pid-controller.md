---
layout: post
published: true
title: A PID controller simulation in a Haskell DSL called Copilot
summary: 
---

I stumbled onto a Haskell DSL called [Copilot](http://leepike.github.com/Copilot/) 
for building software health monitors in embedded systems.

The physical part of a cyber-physical system is continuous, so infinite streams
of data a good models for their inputs.  Consider an autopilot in a plane.  It
it might sample an altitude sensor periodically, but it needs to keep doing that for
an unknown amount of time (until the plane lands).  It can't buffer data, it 
needs to handle each sample as it comes.

Copilot makes it simple to use infinite sampled data streams in Haskell.  External
streams represent the sensor (or other) data coming into the monitor.  A spec
is the container around the monitor, the trigger, and the streams.  The trigger
is the function to call when the monitored condition occurs.

The idea of infinite samples streams reminded me of Simulink, so I decided to build
a PID controller simulaton.

Below is my spec:

    impulseResponse :: Spec                                                            
    impulseResponse = do                                                               
      observer "y" (y)                                                                 
      trigger "danger!"                                                                
        (maxErrExceeded) []                                                            

The name of the spec is impulseResponse, my trigger function to call is
called danger! and my monitoring condition is maxErrExceeded.  Observers
are used to monitor the value of the streams.  The have the form:

    observer "description" (variable_name)

Where description can be any string you want.  It appears as the column header
in the output when the spec is interpreted.  I monitor almost all the variables 
in the real version for plotting.

I defined normal float variables for the coefficients of my PID controller:

      p = 0.5   -- P term coefficient
      i = 0.2   -- I term coefficient
      d = 0.2   -- D term coefficient

Now I need to define streams for the requested input (y), the commanded output
(u) and the error (e):

      y :: Stream Double    -- controller input
      u :: Stream Double    -- controller output
      e :: Stream Double    -- error between input and output

For a real PID controller I also need to know the derivative for the error and
the integral:

      integral :: Stream Double     -- integral of the error
      dedt :: Stream Double         -- derivative of the error

Calculating the streams is the fun part.  The input (y) is the most straightfoward.
All I wanted was an impulse, so I made an infinit stream of ones and added
some zeros at the beginning:

      y = [0.0, 0.0, 0.0, 0,0] ++ 1.0 -- input is an impulse

For u, the initial command is 0.0, but the other command need to be calculated:

      u = [0.0] ++ command -- the PID controller output

Error is just input minus output

      e = y - u -- the error stream

Adding a zero infront of a stream has the effect of "delaying" it one tick.
To calculate the derivative of the error, I delay the error stream one tick
and subtract it from itself:

      dedt = e - ([0.0] ++ e)

The integral just adds the error to itself for each tick:

      integral = [0.0] ++ integral + e

All the streams are defined, so I just define a PID controller in the 
textbook way:

      command = u + e * p + integral * i + d * dedt -- a tradition PID controller

The condition I want to monitor is the size of the error.  I just picked a random
value of 0.2.  (It should really monitor the absolute value of the error).

      maxErrExceeded :: Stream Bool
      maxErrExceeded = e > 0.2 -- the condition to monitor

For output I just observed the variables I was interested in and interpreted
my spec the normal Copilot way:
    interpret 50 impulseResponse

I put the output into a .txt file and graphed it with a python script (see end
of post for code).

As a simple first test, I tried P = 0.5, I = 0.0, D = 0.0.  This is not really
a PID controller, but just a P controller.  Here's what I got:
<img src="https://raw.github.com/scottcarr/scottcarr.github.com/master/images/pid_cont_p.png">

Ahh, the good 'ol P term.  He's slow but he's reliable.  The P=0.5 controller
isn't half bad if you ask me.

For a cooler graph, I used P=0.5, I=0.2, D=0.2:
<img src="https://raw.github.com/scottcarr/scottcarr.github.com/master/images/pid_cont_pid.png">
  

graphcont.py:
    import numpy as np                                                                 
    from pylab import *                                                                
                                                                                       
    if __name__ == '__main__':                                                         
        ion()                                                                          
        f = open("cont.txt")                                                           
        y = []                                                                         
        u = []                                                                         
        e = []                                                                         
        intg = []                                                                      
        for i, line in enumerate(f):                                                   
            if i != 0:                                                                 
                s = line.split()                                                       
                #print s                                                                                
                y.append(double(s[-1]))                                                
                u.append(double(s[-2]))                                                
                #e.append(double(s[2]))                                                
                #intg.append(double(s[3]))                                             
        #print y, u                                                                    
        figure(1); clf()                                                               
        plot(y)                                                                        
        plot(u)                                                                        
        ylim(-0.1, 1.5)                                                                
        legend(["target", "actual"])                                                   
        title("p=0.5, i=0.2, d=0.2")                                                   
        show()
