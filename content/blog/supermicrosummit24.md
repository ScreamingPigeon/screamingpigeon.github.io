+++
title = 'Supermicrosummit24'
date = 2024-08-27T11:55:40-05:00
layout = 'Default'
tags = ['blog', 'conference']
+++


# Super Micro Open Stroage Summit 2024
[https://www.thecube.net/events/supermicro/open-storage-summit-2024](https://www.thecube.net/events/supermicro/open-storage-summit-2024)

I attended the last 2 events at this stummit. Here are my thoughts and notes:

## The New High Performance Computing: Optimized Storage from HPC to AI
Speakers:


- Randy Kreiser Supermicro
- CJ Newburn NVIDIA
- Balaji Venkateshwaran DDN
- Bill Panos Solidigm


CJ-
The moderator looks like he is being held at gunpoint.

 Rate at which usage models are changing

need for new infrastructure

new models characterized by large scale and making
    memory accesses of fine granularity

key-value store based storage systems?

At a rack level, there is much more integration.
GPU is initiating requests from gpu, threads acting individually
cpu/gpu direct storage.

Higher fine grained iops requirements from newer
applications.



Balaji - 
DDN wants to be the one stop shop for storage and data management
Streamline and optimize the entire flow for large scale datasystems
Automate dataplacement and maximize flash performance.
Secure multitenancy, maximize safe resource utilization.

As the utilization of data and blocks increases, are you able to maximize
RoI of the other infrastructure? You want the ability to keep the GPU busy
using an optimized end2end application stack.

Don't want the storage to bottleneck the entire system.


Bill - 
Think about the subset of data types (block, object, files) needs to be 
collected/augmented on physical media. Needs to be optimized against
density per terrabyte. High Capacity and sequential writes. Sequential
writes happen in real time. You want a product that can do ingress and egress
at the same level of bandwidth for AI applications.

Randy -
Liquid cooling is becoming an important thing. Musk is designing his 
new datacentres around liquid cooling. A lot of what we talked about 
has requirements like 1.8tb/s use pcie gen5 right now, pice gen6 in the future. 
You're always going to need hard drives. We're getting to exascale systems. Nvme
is getting to a terrabyte per second speeds. We're looking at virtually 10tb/s in the 
near future (provided by ddn).



CJ -
GPU use cases have been changing from compute monsters
to data io monsters. thousands of threads doing individual accesses
Accessing storage that might not fit in memory in concurrently, which
means you need high bandwidth pcie. working scaled accelerated access (scata).

Drastically increase the iops, and do fine grained access and do scat and gather and pass
data through pcie busses as fast as possible.

Another big concern is security. The compute infra is so large, we can't 
afford to not share it. Storage is a vulnerability. Partnering with DDN
on infinea. Plugin something at user level that sends file requests to the dpu on pcie bus
which should drastically improve security

Bill -
Gen5 just started shipping in volume, wild to be thinking about Gen6 already. The progression in interface is speeding up...felt like we were on Gen3 for a lifetime.


Balaji -

the datapath from flash to the gpu is the thread that ties these companies
together. As the use cases and data types explode, you want to abstract
away complexity for the customers.

The entire datapath between the gpu and flash - is secure scalable and fast.




Lots of coprocessors coming to meet future applications and optimize for power
and performance.
