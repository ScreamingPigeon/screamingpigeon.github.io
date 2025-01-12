+++
title = 'Danielkroening'
date = 2024-10-09T12:36:28-05:00
draft = true
tags = ['blog', 'talks']
+++

Speaker Bio: Daniel Kroening is a Senior Principal Applied Scientist at Amazon,
where he works on the correctness of the Neuron Compiler for distributed training
and inference. Prior to joining Amazon, he worked as a Professor of Computer Science
at the University of Oxford and is the co-founder of Diffblue Ltd., a University
spinout that develops AI that targets code and code-like artifacts. He has received
the Semiconductor Research Corporation (SRC) Inventor Recognition Award, an IBM Faculty
Award, a Microsoft Research SEIF Award, and the Wolfson Research Merit Award. He serves
on the CAV steering committee and was co-chair of FLOC 2018, EiC of Springer FMSD, and
is co-author of the textbooks on Decision Procedures and Model Checking.



# AI Acceleratord 101

## Annapurna Labs - Silicon Innovation at AWS
- AWS Nitro System: Hyperivisor, network storage, ssd, and security. Virtualization (Zen - Hypervisor) 
- AWS Graviton3 - Powerful and efficient modern applications. ARM CPUs on servers - power efficient
- AWS Inferentia and Trainium - ML accleration

Annapurna Labs is an acquisition.

There has beene exponential growth in LLM Model Complexity in terms of # of parameters. As number of parameters increase
you still need to access each parameter to answer questions. Server compute has not grown as fast as model complexity.
There's going to be a gap between buyers and what industry can deliver.

It's all going to be compiler technology that covers this gap. Offers biggest benefit in terms of performance gains.

Q: Even with 2T param models, theree is insufficient data. How can we ever hit 10T?
A: Not so worried, we will find it somewhere. Humans will work hard, for a little pay, to make the data exist. Backfilling
things like audio and video - eventhough they might not be as dense in terms of information when compared to text

## Why build an AI Accelerator?

ASICs <-> Accelerator <-> FPGAs <-> GPUs <-> Vector <-> VLIW <-> CPUs
Specialized <--> General Purpose

Neural Architectures are getting more and more dynamic. Which means the shape of computing depends on the input. Architecture
called "mixture of expoerts" very dynamic nature. People built llama chips, but these struggle as soon as you do branching
in computation styles. There is some benefit to specialized chips, but there is pushback because they may not be
future proof.

Not only does hardware need to cover this, compilers also need to cover this. 


## Start from workload
- load data from memory 
