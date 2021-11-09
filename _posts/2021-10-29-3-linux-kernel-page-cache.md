---
title: 3 - Linux Kernel Page Cache
date: 2021-11-8 15:40:00
categories: [QogChamp]
tags: [Linux Kernel Exploitation, Page Cache]
---
One of QogChamp's capabilities is performing "ghost file operations" or file operations that are easy to clean up and have little to no footprint. The page cache is the main structure used to pull this off. 


## What is a Page Cache?
As you might know, block devices (Disks/SSDs) are EXTREMELY slow compared to RAM and CPU. 

Now, this is a big problem, especially for read/write operations, as every time a program wants to read/write to/from a file, it needs to wait for the disk to complete the operation. This, due to the slow nature of the block devices, can burn CPU cycles and force programs to block for extended periods while waiting for the disk to complete. In the long run, this is bad and can greatly extend the runtime of certain programs that perform file operations (So basically, almost every program). 

So, how do we solve this problem? As it turns out, the solution is surprisingly simple: make a copy of the file in memory. This makes file operations significantly faster as instead of constantly asking the disk, we simply perform our file operations (read, write, mmap, etc.) on memory which is much faster. We call this in-memory representation the **page cache**.

A common question that might follow from this is if we're doing all our file operations on the page cache, how do changes get synced with the disk?

To sync changes, the Linux Kernel undergoes a process known as deferred write. With deferred write, the kernel waits until a good time to actually sync the changes with the disk. This is much faster than writing back immediately for one main reason: it allows us to do async IO. 

Assuming that the user is not doing async IO, when a syscall like `write()` returns, the caller expects the buffer to be written to the disk. Therefore, if we are not using a page cache, we would have to wait for the write to the disk to complete, which can often be extremely slow. By using the page cache, we can give the user the illusion that the buffer was written to the disk via page cache (Which is much faster), and later, we can asynchronously write it back to the disk. Thus, we can return control more quickly to the user program in cases like `write()` without giving up any functionality, thus making the page cache a very fast and powerful method.

However, there are issues with the page cache. Because changes are written back to the disk later, those changes could be sitting in memory, unsynced for long periods. This can cause issues as if a computer crash occurs at an inopportune time, changes could be lost. Because of this, many programs like databases that cannot afford this loss of data use `sync()` frequently or use the `O_DIRECT` flag when writing, both of which start writeback immediately. 

The nature of deferred write is immensely important for Qogchamp and is something that I will get into when discussing mmap.

## address_space
As we discussed in the last post, the main structure used to handle the page cache is `struct address_space`. Let's go into more detail into what the structure contains. 

```C
struct address_space {
	struct inode		*host;
	struct xarray		i_pages;
	gfp_t			gfp_mask;
	atomic_t		i_mmap_writable;
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
	/* number of thp, only for non-shmem files */
	atomic_t		nr_thps;
#endif
	struct rb_root_cached	i_mmap;
	struct rw_semaphore	i_mmap_rwsem;
	unsigned long		nrpages;
	pgoff_t			writeback_index;
	const struct address_space_operations *a_ops;
	unsigned long		flags;
	errseq_t		wb_err;
	spinlock_t		private_lock;
	struct list_head	private_list;
	void			*private_data;
}
```

Above is the `struct address_space` from the 5.14 Linux Kernel. There are a few important fields.

 The first field is `host`. `host` represents the inode that the page cache is mapping. This can be thought of as the file that the page cache is mapping into memory. I should note that `struct inode` also references the `struct address_space`.

 The next field of importance is `i_pages`. `i_pages` is a special kind of array (Don't ask me how. That's beyond me) that contains pointers to `struct page`s. These `struct page`s are the descriptors for the pages that are the in-memory representation of the file. In other words, this is the meat of the page cache. By kmapping the `struct page` pointers from `i_pages`, you will get direct access to the in-memory representation of the file. This is critical to getting the exploit to work.

 To traverse the `struct xarray`, there are a suite functions in [xarray.h](https://elixir.bootlin.com/linux/latest/source/include/linux/xarray.h) that are available. These functions are used throughout the page cache code. 

 The last field that is important is `const struct address_space_operations *a_ops`. If you're familiar with the Linux Kernel, this should be a familiar sight. This struct contains the "member functions" of the `address_space` object - yes, member functions in the same sense as Java. We will continually revisit this function as we discuss specific syscalls.

In order to better demonstrate how `struct address_space` works with other Linux Kernel data structures, I have placed a fantastic image below.

![Page Cache](/assets/img/3/page_cache.jpeg)

In the picture, the `struct page` pointers are stored in `i_pages` and these `struct page`s are mapped into various VMAs throughout the system. These various VMAs also contain `struct file`s which then point back to the `struct address_space`.

## What's Next
From here, I want to go into how exactly `read`, `write`, `mmap`, and `sync` work as the knowledge gained from deeply studying these three syscalls will be critical in understanding how exactly this exploit works. 

## References
Information on Page Cache - <https://courses.cs.vt.edu/~cs5204/fall15-gback/>

More Information on Page Cache - Understanding the Linux Kernel 3rd Edition by Daniel P. Bovet and Marco Cesati - Chapter 15

`struct address_space` - <https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L459>



