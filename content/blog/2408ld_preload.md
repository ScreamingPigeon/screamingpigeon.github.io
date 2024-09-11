+++
title = 'Interacting with root processes using $LD_PRELOAD'
layout = 'Default'
date = 2024-08-09T05:41:22+05:30
author = 'Prakhar'
tags = ['blog', 'operating-systems', 'compilers','c']
draft = true
+++

# What is `$LD_PRELOAD`?

As listed in the [man pages](https://www.man7.org/linux/man-pages/man8/ld.so.8.html) for `ld.so`:
>  **LD_PRELOAD**
>
> A list of additional, user-specified, ELF shared objects
> to be loaded before all others.  This feature can be used
> to selectively override functions in other shared objects.

Effectively, `$LD_PRELOAD` is an environment variable that points to
one or more Shared Object (.so) files that are linked
to an ELF file at runtime.

Clearly, this can lead to some pretty cool things.


