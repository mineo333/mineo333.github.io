---
title: What is QogChamp?
date: 2021-10-26
categories: [QogChamp]
tags: [Kernel Exploitation]
---
So, the primary reason I made this blog was to document and discuss the logic and code behind my rootkit: QogChamp. In this post, I want to answer the simple question: What is QogChamp?

Through this post, I hope to answer some very basic questions about its name, its purpose, and other things.

# What is QogChamp?

QogChamp is a novel Linux kernel rootkit that aims to utilize innovative exploits in order to accomplish goals with more power, more effectively, or with a lesser footprint.

# Why is it called QogChamp?

The reason why this exploit is called QogChamp is quite a long story. Basically, what happened is that the very first version of this kernel exploit was created by changing a string `"PogChamp"` to `"QogChamp"`. This became a long running joke in my friend group, so the name just stuck.

As of today, the codebase and its capabilities have progressed far beyond the original `"PogChamp"` to `"QogChamp"`, but the memories and the name remain. 

# What is the purpose of QogChamp?

The purpose of QogChamp is to provide a series of novel kernel exploits. In my opinion, today's Linux rootkits are getting boring as almost all of them fundamentally rely on syscall hooking (Modifying the syscall table) to perform their tasks. Through QogChamp, I hope to provide an alternative to this method of syscall hooking and provide new and exciting ways to do kernel exploitation.

# What can QogChamp do?

QogChamp can do a variety of things. As of the writing of this piece, QogChamp mainly focuses on exploiting the page cache in order to perform file operations with little to no footprint.

As of right now, I am working on network capabilities that allow QogChamp to communicate covertly without the use of a socket. The plan is to do this by reading and writing directly from a NIC's tx and rx ring buffer, but this is very subject to change as I learn more about Linux networking internals.

In the future, I hope add more features and turn QogChamp into a proper rootkit.

# Where can I find QogChamp?

You can find it at my GitHub [here](https://github.com/mineo333/Qogchamp)

In fact, if you go to the VERY old branch - `working_vuln` - you will find the original code that changed PogChamp to QogChamp. `working-vuln` is a complete misnomer and is named as such because originally, I thought this was a legitimate zero-day.

# What's next?

Over the next few posts, I hope to delve deep into the history of QogChamp as well as explaining what it does and how it gets it done.