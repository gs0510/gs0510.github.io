---
layout: page
title: Talks!
permalink: /talks/
---

These are the talks that I have given in the past!

[Replacing PID Allocation implementation with IDR API](gs0510.github.io/docs/OSS_Europe_Slides.pdf)
The Process ID in the Linux Kernel is implemented using a bitmap. Using the IDR API to replace the implementation not only leads to more readable code, it also leads to lesser code and a speedup in PID allocation.

[Write your own container in Rust!](https://www.youtube.com/watch?v=pCRnMrDeLV4)

You might have seen a few cool looking charts in various presentations about Docker where you get a quick “Docker uses namespaces, cgroups, chroot, etc.” to create containers. But why does it take all these pieces to create a contaienr? Why is it not a simple syscall and it’s all done for me? In this talk, we will dispel the magic behind containers and write our own in Rust!

[Syscalls for Rustaceans!](https://www.youtube.com/watch?v=G0e2lVENaCU)

Even if you haven’t used Rust’s nix package, and have just written println!(“Hello, World!”), you have most certainly used syscalls. In this talk, you will learn what syscalls are, how they work, how you can track them using strace and how you can write your own syscall tracer (strace) in Rust!

[ptrace: The Sherlock Holmes of syscalls!](https://www.youtube.com/watch?v=6Y0-yekwsUQ)

Ever wondered what happens when you ask the debugger to set a breakpoint at line 5? Or how strace allows you to be a debugging wizard without using the debugger or the source?! In this talk, we’ll talk about ptrace, the linux syscall which gives the debugger, strace and many other tools their magical powers!

[So good they can't ignore you!](https://www.youtube.com/watch?v=xsFPAZCcPKQ)

You are good at your job, are polite and deadline oriented. Your long-term professional goals are well-defined, and you work toward them consistently. But working hard is sometimes not enough. People around you need to see and value your work. This talk will cover hard-work, visibility, the myth of meritocracy, and what we owe one another once we are successful.

[Printing floating point numbers is surprisingly hard!!](https://youtu.be/QEZ0N0rrbL0?t=6732)

Not many of us have wondered “how are floating-point numbers rendered as text strings?” and for good reason! This doesn’t seem like a hard problem to solve! But even in 2020, you don’t have guarantees in some languages that when you convert a string to float and vice versa you will get the same number! In this talk we will explore why printing floating point numbers is hard, arbitrary precision arithmetic, and the state-of-the-art dragon algorithms for printing floating point numbers!
