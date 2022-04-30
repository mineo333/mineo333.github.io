---
title: Beginnings
date: 2021-10-26 16:30:00
categories: [QogChamp]
tags: [Linux Kernel Exploitation, Syscall]
---
In this post, I hope to tell you about how QogChamp started and what the original codebase looked like. This will be a fairly brief post and won't get too much into the codebase as the functions used in the original codebase are not used anymore in the current implementation of the rootkit.

## Motivation

QogChamp started out in my high school in a class called Computer Systems, in which we completed a capstone project.  

While searching for an idea for this capstone project, I had the absolutely brilliant idea of attempting to emulate the `ptrace()` syscall without actually hooking anything. This idea made me go down the rabbit hole that eventually led to QogChamp.

## Catching Syscalls
When attempting to emulate `ptrace()`, I initially started by attempting to catch syscalls. As you might imagine, this is really hard without the faculties of `ptrace()`. 

The solution I eventually devised ended up using the `task_struct` regset API. This API is, hilariously, directly a subset of the ptrace API and allows one to get the current registers of any given `task_struct`. This API was absolutely critical to getting this "exploit" to work.

In order to understand how this "exploit" worked, it is important to go over some structures and code. In the original code, I have a function called `get_gen_regset`. What this function does is that it takes in a `pid` and returns the `struct user_regs_struct` (Structure containing the registers and their values) for that pid. This `struct user_regs_struct` was the focal point of this exploit.

Interestingly, `struct user_regs_struct` has a field called `orig_ax`, containing the current syscall number (If applicable). This is critical as if `orig_ax` is a valid syscall number, then we know that a syscall is in progress. Using this fact, we can perform our attack. 

If `orig_ax` is a valid syscall number, then we know that the other registers (Specifically `di`, `si`, `dx`, etc.) contain the system call arguments. Through this, we are able to, in a sense, view syscalls as they go back and forth.

## Drawbacks
This ptrace code was frankly abysmal. It was abysmal for a variety of reasons. First of all, it was completely unpredictable. Due to the nature of this exploit, you were essentially at the mercy of the scheduler/interrupt handler, and you had to hope that your kernel code was running while the program you were trying to track was running.

In addition, because the code could never know when a syscall would happen, the code would be forced busy loop (Infinite for loop) until it found a valid syscall number in `orig_ax`. This, as any seasoned kernel developer will tell you, is a marking of horrible kernel code.

## QogChamp Begins!
It was from this basic exploit that QogChamp got its name and was what inspired me to create the exploit as it is today. 

QogChamp initially started when I was messing with the `write()` syscall. Using the technique described above, I was able to get the buffer passed into `write()`. With a buffer in hand, I wondered what would happen if I modified it. So, I did some research, wrote some page-walk code, and eventually got the capability to modify to the buffer. 

When I modified the buffer, something astounding happened: the changes were persistent. In other words, if I re-ran the program it would use the modified buffer instead of the original buffer. This odd behavior serves as the inspiration for the rest of the exploit.

QogChamp got its name from this because the buffer that I originally testing was `"PogChamp"` and the change that I made was changing the P to a Q.

## What's Next
From here, I'm going to focus on the exploit as it is, and I'm going to try my best to explain the unbelievably complex clockwork that makes this exploit possible.

## References
`get_gen_regset` - <https://github.com/mineo333/Qogchamp/blob/master/src/regset.c#L18>