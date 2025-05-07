+++
title = 'Developing an Out-Of-Order RISC-V CPU: part two'
layout = 'Default'
date = 2025-05-03T17:46:17-05:00
author = 'Prakhar'
tags = ['projects', 'computer-architecture']
draft = true
+++

This is going to be a follow-up to my last post. I realized that the topic
was going to be to big too cover (and also write) in one sitting. In the last 
[blog post](https://screamingpigeon.github.io/projects/ooo_pt1/), I go over
the general premise of out-of-order processors. In this post, I will try to share 
my experience designing and verifying such a CPU, and reflect on the challenges and
rationalization of certain design choices.



## Timeline 

The computer architecture class gives you about 7 weeks to build out your OOO processor.

- The first week gives you time to setup a frontend (Fetch, Decode, and Instruction Queue) and your instruction cache
- The next 2 weeks give you time to get issue/dispatch logic working, and also create 
functional units for arithmetic operations. You are also exptected to get the ROB and commit logic
going in this phase
- The next 2 weeks are for implementing a data cache, and a load/store unit. You are also supposed 
to get branches working.
- And finally - you have 2 weeks to implement any extra features that you think might help your processor
run better.


During these last 2 weeks - a leaderboard was setup. Nightly runs would evaluate the performance of your 
processor and update your ranking. The best CPU would get $2000 from Optiver - a quant firm.


The cash prize + clout become a strong motivating factor for a lot of teams. The last couple weeks ended
up being a fevrent rush for the top spot - with teams constantly redesigning bits and pieces of their
CPU to milk every last bit of performance



## Building the baseline CPU

### Checkpoint 1

The frontend for any implementation of an OOO processor is pretty straightforward. The only thing to watch
out for is a [superscalar]() implementation. 

Think of the frontend as a single lane one-way street. A (parameterizable) superscalar frontend would require
designing the frontend to support two, three, or N lanes. 


BUT! Don't OOO CPUs only execute out of order. Aren't the issues and commits in order? Yes - but issues 
are not part of this checkpoiht - so my team decided to cross that bridge when we get to it, and ignored 
this problem



