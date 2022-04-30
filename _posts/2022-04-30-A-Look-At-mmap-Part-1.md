---

title: A Look At mmap - Part 1

date: 2022-04-30 14:00:00

categories: [QogChamp]

tags: [Linux Kernel Exploitation, Syscall]

---

  

## Disclaimer

  

As always, nothing in this article is guaranteed to be accurate nor correct.

  

If you see anything blatantly wrong or missing from these articles, do not hesitate to email me at <sharad@mineo333.dev> or dm me on discord at `write(1,"mineo333",8);#3385`

  
  

## Introduction

  

In these next few articles, we're going to discuss exactly how `PAGECACHE_TAG_DIRTY` is set. We will start by discussing the `mmap` syscall.

  

For brevity purposes, this article is split into a few subparts, as one singular article would be too much.

  
  

## Intro

  

`mmap` is perhaps one of the most important syscalls in the Linux operating system, as it is how memory mappings are formed. It is through `mmap` that programs are run, shared libraries are linked, and so much more. In other words, it is safe to say that without `mmap`, Unix operating systems would crumble.

  

## Memory Mappings

  

One thing we need to thoroughly understand before we even attempt to understand `mmap` is memory mappings.

  

A memory mapping can simply be described as a page-aligned, reserved area in the virtual address space. What the purpose of this reserved area is depends on the memory mapping.

  

There are exactly 4 types of memory mappings: private/file-mapped, shared/file-mapped, private/anonymous, and shared/anonymous.

  

## File-Backed vs Anonymous

  

Let's first discuss the difference between file-mapped and anonymous mappings.

  

Put simply, a file-backed mapping is a mapping whose data comes from a file.

  

For example, suppose we have a file-backed mapping that maps `libc.so.6` and exists between virtual addresses `0x1000` and `0x3000`. In this case, if you dereferenced the address `0x1000`, you would get the first byte of `libc.so.6`. Likewise, if you dereference `0x2000`, you would get the byte 4096th byte in `libc.so.6`

  

In this sense, the file is literally mapped to memory.

  

Note that certain limitations do exist for file-backed mappings. There are two limitations.

  

The first limitation essentially forces you to map the file one page at a time. So, you must map 4096 bytes of the file at a time. As a result, you can't map only the first 500 bytes. Instead, you must map the first 4096 bytes. In addition, the mappings must be on page boundaries within the file. So, you can't map bytes `500 - 4596`. Instead, you must map bytes `0 - 4096`, `4096 - 8192`, etc.

  

The second limitation states that the mappings must be contiguous within the file. So, you can't map bytes `4096 - 8192` and `12288 - 16384` in the same memory mapping. Instead, you must map the entire range of `4096 - 16384` or put them in 2 different memory mappings.

  

File-backed mappings are most often used to map certain portions of an executable when running a program.

  

The second kind of memory mapping is an anonymous mapping. Anonymous mappings are exactly the opposite of file-backed mappings - they are *not* file backed. As such, instead of being filled with file-data, anonymous mappings are initialized to 0 bytes. Common examples of anonymous mappings include the heap and stack, as they are never backed by any particular file.

  

## Shared vs Private

  

While file-backed vs anonymous has to do with the actual contents of the mappings, shared vs private has to do with how mappings interact with multiple processes.

  

Shared vs private can often be slightly more confusing than file-backed vs anonymous, as the meaning can change depending on if we are dealing with a file-backed or private mapping.

Let's start by discussing shared mappings.

Shared mappings are mappings that are *shared* between processes and the disk. What this means depends on if it is anonymous or file-backed.

If a shared mapping is anonymous, then, if `fork` occurs, the resulting process will have the same mapping pointing to the exact same pages. 

For example, suppose Process A has a shared anonymous mapping between addresses `0xf000` and `0x10000`. Suppose Process A does a `fork` and creates Process B. If Process B writes to any address between `0xf000` and `0x10000`, Process A will see those changes. Because of this behavior, shared anonymous mappings are very rarely used aside from certain IPC implementations.

  

If a shared mapping is file-backed, then any writes made to the mapping will be written back to the disk.

For example, suppose Process A maps a file called `example.txt` to memory between addresses `0x1000` and `0x3000`.  Because the mapping is shared and file-backed, any changes made to any address in that range will be seen in `example.txt`. 

Naturally, if a process with a shared file-mapping does a `fork`, any changes made to the mapping can be seen by both the parent and the child (As the changes are committed to the disk).

On the other hand, private mappings are mappings that are *private* to a particular process. Another term that is used for these mappings is copy-on-write (CoW) as the pages in these mappings are copy-on-write. Like with shared mappings, the meaning of this changes depending on whether the mapping is file-backed or anonymous.

  

If a private mapping is anonymous, then it is copy-on-write between processes. 

For example, suppose Process A has a private anonymous mapping between `0x1000` and `0x3000`. Suppose Process A then does a `fork` creating Process B. If Process B writes to any byte between `0x1000` and `0x3000`, then Process A will not see that change.

  

We will discuss how exactly the kernel accomplishes this when we talk about page faults and the `mmap` implementation.

  

If a private mapping is file-backed, then any write will not be committed to the disk. Similarly, much like with anonymous mappings, any writes made to file-backed mappings will also not be shared by processes. In other words, file-backed, private mappings are anonymous mappings + the changes don't get committed to the disk.



  

With an understanding of all of this, we are prepared to talk about the `mmap` syscall.