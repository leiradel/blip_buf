This shows how to use blip_buf in an existing sound chip emulator that generates samples itself. It's broken into directories with each change made from the previous. Run a diff between steps to see the changes made.


1.using blip for output at samp
-------------------------------
First just use blip to render the normal samples, without any rate change. Instead of outputting directly to the sample buffer, we use blip at the same time points, then read the samples at the end. This serves to be sure we've got the very basic things in place.


2 Break out delta handling
--------------------------
Move delta-adding code into common function, in preparation for future changes.


3 Clock rate above sample rate
-------------------
Now that delta adding is contained, raise the clock rate above the sample rate again.


4 Eliminate old fractional sample handling
------------------------------------------
Eliminate unnecessary fractional sample handling from old clock rate code.


5 Break out square/noise channels
---------------------------------
This will make it easier to optimize them.


6 Eliminate more fractional sample handling


7 Minor tweaks before channel optimization


8 Optimize channel loops
------------------------
Turn things inside-out so that loop runs once per event rather than per clock, advancing the number of clocks until the next event. Add interface to run by clock precision rather than sample as before.
