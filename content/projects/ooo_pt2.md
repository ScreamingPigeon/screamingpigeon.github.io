+++
title = 'Developing an Out-Of-Order RISC-V CPU: part two'
layout = 'Default'
date = 2025-05-05T17:46:17-05:00
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
out for is a [superscalar](https://en.wikipedia.org/wiki/Superscalar_processor) implementation. 

Think of the frontend as a single lane one-way street. A (parameterizable) superscalar frontend would require
designing the frontend to support two, three, or N lanes of instructions.  

BUT! Don't OOO CPUs only execute out of order? Aren't the issues and commits in order? Yes - but issues 
are not part of this checkpoiht - so my team decided to cross that bridge when we get to it, and ignored 
this problem for the time being.

For now, we designed the fetch stage to request an address (pretty straightforward), but then fetch `(PC + N)` on the 
next cycle. The decode stage was designed to handle decoding (N) instructions (`PC`, `PC + 1`,`...` ,`PC + N-1`) 
instructions simulatenously. The reasoning was that, we could just design our icache to spit out N instruction given that
`N < cacheline size /4`.

We were considering to fix N * instruction size to 1/2, 1/4 or 1/8 of the cacheline size to streamline the logic.


I personally worked on implementing the cache and the cache arbiter. In Von-Neumann Architecture, the programs (instructions)
and data reside in a "unified memory space". However in practice, there are several benefits to physically 
differentiation instruction and data memory. Hardware design influences software, and software influences hardware design
(forming a feedback loop if you will). 

An example would be [`ELF`](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) has different sections for the program (.txt) and data
(.bss). And in most cases (except a program that explicitly modifies itself (aka its instructions)) - there are a lot of 
performance benefits from seperating the memory subsystem for instructions and data. Moreso, modern compilers will probably
optimize away self-modifying patters when compiling code for processors that have a split memory subsystem.

Anyways, getting back to the point here. The main benefit of splitting the i-cache and d-cache would be
- The icache ends up being read only, reducing size, area, and allowing faster clock speed on fetch
- icahce and dcache can be used in parallel, by the fetch stage, and by a load-store FU, reducing stalls

In our case, the memory was unified at the RAM level (there was a single block RAM for ALL memory), and as such 
the caches would need to go through an arbiter. We decided to always prioritize the instruction cache as "slowdowns"
in the fetch stage would affect the entire system, as opposed to slow downs on the load-stores would just affect its FU.


The instruction queue, that would hold decoded instructions - was designed as a [FIFO](https://www.chipverify.com/verilog/synchronous-fifo).
We parameterized the size of the FIFO, and icache, so as to be able to sweep parameters later in the assignment to select
the most optimize our design.

Given the limited amount of work needed for the checkpoint - completing it was fairly uneventful.

### Checkpoint 2

Completing this next checkpoint probably required the most grinding - given the insane time crunch. We basically
had to design and test the remaining skeleton of the CPU in 2 weeks. The main stages that needed work were
- Physical Register File
- Issue logic (Register Renaming, RAT, Free List, Issue Queues, Dispatch logic)
- I-extension Functional Unit
- M-Extension Functional Unit
- CDB, CDB Arbiter
- Backend (ROBs and Commits, RRAT)

I worked on the RAT, Free List, PRF,  I-Extension Functional Unit, and the RRAT, and the CDB arbiter for this checkpoint. 





### Checkpoint 3
Another stacked checkpoint (although not as terrible as CP2). For this one we were implementing
- Static not-taken branch prediction scheme
- Branch Resolution and Flushes
- Loads and Stores (FU)



### Competition Spec
For the competition, we decided to implement the following features
- Branch Predictor with BTB with Saturating coutner
- Split Load Store Queue
- Pre-commit store buffer
- Next line prefetcher


