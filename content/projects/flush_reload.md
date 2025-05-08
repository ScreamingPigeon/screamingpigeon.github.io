+++
title = 'Flush Reload'
date = 2025-05-06T15:09:05-05:00
layout = 'Default'
author = 'Prakhar'
tags = ['projects', 'computer-architecture']
draft = true
+++

# Intro
Last November, I got the chance to meet [Daniel Genkin](https://faculty.cc.gatech.edu/~genkin/) - at the [Midwest
Security Workshop](https://www.midwestsecurityworkshop.com/agenda.html). He is one of the authors/discoveres of
[SPECTRE](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) and [MELTDOWN](https://en.wikipedia.org/wiki/Meltdown_(security_vulnerability))
, some of the most critical vulnerabilities discovered in the last decade. These were also unpatchable due to them being
 hardware vulnerabilities.

Bumping into Daniel was a happy coincidence for me. I was aware that MSW had a section on HW Security (which is why I went in
the first place) - but I never knew that such an epic security researcher was going to be there. 

Talking to him was a definitely a cool experience - and I got some solid advice about how to evaluate fields of study for 
Graduate School. My main problem with grad school is having one semester to decide on what you want to do - and then 
committing to it for the remaining 4.5 years (if not longer). Daniel recommended that I try recreating some of 
the cornerstone HW vulnerabilities to get a feel of how much I like the work. 

If I recall correctly, his perspective of HW sec, particularly that towards exposing vulnerabilities is - 


> *Replicate every single known hardware vulnerability on a new target and collect results. It is highly unlikely
    that you will discover something in your first attempt. New hardware is immensenly complicated, proprietary,
    and obfuscated. A huge chunk of the work is reverse-engineering the system inside.*

> *Eventually, you will learn bits and pieces about what's inside - and then you can try more specific approaches,
    but even then, there is no guarantee of success.*

This is me paraphrasing, but I feel like it captures the gist of what he said pretty accurately. Coming up with HW
exploits is an extremely iterative process.


Anyways, getting back to why I am writing this. I am going to attempt to describe [Flush + Reload](https://dl.acm.org/doi/10.5555/2671225.2671271)
, an important demonstration of a [Side Channel attacks](https://en.wikipedia.org/wiki/Side-channel_attack) - and then create
a naive implementation/demo.


# Flush Reload 

A few important concepts before we get into how this attack works

### Paging
[Paging](https://en.wikipedia.org/wiki/Memory_paging) is a term you might have heard if you have taken an Operating Systems or Computer
Architecture course in university. Paging operates on the principle of maintaining a "logical" or "**virtual**" map of your memory, that
is mapped to the actual "physical" memory on your system. Different processes utilize distinct mappings between virtual/logical memory 
and physical memory - which is enforced by the kernel. Such a techniquee offers 3 advantages 
- Security: Because processes use different paging schemes, it becomes hard for a malicious process to interfere with other 
processes on your system. The ability to manipulate pages is managed by the kernel - and as such - only priveleged contexts
can manipulate pages.

- Simplicity: Programs no need to allocate physically "contiguous" memory. For example - you need 2GiB of contiguous memory to
allocate an array. With paging - the OS can map blocks of memory at scattered physical addresses to a "continous" address space 
in userspace. This makes writing user programs simple as you no longer need to worry about [memory fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing))

- Efficiency: Pages containing memory that is needed by 2 different processes in different address spaces can be remapped and reused. A single physical adddress
can be be mapped to 2 different logical addresses - and reused. Additionally it becomes easy to manage memory consumed by "idle"
processes. The pages used can be "swapped" from RAM to Disk - freeing up low-latency space for active processes.

I would highly recommend reading [Operating Systems: Three Easy Pieces, Ch 18](https://pages.cs.wisc.edu/~remzi/OSTEP/)
if you want to learn more about paging. It is a thorough explanation.


### Caches
As mentioned in [this entry](https://screamingpigeon.github.io/projects/ooo_pt1), low-latency memory is very expensive to 
fab write into the die of the CPU. As such - there usually exists a memory hierarchy like 

    Registers --> L1 Cache --> L2 Cache --> L3 Cache --> (DD)RAM --> Disk(HDD/SSD)

Going from left to right, memory becomes cheaper but also slower. 

A modern day CPU contains several cores. Usually - the first 3 stages of the memory hierarchy is per-core, with the 
Last Level Cache (LLC) being shared between all the cores. My work computer (the one I am currently typing this post on XD),
has a [Intel i7 10850H](https://www.intel.com/content/www/us/en/products/sku/201897/intel-core-i710850h-processor-12m-cache-up-to-5-10-ghz/specifications.html#specs-1-0-2)
which has a 12 MB smart-cache. I assume almost all of its size comes from the LLC.


### TLB and Cache-Paging Interactions

Classic x86 utilized a hardware structure known as the [Translation Lookaside Buffer](https://en.wikipedia.org/wiki/Translation_lookaside_buffer)
(TLB). This was essentially a fully-associative cache that held the mappings between virtual and physical memory addresses. When
changing the address-space - the TLB was modified to reflect the new mapping.


Now, if you think about it - All the loads and stores usually happen from/to the L1 Cache. How are the Cache and TLB
synchronized with each other. This is a completely valid concern -  does the cache use physical addresses, or 
virtual addresses. What happens if there is an address-space switch? Do we flush the entire cache?

There are 3 basic schemes of Cache-TLB coherency known as:

1. PIPT - Physically Indexed Physically Tagged
2. VIVT - Virtually Indexed Virtually Tagged
3. VIPT - Virtually Indexed Physically Tagged

Usuaully VIPT is the more robust scheme. Intel tends to use this scheme in its caches. If you are curious about how 
these schemes work and why VIPT has benefits, [these slides](https://view.officeapps.livehttps://view.officeapps.live.com/op/view.aspx?src=https%3A%2F%2Fcourses.physics.illinois.edu%2Fece411%2Ffa2022%2Fslides%2Flect08_l.pptx.com/op/view.aspx?src=https%3A%2F%2Fcourses.physics.illinois.edu%2Fece411%2Ffa2022%2Fslides%2Flect08_l.pptx)
do a pretty good job (I found these after a quick google search).

**Section 2 of the Flush + Reload** [paper](https://dl.acm.org/doi/10.5555/2671225.2671271) also explains the prerequisite 
concepts needed to understand how the attack works.


## How does Flush Reload work?
Flush + Reload works on x86 systems, and runs on a seperate core than the target process - effectively running in parallel.
It also operates by manipulating pages that are shared by both the target and the attacker.

The attack consists of 3 phases - **flush**, **wait**, and **reload**. 

1. **Flush**:

The monitored cacheline containing the shared page is flushed from the memory hierarchy. This is done with the 
[`clflush`](https://www.felixcloutier.com/x86/clflush) instruction. This instruction basically takes an 
address, and flushes any cacheline containing said address from all 3 caches.


2. **Reload**:

The attacker waits for some unspecified amount of time.


3. **Reload**:
The attacker attempts accessing, or "reloading" the address it just flushed from the hierarchy. The amount of time
it takes for this load to execute is measured.



Depending on how long the reload takes - it is possible to infer if the target process accessed that cacheline. A short 
reload time implies that the data was reloaded by the target into the shared LLC, 
and just needs to be loaded into CPU-specific caches (L1 and/or L2). A long reload time implies that the cacheline 
was not loaded by the target, as it includes the delay of fetching a line from RAM to the LLC.

The target's access of this cacheline is independent of the attacker process. This means that a long wait time can help
guaruntee a fast reload - but it reduces the temporal resolution of the attack.

The paper mentions that an approach to improve attack granularity is to target looped executions in the target program.
The repeated accesses can allow for a small wait phase - allowing the attacker to sample memory at a higher rate.

The actual "attack" part of this exploit runs off of statistical analysis of the delay times for different cachelines - 
which can be used to infer program state for popular open-source programs, like GPG (Gnu's Privacy Guard) - an opensource 
security library that implemnts [RSA](https://en.wikipedia.org/wiki/RSA_cryptosystem) encryption.


### Inferring Private Keys

Using memory access times to decipher private keys sounded pretty unbelievable when I first read the paper. But it became 
intuitive pretty quick. 

The algorithm for RSA is well documented. GPG is FOSS and it's [source](https://github.com/gpg/gnupg) is available to anyone.

It is known that the sequence of operations Square-Reduce-Multiply-Reduce indicates setting of a `1` in the 
key. The sequence Square-Reduce without a multiply next indicates setting of a `0`.

If the attacker were to constantly flush instructions associated with Square, Reduce, and Multiply,
and then poll load times - it would be so easy to extrapolate what bits are being set in the key!
Low load times for cachelines associated with Square would imply the target did a square operation - and
so on. 

This is exactly how flush reload works. Figuring out a sampling rate to get reliable data is hard -
but that's just an implementation problem - we have intuition on our side now.

In fact, overcoming the implementation details of this attack would appear to be the hard part.


### Challenges


1. OOO Execution: 
2. ASLR 
3. Context Switches
4. Speculative Execution
5. Virtualization
