---
title: An In-Depth Look Into mmap
date: 2022-04-30 14:00:00
categories: [QogChamp]
tags: [Linux Kernel Exploitation, Syscall]
---

## Disclaimer

Although my aim is to make this article as comprehensive and correct as possible, no information in this article is guaranteed to be correct nor is it guaranteed to cover all edge cases and all portions of the implementation. I hope to use these articles as a place where I can share my thoughts and perspectives on the Linux kernel.

If you see anything blatantly wrong or missing from these articles, do not hesitate to email me at <sharad@mineo333.dev> or dm me on discord at `write(1,"mineo333",8);#3385`

**Note that this article is decently incomplete. I just wanted to get it out there**

## Introduction

In these next few articles, we're going to discuss exactly how `PAGECACHE_TAG_DIRTY` is set. We will start by going the `mmap` route and describing the `mmap` syscall. 

## How to read this article
This article goes step by step through the `mmap` syscall. I HIGHLY recommend that you follow along on your own as I don't put much of the actual code in this article. In order to follow along either start at the entrypoints mentioned, or, if you're skipping around, use [this](https://elixir.bootlin.com/linux/v5.14/source) wonderful website to search look up the symbols.

## Introduction to mmap
`mmap` is perhaps one of the most important syscalls in the Linux operating system as it is how memory mappings are formed. This is because it is through `mmap` that executables in the disc can load and run in memory. In this section I want to briefly discuss how `mmap` works from a user perspective. 

In order to discuss how `mmap` works, it is important to understand what a memory-mapping is. We will discuss how memory mappings are implemented later, but it is important to understand what a memory-mapping is conceptually. A memory-mapping is nothing more than a collection of pages that have a set of shared attributes (i.e. all the pages in the .text memory mapping might be readable and executable). Typically, these pages are virtually adjacent. 

Now that a memory mapping has been defined, let's discuss the `mmap` syscall from a user perspective.

`mmap` takes in 6 arguments and returns a `void*` pointer. Let's start by discussing the arguments. 

The first argument is `void* addr`. This essentially represents where to start the memory mapping. However, the kernel is under no obligation to follow this. According to documentation, if this argument is lower than the value located in `/proc/sys/vm/mmap_min_addr`or if there is already a memory mapping at that location, then the kernel will not follow this hint and instead return an entirely different pointer. However, if this value is valid, then the kernel will return a memory mapping at the nearest page-boundary (page-aligned address). 

The second argument is `size_t length`. This is simply the length of the memory mapping. Due to how paging works, this length will naturally be rounded up to be page-aligned. 

The third argument is `int prot`. There are 4 protection values: `PROT_EXEC`, `PROT_WRITE`, `PROT_READ`, and `PROT_NONE`. These represent what you can do to the memory mapping. `PROT_EXEC` means that the data in the mapping is executable, `PROT_WRITE` means that the data is writable, `PROT_READ` means that the data is readable, and `PROT_NONE` means that the pages are not accessible. `prot` works similarly to how `flags` works in the `open()` syscall with `O_RDONLY`, `O_WRONLY`, and `O_RDWR`. 

The fourth and perhaps most important argument is `int flags`. `flags` essentially describes certain attributes of the memory mapping. There are many attributes but only 3 that are important: `MAP_SHARED`, `MAP_PRIVATE`, and `MAP_ANONYMOUS`. These 3 flags can be split into 2 categories: sharing and file-backing. The first two can be considered "sharing" attributes while `MAP_ANONYMOUS` can be considered a "file-backing" attribute. Let's discuss what these mean. 

There are two kinds of memory mappings: file-backed and anonymous. The difference between these two kinds of mappings is actually fairly simple. File-backed mappings are mappings whose pages originate from a file (i.e. they are in `struct address_space`). These types of mappings are typically how executables are loaded into memory. 

The other kind of mapping is an anonymous mapping. An anonymous mapping is exactly the opposite of a file-backed mapping and contains pages that have no correspondence to any kind of file. These kinds of mappings are achieved by setting `MAP_ANONYMOUS` in `flags`. These kinds of mappings are most often used when creating the stack.

Due to the nature of Qogchamp, we will be dealing exclusively with file-backed pages.

The other kind of option is `MAP_SHARED` and `MAP_PRIVATE`. These two options mean different things when talking about file-backed or anonymous mappings.

In the case of file-backed mappings, adding `MAP_SHARED` means that changes to mapped pages will be written back to the disk. On the other hand, `MAP_PRIVATE` makes the pages "copy-on-write" meaning that they will be shared until they are written to at which point they will be copied to be private or anonymous. Pages in `MAP_PRIVATE` mappings will not be written back to the disk.

In the case of `MAP_ANONYMOUS`, `MAP_SHARED` means that pages in the anonymous mapping can be shared among processes. This allows two processes to concurrently write to the pages within the mapping much like two processes might concurrently write to a file. This can serve as a rudimentary form of IPC between a parent process and a child process. `MAP_PRIVATE` creates a mapping with pages that cannot be shared. In the case that the parent is forked, then the child receives a copy-on-write mapping which works much in the same that file-backed mappings have copy-on-write. 

I should note that a lack of `MAP_ANONYMOUS` implies that the mapping is file-backed. 

The fifth argument is `fd`. This is only valid in the case of file-backed mappings and essentially serves to inform the kernel on what file to map.

The final argument is `offset`. This is also only valid in the case of file-backed mappings and tells the kernel the offset within the file from which to begin the mapping. This `offset` must be page aligned or the kernel will return `EINVAL`.

## task_struct, mm_struct, and vm_area_struct

Before we begin discussion of the `mmap` syscall, we first need to discuss some essential structs: `struct task_struct`, `struct mm_struct`, and `struct vm_area_struct`.

At the top we start with `task_struct`. `task_struct` is undoubtly one of the most important structs in the Linux kernel as it represents a process within Linux. Within this struct there is pretty much everything you need to know about a task from open file-descriptors, running threads, PIDs, namespacess, and, of course, memory mappings. Memory mappings are located within the `mm_struct` which is located under the `mm` field in `task_struct`. 

The `mm_struct` has pretty much all the information about memory space of a process including page tables and memory mappings. We are obviously concerned with the latter. In Linux these memory mappings are represented as `vm_area_struct`s. In order to understand the `mmap` syscall, it is important to discuss what these `vm_area_struct`s contain. So, I have placed an abridged version of it below.

```c
struct vm_area_struct {

	unsigned long vm_start;

	unsigned long vm_end;

	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

	unsigned long rb_subtree_gap;

	struct mm_struct *vm_mm;

	pgprot_t vm_page_prot;

	unsigned long vm_flags;	

        .
        .
        .
        .

	const struct vm_operations_struct *vm_ops;

	unsigned long vm_pgoff;	

	struct file * vm_file;		
} 
```

As you can see, there are many important fields in `vm_area_struct`. Let's discuss them one by one. 

1. `vm_start` - The virtual address that this memory mapping begins at.

2. `vm_end` - The virtual address that this memory mapping ends at +1. In other words, the virtual address space of a `vm_area_struct` can be written as [vm_start, vm_end)

3. `vm_next` and `vm_prev` - Read below

4. `vm_rb` - Read below

5. `rb_subtree_gap` - Read below

6.  `vm_mm` - The `mm_struct` associated with this memory mapping

7. `vm_page_prot` - The page flags that the ptes within this memory mapping will use. As you might remember from the article on paging, the last 12 bits in a PTE are used to contain certain flags. This `vm_page_prot` specifies those flags. A specific list of these flags can be found [here](https://elixir.bootlin.com/linux/v5.14/source/arch/x86/include/asm/pgtable_types.h#L10).

8. `vm_flags` - This represents the access rights and properties for this memory mapping. These are analagous to the flags via `int prot` in the mmap syscall. A full list of these flags can be found [here](https://elixir.bootlin.com/linux/v5.14/source/include/linux/mm.h#L268).

9. `vm_ops` - This struct represents all the operations associated with this memory mapping. We will find this useful when we talk about page faulting.

10. `vm_pgoff` - The number of pages into the file that this memory mapping begins.

11. `vm_file` - The file backing this memory backing, if it exists.

A question that follows from this is how are these `vm_area_struct`s organized within a `mm_struct`? `vm_area_struct`s are organized in two primary ways: a red-black tree and a linked-list.

Let's start with the easy one: linked-list. Within each `vm_area_struct`, there are two fields `vm_next` and `vm_prev`. `vm_next` points to the next contiguous mapping (Or NULL if there is no mapping) while `vm_prev` points to the previous contiguous mapping (Or NULL if there is no mapping). This allows for O(N) traversal of memory mappings. However, as we will see this is not good enough.

`vm_area_struct`s are also organized in an red-black interval tree. This is so that we can search the `vm_area_struct`s in O(log(N)) time. We will find this to be very useful while doing the `mmap` syscall as it makes finding an open interval in which to place a new mapping that much easier.

This red-black tree is constructed via multiple parts. The first thing is the nodes. Each `vm_area_struct` has a node associated with it which is located at `vm_rb` within the `vm_area_struct`. These nodes are then organized as a standard interval rb tree with the interval being defined as [vm_start,vm_end). The root of this rb tree is located in `mm_struct` as the `mm_rb` member. 

In addition to the rb tree, each `vm_area_struct` also has a `rb_subtree_gap` which represents the "Largest free memory gap in bytes to the left of this VMA. Either between this VMA and vma->vm_prev, or between one of the VMAs below us in the VMA rbtree and its ->vm_prev". 

A combination of both of these helps greatly during the mmap syscall.

With all of these structures discussed, we can finally get into the mmap syscall.
 

## The actual mmap syscall
With an understanding on what the `mmap` syscall does in userland, we can begin discussing what the kernel does in the backend to accomplish this mapping. Once again, we can use [this](https://filippo.io/linux-syscall-table/) handy site to tell us where the entrypoint of `mmap` is. As it turns out, we find the entrypoint for `mmap` [here](https://elixir.bootlin.com/linux/v5.14/source/arch/x86/kernel/sys_x86_64.c#L96).

The first thing we see is an if statement which does `off & ~(PAGE_MASK)`. This check essentially checks if the offset is not page aligned. Mathematically, this calculation is functionally the same as `off % PAGE_SIZE != 0`. As we discussed earlier, this is where the syscall returns `EINVAL`. 

If that check is satisfied, we immediately jump to `ksys_mmap_pgoff` with the only change being that we do `off >> PAGE_SHIFT`. This makes offset in terms of pages instead of terms of bytes which is what the kernel tends to prefer.

Looking over `ksys_mmap_pgoff` we see that its really only performing some rudimentary setup. We start at the first if-statement. In the case that `MAP_ANONYMOUS` is not set (We have a file-backed mapping) we enter the if-statement. The first thing we do is call `audit_mmap_fd` which sets up the `audit_context`. This will become important later for checking permissions. 

We then call `fget(fd)`. This gets the corresponding `struct file` from the fd array as well as establishes a reference to it (So the file doesn't get dropped from under us). For those of you who don't know, the process control block within the Linux Kernel is known as the `task_struct`. Within this structure, there is everything that is necessary to represent a task. One of field within this structure is `struct files_struct`. Within `files_struct` there is another structure known as `fdtable`. Within this structure, we have a `struct file**` array (Array of `struct file*`). This is actually why file descriptors return as an integer from `open` - because they serve as an offset into this array. Within this array, we have a series of `struct file*`s each which serve as a representation of an open file.

I should note that there is a difference between `struct file` and `struct inode`. `struct inode` is a representation of a file on the disk which `struct file` is a process's view of an open file. Thus, there can be many-to-one mapping between `struct file`s and a `struct inode`.

If this does not return null (We have opened the file), then we continue. Following this check, we then check if the file is hugepages. I will not discuss hugepages in this post.  

We then do `flags &= ~MAP_DENYWRITE`. This essentially removes the `MAP_DENYWRITE` flag entirely. We do this because, according to documentation, `MAP_DENYWRITE` is ignored. 

With all of these checks done, we move to `vm_mmap_pgoff` with `fd` replaced with `file`. 

Within `vm_mmap_pgoff`, one of the first things we see is the `struct mm_struct`. This is one of the most important structs in the kernel as it overarching struct containing all the memory mappings for a given program. The primary idea of `mmap` is to create a new memory mapping within this `mm_struct`.

Within `vm_mmap_pgoff`, the first real thing we do is call `security_mmap_file`. This function calls the Linux security subsystem. Although this deserves an entire article of its own, the Linux subsystem essentially serves to support LSMs or Linux Security Modules. These LSMs serve as a mechanism by which certain access control policies such as MAC and DAC can be controlled at a kernel level. `security_mmap_file` essentially just calls these LSMs. There are many LSMs in use today including SELinux and AppArmor. 

If those checks pass, we can actually start performing the mmap. The first thing we do is acquire exclusive write permission on the mmap rw-semaphore by calling `mmap_write_lock_killable`. It is very important that this acquisition process is killable as if a fatal signal is received then the syscall should exit immediately. In the case that we do receive a non-zero output, we immediately return a `EINTR` signifying that the syscall was interrupted. I should note that due to the way that signals are handled and recieved by processes, this `EINTR` will never actually be received by the calling process. Rather, as soon as the syscall attempts to reenter userspace, it will be taken off the runqueue permanently at which point it will be effectively killed.

If all of this succeeds, then we enter `do_mmap` (The mmu version - obviously) - the true beginning of the mmap syscall. Within `do_mmap`, the first thing we do is perform a bunch of checks among other things. Below I have written a list of each check in order and why they are there.

1. We first check if `!len` or if `len == 0`. In this case, we are not mapping anything so just leave immediately. 

2. We then check the calling process's `personality`. This is a somewhat arcane field in `mmap` and essentially determines whether `PROT_READ` "implies" `PROT_EXEC`. The personality field is intended to allow Linux to emulate the behavior of syscalls on other flavors of UNIX. The allows some degree of cross-compatability between Linux and other flavors of UNIX and allows certain executables meant for those alternate versions of UNIX to be run on Linux. This personality field is set when the process executable is being loaded into memory. If the personality has `READ_IMPLIES_EXEC` and the `mmap` call has `PROT_READ` then `PROT_EXEC` is set as well (Assuming the filesystem is executable). For more information on this topic please refer to this documentation [here](https://elixir.bootlin.com/linux/v5.14/source/arch/x86/include/asm/elf.h#L280).

3. If `flags` has `MAP_FIXED_NOREPLACE` add `MAP_FIXED`. This will force the `mmap` syscall to place the mapping at the defined addr. `MAP_FIXED` gurantees that the mapping will be at the addr supplied while `MAP_FIXED_NOREPLACE` guarantees that it will not intersect with any other existing mapping. Thus, `MAP_FIXED_NOREPLACE` can be considered a superset of `MAP_FIXED`

4. If `MAP_FIXED` is not set, then attempt to round `addr` to the min address. In Linux there is a tunable constant known as `mmap_min_addr` which essentially sets the minimum address where memory mappings can occur. If, our supplied `addr` is less than `mmap_min_addr`, `round_hint_to_min` attempts to move `addr` as close to the min specified.

5. We then do `PAGE_ALIGN(len)`. This will make sure that we don't create a mapping that is not page aligned. We then make sure that we didn't overflow over the max `unsigned long` value. This overflow calculation itself works on a simple principle. The `PAGE_ALIGN` macro expands to the following calculation: `(len + (PAGE_SIZE-1)) & ~(PAGE_SIZE-1)`. Mathematically, this calculation can, at maximum, overflow to `PAGE_SIZE - 1` and `PAGE_SIZE - 1 & ~(PAGE_SIZE-1) = 0`. Thus, if `PAGE_ALIGN(len, PAGE_SIZE) = 0`, then we overflowed. If this is the case then return `ENOMEM`.

6. Next we make sure that the offset doesn't overflow. This calculation is a little confusing. As you might know, in the case of unsigned, if the maximum value is hit, then the value wraps around to 0. This is exploited here. In this section, we make sure that the total number of pages specified by length (`len >> PAGE_SHIFT`) and the total number of pages specified by the offset (`pgoff`) won't overflow or wrap around to `0`. This calculation works because at this point both `len` is page aligned and non-zero. So, `(len >> PAGE_SHIFT) + pgoff >= pgoff`. Thus, if `(len >> PAGE_SHIFT) + pgoff < pgoff` then we know we have a problem. If we do then return `EOVERFLOW`

7. We then check if the amount of mappings we currently have is greater than `sysctl_max_map_count`. This is another tunable parameter which limits how many memory mappings a process can have. In the case that it is return `ENOMEM`

With all of these checks complete, we can finally try to get some unused area. We do this by calling `get_unmapped_area`.

In `get_unmapped_area` the first thing we do is call `arch_mmap_check`. This only exists for a few architectures (Not including x86) and helps perform some architecture-specific checks. In the case of x86, this always returns 0.

We then make sure that `len` does not go over `TASK_SIZE`. `TASK_SIZE` is the maximum virtual address a userspace task can access. Thus, if `len` is greater than `TASK_SIZE` there is no conceivable mapping that can be created that is of size `len`

We then try to get the `get_area` function. This is the function that does most of the heavy lifting and will help us get the unmapped area that we are looking for. First, we assign `current->mm->get_unmapped_area` to `get_area` function. This, in our case, is most likely `arch_get_unmapped_area_topdown`. For more information on why this is look at the `get_unmapped_area` in the references section.

It is at this point our analysis splits into 2.5-ish paths: file-mapped backing, shared-anonymous backing, and everything else. 

If a file is inputted (Which we are assuming) then we use the fs-specific `get_unmapped_area`. Much like with `fsync` we will be analyzing `ext4`'s implementation. Looking into `ext4`'s `struct file_operations`, we quickly find that the `get_unmapped_area` is assigned to `thp_get_unmapped_area`. So, we can look into that. 

Looking into `thp_get_unmapped_area`, we start off by checking if the inode associated with the file is DAX. If it is not, then we call `get_unmapped_area`. This ends up calling `arch_get_unmapped_area_topdown`. We will assume that DAX is not being used.

In the case that an anonymous but shared mapping is used, we use `shmem_get_unmapped_area`. Looking into this function, we also see that it almost immediately calls `arch_get_unmapped_area_topdown` after checking that `len > TASK_SIZE` (Which is redundant).

In the case that we have anything else, we just call `arch_get_unmapped_area_topdown` directly.

So, as is evident, no matter what path we take we always somehow end up at `arch_get_unmapped_area_topdown`. So, let's discuss it.

Looking into `arch_get_unmapped_area_topdown`, the first thing we check is if we have `MAP_FIXED`. If we have `MAP_FIXED`, then we immediately return `addr`. This is fairly obvious as `MAP_FIXED` provides a guarantee that we place the memory mapping at the address provided. We then check if `len > TASK_SIZE` as such a request is literally impossible to fill. Thee next thing we check is if we are in a 32-bit or legacy syscall. If that is the case, then we take the legacy codepath. We will assume that we are not going to take the legacy codepath. 

The next thing we do, assuming we are not taking the legacy codepath, is that we handle the `addr` hint that was passed in as a syscall argument. More specifically, we check if it is even usable. The first thing we do is page align the `addr`. This makes the `addr` actually usable. The next thing we check is if the `addr` hint is actually valid. This check is done in `mmap_address_hint_valid`. This check really comes down to `addr` and `addr+len` both result in valid virtual addresses that can be mapped. A full description of what this function does can be found [here](https://elixir.bootlin.com/linux/latest/source/arch/x86/mm/mmap.c#L171).

In the case that the hint is not valid, we simply ignore the hint and we jump to the `get_unmapped_area` label which finds a valid mapping for us. If it is valid, we then execute `find_vma` on the address. `find_vma` is a very simple function which, according to documentation, finds the vma where addr < vm_end. We do this so that we don't return a hint address that is already within another vma. With this vma, we perform a few checks to see if the address hint is valid. 
 
We first check if the vma is NULL. If it is NULL then we are able to return `addr` as we are guaranteed that there does not exist a vma higher than this. If it is not NULL, then we perform the second check. This involves a function called `vm_start_gap`. `vm_start_gap` essentially returns the start of a vma. This is typically `vma->vm_start` except in the case of the stack (`vma->vm_flags & VM_GROWSDOWN`). In the case of the stack, we also need to subtract the stack guard pages which essentially prevent improperly recursing functions from infinitely running it down the address space. If the hint `addr+len <= vm_start_gap` then we can be sure that it does not intersect with the next highest vma. We do this because it is possible that `find_vma` could return a vma where `addr < vma->vm_start < vma->vm_end`.

In the case that we do not provide a hint or the hint is invalid, we move into an algorithm which finds an unmapped area for us. We start this section by creating a search-info struct with `.flags = VM_UNMAPPED_AREA_TOPDOWN`, `.length = len`, `low_limit = PAGE_SIZE`, `.high_limit = mmap_base`. The fields of this struct are fairly self-explanatory. The `flags` field specifies that we are using the non-legacy algorithm for finding the unmapped region, `length` specifies length, `low_limit` specifies the lowest point we can establish a mapping, and `high_limit` specifies the highest byte at which we can establish a mapping. 

After establishing this struct, we move into an if statement. This if-statement is mainly regarding "high addresses". By default, the `DEFAULT_MAP_WINDOW` is `1 << 47 - PAGE_SIZE`. However, in the case of 5-level page tables, we can map higher than that. So, we have a larger address space. This is best explained in the `DEFAULT_MAP_WINDOW` reference. Although its ARM, it is valid. 

Following the if statement, we setup some more portions of the info struct such as alignment, and with that, we are ready to enter the function `vm_mapped_area`. I will not go over this function, but it essentially traverses the rb-interval tree until a valid, unmapped region is found. This address is then returned. If the address is not valid then we jump into the legacy algorithm. 

From here, we return to `do_mmap` and we continue. 

Looking at `do_mmap`, we first check if the returned value is an error value. If it is, then return immediately.

Following this check, we then perform a check with `MAP_FIXED_NOREPLACE`. As we discussed earlier, `MAP_FIXED_NOREPLACE` mandates the newly created vma not intersect with 
an already existing `vma`. So, if it does then we return an error value. 

We then check if our provided protection is exclusively `PROT_EXEC` (The pages cannot be read nor written). If we do, then we create what is known as a protection key. I'm not going to go into this feature here, but you can find information about it both online as well as in the Intel Manual. 

From here, we perform 2 operations. The first thing we do is calculate the `vm_flags` field for our `vm_area_struct`. This essentially takes our `prot` argument and turns transforms it into the proper `vm_flags` which will be used from here. In addition to getting the `prot` flags, we also append `VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC` in order to indicate that the vma could be made readable, writable, or executable in the future via `mprotect`. The second thing we do is, if prot contains `MAP_LOCKED`, we check if we are at all able to lock the mapping. If we cannot lock the mapping then we return an error. 

If all of these checks pass, we reach yet another fork in the road. Here we need to do some file/anonymous-mapping-specific operations. Let's start first by discussing what happens if it is a file-backed mapping. 

If it is a file-backed mapping, we first retrieve the associated inode. With this inode we enter `file_mmap_ok`. What this function essentially does is make sure that the file can actually be mapped. This works by essentially checking that a combination of `len + (pg_off << PAGE_SHIFT) <= max_file_size`. This essentially serves as a simple check to make sure that the file can be mapped at all. If this check passes, we are able to move on and we can be sure that the file can at least theoretically be mapped.  

From here, we move into a switch statement. There are three choices here: `MAP_SHARED`, `MAP_SHARED_VALIDATE`, and `MAP_PRIVATE`. 

1. If the mapping is `MAP_SHARED`, we remove the unsupported flags via `flags &= LEGACY_MAP_MASK` and we fall through to `MAP_SHARED_VALIDATE`. This originates from the man page of `mmap` which says that `MAP_SHARED` silently ignores unsupported flags. 
2. The second option (Or as a fallthrough from `MAP_SHARED`) is `MAP_SHARED_VALIDATE`. In this section we perform quite a few permission checks.  
	1. The first set of checks we perform is with `prot & PROT_WRITE`. With `PROT_WRITE` we first check if the file descriptor was opened with `FMODE_WRITE`. If it was not, then we do not have write permissions and thus we error. The second set of checks we perform is making sure that the inode is not a swapfile as we cannot `mmap` a swapfile. 
	2. The next check we perform is making sure that the file is not append-only. We do this because in an append-only file, written data is immutable. So, if we were able to `mmap` and write, we would inherently be breaking that principle as with `mmap` we would be able to write anywhere in the file. 
	3. The third check we make is making sure that there are no pending locks on the file as if there are, we will not be able to do any work on the file
	4. From here, we come upon an interesting 3 lines. We see that the kernel adds `VM_SHARED | VM_MAYSHARE`, and, if the file descriptor is not writable, it takes off `VM_SHARED | VM_MAYWRITE`. This indicates that the Linux Kernel seems to treat non-writable shared mappings as equivalent to private mappings. This makes sense for reasons we will see in a future article. This also brings up a concern with `mmap`'s interaction with `fcntl`. With `fcntl`, the file can be made writable. However, how does with impact this vma as `VM_MAYWRITE` has been taken off. Ideally, `fcntl` would go through all the mappings and add `VM_MAYWRITE`, but I don't know. 
	5. With all of these checks complete, we then fallthrough into `MAP_PRIVATE`
3. Looking into `MAP_PRIVATE`, we perform our final set of checks. 
	1. The first check we perform in `MAP_PRIVATE` is making sure that we can read the file descriptor. If we cannot read the file then there is no point in mapping it. So, return an error.
	2. We then check if the file is not executable. If the file is not executable and we are requesting an executable mapping, we return an error as we cannot satisfy such a request. We also remove `VM_MAYEXEC` as the file cannot be executable. 
	3. We then make sure that the `file_operations` struct of this file descriptor has a `mmap` function. This is very important for later in the `mmap` syscall.
	4. We finally make sure that `VM_GROWSDOWN` and `VM_GROWSUP` are not set. Both of these flags are used to indicate that the vma is the stack. Obviously, files cannot be the stack, so such a request would be impossible to satisfy.

In the case that we have an anonymous mapping, we perform a small set of simple checks. 

1. If the anonymous mapping is shared, we begin by making sure that `VM_GROWSDOWN` and `VM_GROWSUP` are not set. This is because the stack can NEVER be shared. If this check passes, we set `pgoff` to 0 and we append `VM_SHARED` and `VM_SHARED` to vm_flags
2. If the anonymous mapping is private, we set `pgoff` to `addr >> PAGE_SHIFT` or `addr` in page units. This `pgoff` will be useful when we configure the `anon_vma`.

Once these checks are big checks are complete we perform one final check with `MAP_NORESERVE`. According to documentation `MAP_NORESERVE` essentially demands that there is no swap space allocated for this vma. So, it is possible that pages in this mapping could be dropped if memory usage is high. We only honor `MAP_NORESERVE` if we are allowed to overcommit memory (Use more memory than we have access to). If we are not allowed, then we cannot honor this `MAP_NORESERVE`. 

Once we have FINALLY finished all of these checks we move into `mmap_region` which is essentially the home stretch for `mmap`. It is in this code that the actual `vm_area_struct` is setup. 

Looking into `mmap_region`, we start off with some fairly standard initialization with a `mm_struct`. 

The first check starts by running `map_expand_vma`. This essentially makes sure that we can accommodate this mapping. In other words, we want to make sure that we can map sufficient pages in order to complete this vma. If this returns false, we have another course of action. As we discussed earlier, a `MAP_FIXED` vma can intersect with other vmas. So, we can subtract the overlapping pages and recheck. If we still do not have sufficient pages, we simply return an OOM error. 

If this check passes, we then call `munmap_vma_range`. What this does is remove any overlapping mappings. As we discussed earlier, `vm_area_struct`s CANNOT be overlapping. So, if we are using `MAP_FIXED` and overlap, the old mapping needs to go. 

Next, we perform another check. If we have a private, writable mapping, we make sure that we have sufficient pages to support this mapping. If we do not, then we do not have enough memory to satisfy this request.

Once all of these checks we passed we go into `vma_merge`. This function essentially checks if we can merge with an already existing mapping. This is often very useful in something like `brk` where the new mapping is guaranteed to be adjacent to the old mapping. I'm going to go into the algorithm used here, but the function is well documented and I encourage you to read into it. If `vma_merge` successfully returns a vma, we are permitted to exit our immediately. If not, we must proceed as normal. 

Assuming that `vma_merge` did not work, the first thing we do is allocate a new vma via `vm_area_alloc`. Looking into `vm_area_alloc` you may notice that we are not using `kmalloc` or any of its variants. This is because vmas get allocated via a slab allocator which is often used for commonly allocated structures such as `struct page` and `struct vma_area_struct`. 

If the vma was successfully allocated, we then initialize the basic fields in the vma such as `vm_start`, `vm_end`, and `vm_flags`. 

Following this we reach another fork in the road at the if statement. This time we have three options: file-backed mapping, shared anonymous mapping, and private mapping. 

1. If the mapping is a file-backed mapping, we have to perform some accounting
	1. The first thing we do is in the case the mapping is shared. If the mapping is shared then we increment the mapping->i_mmap_writable. This, contrary to what the name suggests, is a counter of how many shared mappings this file has. 
	2. The next thing we do is run `call_mmap`. This function calls `file_operations->mmap`. This gives devices and filesystems the oppurtunity to perform specific functions tailored to their needs. In the case of qogchamp, we will be mapping a regular ext4  file so we will be using `ext4_file_mmap`. Looking into `ext4_file_mmap`, we see that wee perform DAX things and, more importantly, we set the `vm_operations_struct` for this mapping. This will be important when we discuss page faulting. 
	3. Once that check is done, we recheck to see if we can merge with an existing mapping. We perform this check again because `file_operations->mmap` can actually change `addr`. 
	4. Finally, update `vm_flags`. We do this because, much like with `addr`, `file_operations->mmap` can change `vm_flags`
2. The second case is if we have a shared anonymous mapping. If we have shared anonymous mapping, we simply run `shmem_zero_setup`. This function sets the `vma->vm_file` to `/dev/zero`. This ensures that, when we page fault, we are guaranteed to get a zeroed page. This function also sets `vm_ops` with `shmem_vm_ops`
3. Finally, in the case that we have a private anonymous mapping, we simply run `vma_set_anonymous` which sets `vm_ops = NULL`. 

After going through that, we perform yet another check with `arch_validate_flags`. This particular function is exclusive to certain architectures like SPARC. On x86, it simply returns true. 

Once we finish `arch_validate_flags`, we call `vma_link`. This function accomplishes a few things.

Firstly, if we have a file, it takes the lock on the backing `address_space`. Then, it calls `__vma_link`. This function accomplishes two things. Firstly, it adds the newly created vma to its appropriate place in the vma linked list. Secondly, it adds the newly created vma to the rb-interval tree that is located in `mm_struct`. 

We then call `__vma_linked_list`. This function is a nop for any anonymous mappings, but if we have a file-backed vma then this function adds the vma to the `i_mmap` interval tree. As I talked about in the page cache article, each `address_space` contains an interval tree of every vma that maps the backing inode. This is where the vmas are added to that `i_mmap`.

Once we complete both of those functions, we unlock our lock and we increment `mm->map_count`. Then we run `validate_mm` which essentially makes sure that the vmas in `mm_struct` are consistent with the `mm_struct` itself. 

## Conclusion


## What's Next
Next, we will be discussing 










## References
`mmap` - <https://man7.org/linux/man-pages/man2/mmap.2.html>

`task_struct` and `personality` - <https://www.spinics.net/lists/newbies/msg11186.html> and <https://man7.org/linux/man-pages/man2/personality.2.html>

`vm_area_struct` - <https://elixir.bootlin.com/linux/v5.14/source/include/linux/mm_types.h#L311>

`ext4` `struct file_operations` - <https://elixir.bootlin.com/linux/v5.14/source/fs/ext4/file.c#L912>

`get_unmapped_area` - <https://elixir.bootlin.com/linux/v5.14/source/arch/x86/mm/mmap.c#L132>

`TASK_SIZE` - <https://elixir.bootlin.com/linux/v5.16/source/arch/x86/include/asm/page_64_types.h#L64>

`DEFAULT_MAP_WINDOW` - <https://patchwork.kernel.org/project/linux-mm/patch/20181114133920.7134-3-steve.capper@arm.com/>

