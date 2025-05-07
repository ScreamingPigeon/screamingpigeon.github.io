+++
title = 'Developing an Out-Of-Order RISC-V CPU: part one'
layout = 'Default'
date = 2025-05-03T17:44:00-06:00 
author = 'Prakhar'
tags = ['projects', 'computer-architecture']
+++


This post is about 5 months late. I created the markdown file on 12/23/2025
but never got around to writing anything to it. Now, I have some time on my hands (and I 
feel bad about not writing as often as I had planned to) - so I figured I will give this a shot
again.


Last fall, I took [ECE 411](https://courses.grainger.illinois.edu/ece411/fa2024/https://courses.grainger.illinois.edu/ece411/fa2024/)
the infamous computer-architecture class at my university. This was a class I was really looking
forward to. After dealing with some department advising beauracracy (and almost losing my seat in the course)
I did end up taking the class - and I quite enjoyed it. The final project of the class
involves designing a Out-Of-Order RISC-V CPU. This blog post is going to hopefully go over

- Theory of Operation for OOO CPUs
- My experience writing the RTL
- Exposing the rationale for some design choices


EDIT: I am an hour into writing this post. It is clear to me that is going to be hard to 
fit a semester's worth of content into a single blog post. Therefore I have decided
to split this into 2 posts. This one will focus on theory of operation and the next one will
cover my personal experience developing the project, and some exciting news for the future.
Hopefully it doesn't take me another 6 months to write it XD



# What is Out-Of-Order?

If you're not already familiar - I would recommend you look at
        - [Von Neuomann Architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture)
        - [In-Order Pipelined CPUs](https://en.wikipedia.org/wiki/Instruction_pipelining)
        - [Caches](https://en.wikipedia.org/wiki/Cache_(computing))

In most computers - memory is very limited. CPU registers are implemented using flip-flops
which are power hungry, and occupy considerable area. As such you end up consuming a lot of
power/area per bit. BUT, the CPU gets instant access to the register. Therefore, the number 
of CPU registers is usually constrained to a number that optimizes Power/Area/Efficiency for
a given use-case. For example, the RV32-I spec of RISC-V comes equipped with 32 32-bit CPU
registers.

In order for you to run any non-trivial program - you need fast access to memory. Caches are 
usually implemented in SRAM, which give you better power/area but are much slower. If the 
data you want is not present in the cache, then you go to DRAM which is cheaper than SRAM - but
slower and physically far away from your CPU --> which means even slower access times.

And if the data you want has not been paged to DRAM, you go hunting for it in your HDD/SSD which
is an incredibly slow process for a Processor that runs at MHz/GHz frequencies.


Getting back to why Out of Order Matters

Say for example you are attempting to perform the following operation on
```

 1. c = b + M[d]

 2. a = c + b

 3. d = d + b
```

This is a straightforward addition operation. Let's assume `c`, `b`, `d`, and `a` are registers.

`M[d]` means we are attempting to ["dereference a pointer"](https://en.wikipedia.org/wiki/Pointer_(computer_programming))
, i.e. access the data stored at the memory address whose value is stored in register `d`

For the sake of our example, let's say the data at address `d` is not present in the cache
- what happens? The Memory System goes up the memory hierarchy searching for it. Depending
on the state of your computer - it make take anywhere between 10 to thousands of CPU cycles 
to get that data ([AMAT](https://en.wikipedia.org/wiki/Average_memory_access_time)).

A traditional in-order pipelined/multicycle CPU would have to stall all the pursuant instruction
until this memory read is resolved, and only then will execute instructions 2 and 3.

This is a huge waste of time. The fetch, decode and execute stages/hardware of the CPU just
sits there - unutilized while we wait for the memory system to respond. OOO tries to fix this 
problem by exploiting [Instruction Level Parallelism](https://en.wikipedia.org/wiki/Instruction-level_parallelism)


Now let's take a look at instructions 2 and 3. Evidently, instruction 2 is dependent on 
the outcome of instruction 1. What about instruction 3? It has no dependency on 
Instructions 1 or 2.

An OOO CPU is capable of executing instructions Out-Of-Order and this example is a great scenario
where an OOO processor would utilize it's "free" hardware resources to execute the pursuant 
"independent/ready" instructions.


# How does it work - Tomasulo

Out of order processors are not a new idea. IBM had already come up with an [OOO hardware
algorithm](https://en.wikipedia.org/wiki/Tomasulo%27s_algorithm) back in 1967. I highly recommend
watching [this video](https://www.youtube.com/watch?v=zS9ngvUQPNM) for a more visual explanation.
It is what I used to get a sense of what is going on when I took the class, and is probably
far better than my attempt here.

! insert picture here


The basic principle of this algorithm, called Tomasulo's Algorithm - is that instructions 
are issued in-order, executed out-of-order, and committed/written-back in-order.

In a Tomasulo processor, there is one common frontend (fetch, decode, issue) - multiple "functionl
units" (which are ALUs and Memory Units for load/stores). Once executed - the functional
units send the result to the commit buffer / reorder buffer - that commits the results of 
the instruction in-order.


Each kind of functional unit has a queue known as reservation stations. 

    The reservation stations control when an instruction can execute


Each functional unit has a tag . The reservation stations, and the FUs are connected through
a Common Data Bus. Each "broadcast" of a completion from the FU is accompanied by a tag

Instuctions are fetched and decoded 
in order, and then placed in the issue queue.

Each register/operand in an operation is assigned also assigned a tag - via a technique called
["Register Renaming"](https://en.wikipedia.org/wiki/Register_renaming).
In Tomasulo, each operand is assigned a "tag". Unused registers have a special tag. Registers
that are already in use - are assigned a placeholder tag - usuallly pointing to the tag of 
the functional unit where we expect the result to originate from.

Let's go back to my initial example


```
 1. c = b + M[d]

 2. a = c + b

 3. d = d + b
```
In RISC-V memory and arithmetic operations cannot be muddled. So let's change the example a little
```
 0. e = M[d]

 1. c = b + e

 2. a = c + b

 3. d = d + b
```

### Execution Flow with Tomasulo's Algorithm:

* **Instruction Fetch and Decode:**
   - All three instructions are fetched and decoded in order. The CPU identifies the operations and operands involved.

* **Issue Stage:**
   - **Cyle 0** Issued Instruction 0: `e = M[d]`
     - Since this is the first instruction, let's assume all the registrers and reservation stations are available
     - This is a load operation. Let's assume loads take 4 CPU cycles.
     - `d` is immediately available to use, so it has a tag of `0`
     - since `e` is the destination, it's tag is changed from `0` to `L` (the tag of the memory FU - it can be any arbitrary distinct number)
     - This instruction is now in the RS for Memory FU and will executing in 4 cycles
   - **Cycle 1** Issued Instruction 1 `c = b + e`
     - This is an arithmetic operation. Let's assume it takes 1 cycle to execute
     - The CPU checks if `b` is available. It is, so a tag of `0`
     - The CPU checks if `e` is available. It has a tag of `L`
     - Since `C` is the destination register, it is tagged to `A` (the tag associated with ALU)
     - This is issued to the Reservation Station for the ALU. Howe 
     - Since the operand for Instruction 0 is available, it starts being executed by the memory FU
   - **Cycle 2** Issued instruction 2 `a = c + b`
     - `c` is dependent on the result of Instruction 1, so it takes on the tag of `A` 
     - `b` is tagged as `0` since it is readily available.
     - Instruction 2 is also issued to the RS for the ALU FU
     - Instruction 0 is on the 2nd cycle of execution
     - Instruction 1 is waiting for a broadcast on the CDB associated with the tag `L` so it can mark its operand as ready and begin execution
   - **Cycle 3** Issued Instruction 3 `d = d + b`
     - This instruction can proceed independently as it only requires `d` and `b`, both of which are available with tag `0`.
     - It is issued to the ALU RS
     - Instruction 0 on it's 3rd cycle
     - Instruction 1 waiting for broadcast with tag `L`
     - Instruction 2 waiting for broadcast with tag `A`
   - **Cycle 4**
     - Instruction 0 on it's 4th cycle
     - Instruction 1 waiting for broadcast with tag `L`
     - Instruction 2 waiting for broadcast with tag `A`
     - All the operands for instruction 3 were ready, so it is being executed by the ALU FU this cycle
   - **Cycle 5**
     - Instruction 0: The load finally completes. The FU can broadcast its result with tag `L`
     - Instruction 3: This also finishes (in one cycle). The ALU FU can also broadcast a result with tag `A`

     Now - there is only one CDB connecting the RSs and FUs. It is the "common" bus after all. Which FU can broadcast
     comes down to an arbitration policy. Let's assume that the memory FU gets first pick. So the tag `L` and result is
     broadcasted. Since the ALU FU can't broadcast - it stalls this cycle. (NOTE: There are a lot of CDB arbitration policies, including 
     having multiple CDBs. We will discuss this more in the next blog post)

     - Instruction 1 receives broadcast with tag `L` - but the ALU FU stalls since Instruction 3 was unable to broadcast
     - Instruction 3 waiting for tag `A`

   - **Cycle 6** 
     - Instruction 3 broadcasts its result, and the ALU accepts the next available instruction to exec
     - Instruction 1 now being executed.
     - Instruction 2 is still waiting
     - Instruction 4 is now being committed/placed in Reorder buffer (ROB)

   - **Cycle 7** 
     - Instruction 1 broadcasts `A`, ALU ready again
     - Instruction 2 operand becomes available, starts being executed by ALU
     - Instruction 3 in Reorder Buffer, waiting to for previous instruction to be be committed 

   - **Cyle 8**
     - Instruction 1 comits
     - Instruction 2 finishes and broadcasts
     - Instuction 3 waiting in ROB
   - **Cycle 9**
     - Instruction 2 commits
     - Instruction 3 waiting in ROB

   - **Cycle 10**
     - Instruction 3 commits



Now obviously I have glossed over some of the details here. Reorder Buffer and commits form different
stages of the backend of the CPU. 

There is an increased pipeline depth in out of order processors due to more need of processing instruction dispatching

But we were able to demonstrate how instructions were able to execute out of order.

The main reasons for stalls in such a processor come from
- ROB getting full - this happens when an instruction in the "Head" of the ROB takes particularly long to complete, and all pursuing 
  instructions complete and occupy the ROB, but can't be committed because the instruction at the head hasn't committed. This 
  is also known as **ROB Starvation**
- Reservation Station getting full
- Issue queues getting full



Making these structures larger allows us to exploit bigger ILP windows - but also increases the combinational logic
needed to make decisions on readiness - limiting clock speed, increasing area and power consumption.



# The Alternative: ERR
There is another alternative to Tomasulo's algorithm: Explicit Register Renaming.

I found [these slides](https://people.eecs.berkeley.edu/~kubitron/courses/cs252-S09/lectures/lec08-prediction.pdf)
that have some nice visuals on ERR. The key difference between ERR and Tomasulo is that ERR de-links scheduling
and renaming. Instead of "tagging" registers to functional units - we have an architectural register file
and a physical register file. The physical registers are far more than the architectural registers. For example
my project CPU had 64 physical CPUs - and only 32 architectural registers.  here is a Register Alias Table 
(the RAT) that keeps track of which physical registers map to which architectrual registers.
Kind of like a hash table. There is also a retirement RAT which keeps track of mappings after 
instruction have completely finished execution and have been committed. This 
helps us deal with branches (which I will talk about more in my follow-up post). 


![funny picture of the RAT](/projects/ooo_pt1_rat_smol.png#center "A RAT")

Each Functional unit has it's own issue queue. which holds instructions that have been dispatched. The issue
queue contains some issue logic - which is a muxtree that "dispatches" instructions with ready operands to the 
functional unit.


ERR can be explained with the following examples:
```
i0. x1 = x2 + x3
i1. x4 = x1 + x5
i2. x1 = x5 + x2
i3. x5 = x1 + x2
```

Let's assume our CPU spec presents only 5 architectural registers (`x1` - `x5`). In an ERR processor, we have 
physical registers - usually more than the number of architectural registers. In our case, let's say we have
10 physical registers (`p1`-`p10`).


The instructions abover present 3 types of "dependencies":
- Read after Write  (RAW): Instruction 0 and 1
- Write after Write (WAW): Instructions 0 and 3
- Write after Read  (WAR): Instructions 2 and 1, Instructions 3 and 2


To make the hazards posed by these data dependencies clear - let's assume that the ALU takes 3 clock cycles to 
complete a 32 bit addition. We have 2 ALU Functional Units in this ERR CPU. Let's assume for the sake of simplicity,
that the issue queues have a depth of "2" (these are usually parameterized)

### Execution Flow with ERR

- **Instruction Fetch and Decode**: All the instructions are fetched, and decoded sequentially - and placed in the instruction queue
  Since there were no instructions prior to these - all the hardwware resources in the deeper stages of the CPU (issue queues, ROB)
  are unoccupied. So instructionsd are sent to the issue queues as they arrive

- **Cycle 0**:
  - `i0` dispatched from instruction queue to issue queue. Since x1 is the destination being written to - it is mapped to
  some physical register `p2` during the issue logic. This is usually done using a bitvector known as the free list. The RAT is updated
  to reflect the new mapping

- **Cycle 1**
  - `i0` moves from issue queue (`IQ1`) and starts being executed by `ALU1`. 2 more cycles to go for it to finish execution
  - `i1` dispatched from instruction queue to an issue queue. Let's say we have a random policy to choose which non-full IQ an instruction
     is issued to. So it is now sitting in `IQ2`. During the issue logic - we know that that `x1` is mapped to `p2` - which is used
     as one of the operands. Since the `x4` is being written to - it is remapped to the next free physical register `p3`. The RAT and free
     list are updated to reflect this. 

- **Cycle 2**
  - `i0` on its 2nd cycle of execution 
  - `i1` waiting in `IQ2` for `p2` to be become available - a RAW dependency - the only "true" dependency
  - `i2` dispatched randomly to an instruction queue 2 `IQ2`. This queue is now full.  This instruction also writes to 
    `x1`. However, `i0` also does that.... what's going on? This is our Write after Write dependency. However, since it is
    `x1` we are writing to - we can simply map it to a new free physical register `p4`. This in no way affects the execution
    of `i1`. This new mapping also ensures that the `x1` used as an operand in `i1` is not affected by the write, since it relies
    on the old-mapping to `p2` - enforcing correctness and eliminating the Write after Read dependency.
    Therefore it is possible to execute `i2` before `i1` without affecting the correctness of the operations - since
    they were issued in-order.


- **Cycle 3**
  - `i0` on its last cycle of execution
  - `i1` waiting in `IQ2` for `p2` to be become available 
  - `i2` moves from `IQ2` to `ALU2` and starts executing
  - `i3` dispatched to `IQ1` - waiting on the results of `i2` or for `p4` to become free (RAW)

- **Cycle 4**
  - `i0` finishes execution and moves to the ROB to be committed. Depending on how forwarding is implemented
    the issue ques may be immediately aware of `p2` becoming available, or only after being committed. In order to keep
    this example short, let's assume some sort of same-cycle CDB-->IQ forwarding mechanism is in place.
  - `i1` ready now, since it has its operand `p2` ready. However `ALU2` busy so it needs to wait
  - `i2` on its 2nd cycle of execution in `ALU2`
  - `i3` waiting on `p4` to be ready in `IQ1`

- **Cycle 5**
  - `i1` waiting for `ALU2`
  - `i2` on last cycle of execution in `ALU2`
  - `i3` in `IQ1` waiting for `p4`

- **Cycle 6**
  - `i2` completes and `p4` broadcasted (NOTE: This instruction will wait in the ROB until `i1` is committed.)
  - `i1` starts execution as `ALU2` becomes available
  - `i3` starts execution as `p4` becomes available

- **Cycles 7 and 8**
  - `i1` and `i3` executing on the 2 ALU FUs

- **Cycle 9** 
  - Both `i1` and `i3` complete. Depending on bus arbitration policies - one, or both of them broadcast

- **Cycle 10 (optional)**
  - The remaining instruction (if at all) broadcasts




And now we have demonstrated how ERR allows for out-of-order execution, solving the problem of WAR and WAW dependencies.
ERR is usually more power hungry and larger than Tomasulo style processors.
The ILP window can be made larger by increasing the size of the ROB, Issue Queues, and increasing the number of FUs/CDBs

## Discussion

And now I have tried to explain the basic premise of OOO cpus. Nothing beats the IPC of pipelined CPUs if memory was ideal and
didn't take ages to respond and was cheap for power/area. But OOO CPUs do a good job with what we have.

It is worth noting that branches get much more complicated to deal with in a OOO CPU. In the case of a branch resolving, and
the prediction being incorrect - when said branch instruction is committed from the ROB - and the misprediction is discovered - the
entire CPU is flushed. Because of the number of queues and stages in an OOO processor - this is an expensive affair - significantly
degrading its performance.


# Conclusion
This blog post took me a lot longer to write than I was expecting. It's my first attempt at trying to explain a fairly technical
topic over text only. I have glossed over a lot of implementation details (ROB, CDB, RAT) and only tried to convey the intuiton for
how OOO processors beat certain limitations - so reading a textbook/lecture slides on this is highly recommended. It should help
cement the foundation of understanding, and go over implementation details I would also recommend taking a look at
 - Computer Organization and Design RISC-V Edition: The Hardware Software Interface by David A. Patterson and John L. Hennessy
 - Computer Architecture: A Quantitative Approach (6th Ed.) by John L. Hennessy and David A. Patterson
 - [BOOM](https://docs.boom-core.org/en/latest/)
 - [Processor Microarchitecture an implementation perspective](https://dl.icdst.org/pdfs/files/15b09def448c317556dc0fc412aee571.pdf)



Part 2 of this will cover some of my experience designing a baseline CPU, and writing benchmarks to see what needed extra bells
and whistles to run better.

If you have an ideas for improving the clarity of this writeup- do let me know :D



