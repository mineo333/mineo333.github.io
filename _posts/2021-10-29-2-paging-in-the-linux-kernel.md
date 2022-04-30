---
title: Paging in the Linux Kernel
date: 2021-10-29 12:30:00
categories: [QogChamp]
tags: [Linux Kernel Exploitation, Paging, MM]
---

Before I begin discussing the internals of the QogChamp exploit, I first need to discuss how the Linux kernel handles paging. 

This article will assume familiarity with paging as a concept. 

## Quick Refresher on Paging
Paging is how we establish a mapping that turns linear or virtual addresses into physical addresses. By using paging, we can do a variety of things such as isolate address spaces as well as prevent external fragmentation of memory. 

In order to turn a linear address into a physical address, we need to do a page table walk as described below from Intel Manual Volume 3:

![Paging Walk Diagram](/assets/img/2/page-walk.png)

Note that the directory pointers have 9-bit offsets instead of 10-bit offsets. If you're used to programming 32-bit paging, then this might take you by surprise. As we will see later, the paging data structures are 8 bytes, so we can only fit 4096/8 = 512 = 2^9 paging data structures in a given directory. Thus, it is sufficient to have a 9 bit offset instead of a 10-bit offset. However, we still need a 12 bit offset into the frame. 

## Paging in the CPU

Before I discuss paging in Linux, it is first important to get a grasp of how paging is handled in the CPU. 

In order to get an understanding of paging in the CPU, we can reference the trusty Intel Developer Manual Volume 3. 

![Paging Data Structures](/assets/img/2/paging-structures.png)

Above is a picture of all the paging data structures in IA-32e (64 bit). There are 4 data structures that concern us: the CR3, the PDPTE, the PDE page table, and the 4 KiB page.

CR3 is a control register in the CPU that always points to the first PDPTE. This is where the page walk is started. The CR3 is swapped out at every context switch in order to enforce different address spaces.

The PDPTE is the first level of paging and contains a 12-bit aligned pointer to a page of PDEs. Typically, we use the first set of 10-bits in the linear address as an offset from the address in CR3 to find the relevant PDPTE.

The PDE is the second level of paging and contains a pointer 12-bit aligned pointer to a page of PTEs. Typically, we use the second set of 10 bits in the linear address as an offset from the address in the PDPTE to find the relevant PDE.

The third and final level of paging is the PTE. The PTE contains a 12-bit aligned pointer to a page frame. Typically, we use the third set of 10 bits in the linear address as an offset from the address in the PDE to find the relevant PTE.

The last 12 bits of a linear address are used to search through the 4 KiB page frame.

Another thing that you might notice is that in every paging data structure, the last 12 bits have a bunch of entries. These are known as flags and contain information about the state of the PTE. The one flag that concerns us is the dirty flag (Marked as D). This dirty flag is set to 1 by the hardware whenever the corresponding page frame is written to.

## Paging in the Kernel

Now that we have discussed paging as a concept and paging in the CPU, we can finally discuss paging in the Linux Kernel. 

For the kernel, the PTE data structure does not provide sufficient information about a page. This is because, in the Linux Kernel, pages are associated with various additional structures such as page caches, slabs, and page pools. So, the kernel needs a place to put metadata regarding these structures. This is where `struct page` comes in. `struct page` is the Linux Kernel's own representation of pages. In contrast to the PTE, the `struct page` is quite large, with each structure being 64 bytes. 

These page structures contain a variety of extremely important fields, each of which deserves its own lengthy articles, but only a few concern us for the purposes of QogChamp. 

The first field of importance is: 
```C
unsigned long flags
```

This is an unsigned 64 bit integer that contains the flags of a given page. The purpose of this field is to extend the flag portion of the PTE and provide a kernel developer with a much higher degree of control as well as more options on how to label their pages.

The possible flags that can go into `unsigned long flags` are detailed in `page-flags.h`. Below I have listed a shortlist of all the possible flags, but if you're interested in the current flags that a page can have please check it out [here](https://elixir.bootlin.com/linux/latest/source/include/linux/page-flags.h).

```c
enum pageflags {
	PG_locked,		/* Page is locked. Don't touch. */
	PG_referenced,
	PG_uptodate,
	PG_dirty,
	PG_lru,
	PG_active,
	PG_workingset,
	PG_waiters,		/* Page has waiters, check its waitqueue. Must be bit #7 and in the same byte as "PG_locked" */
	PG_error,
	PG_slab,
	PG_owner_priv_1,	/* Owner use. If pagecache, fs may use*/
	PG_arch_1,
	PG_reserved,
	PG_private,		/* If pagecache, has fs-private data */
	PG_private_2,		/* If pagecache, has fs aux data */
	PG_writeback,		/* Page is under writeback */
	PG_head,		/* A head page */
	PG_mappedtodisk,	/* Has blocks allocated on-disk */
	PG_reclaim,		/* To be reclaimed asap */
	PG_swapbacked,		/* Page is backed by RAM/swap */
	PG_unevictable		/* Page is "unevictable"  */
};
```

Out of these flags, only a few concern us. The first is `PG_uptodate`. `PG_uptodate` signifies that the page's content is current and matches that on the disk. We will discuss what this means in greater detail as we discuss page and buffer caches. 

The second is `PG_dirty`. `PG_dirty` is a flag that signifies that the page has been written to. It is *very* important to note that this flag is in no way correlated to the dirty bit on the PTE. The dirty bit on the PTE is set by the CPU automatically when a write occurs. `PG_dirty` is set by the kernel (Software). We will discuss later when we get into the details of QogChamp about why this distinction is important as well as under what conditions `PG_dirty` is set.

The third bit that is important is `PG_private`. We will get into this more when we get into the page cache, but what it means is that the private field in the page struct contains a series of `struct buffer_head`s. 

The fourth bit that is important is `PG_mappedtodisk`. All this flag signifies is that the contents of the page correlated to blocks on the disk. We will get into this a lot more as we discuss page cache. This flag is usually when a page is read from disk.

The fifth bit is `PG_writeback`. `PG_writeback` represents pages that are currently under some disk IO. Pages under writeback should not be modified.

The sixth bit of importance is `PG_locked`. Contrary to what it sounds like, `PG_locked` does not lock the actual memory that the `struct page` describes. Rather, it is a lock on the fields in a `struct page`. Thus, any process wanting to modify a `struct page` should hold the lock, and this flag should be set.

The next field (Or set of fields rather) is the anonymous struct that contains the page cache information:
```c
struct {	/* Page cache and anonymous pages */
			/**
			 * @lru: Pageout list, eg. active_list protected by
			 * lruvec->lru_lock.  Sometimes used as a generic list
			 * by the page owner.
			 
			struct list_head lru;
			/* See page-flags.h for PAGE_MAPPING_FLAGS */
			struct address_space *mapping;
			pgoff_t index;		/* Our offset within mapping. */
			/**
			 * @private: Mapping-private opaque data.
			 * Usually used for buffer_heads if PagePrivate.
			 * Used for swp_entry_t if PageSwapCache.
			 * Indicates order in the buddy system if PageBuddy.
			 */
			unsigned long private;
};
```

The first field in this anonymous struct is the LRU. The LRU is a very important data structure that deals with page eviction. To my very limited understanding of LRU, there are two lists: active and inactive. Pages in the active list have been accessed very recently, and pages in the inactive list have not been accessed recently. Based on this LRU list, the Linux kernel makes heuristic decisions on whether to flush or keep any page. The LRU list will not be of concern for QogChamp. 

The second field is `struct address_space`. This is a particularly interesting field as it can actually be two things. If the page is file-mapped, then the `struct address_space` field will point to a `struct address_space`, which contains information on file-mapped shared pages. However, if the page is anonymous (Not file-backed), then it will actually point to a `struct anon_vma` as evidenced in [this](https://elixir.bootlin.com/linux/v5.14/source/mm/rmap.c#L1068) function, which is a descendant of the anonymous page fault handler. `struct anon_vma` is used to keep track of shared anonymous mappings. Only `struct address_space` will be of concern for QogChamp.

The third field is `index`. The index field is used to describe the page's index within the `mapping` field. It works almost in the same way that an index into an array works. 

The final field is `private`. The `private` field is only set if the `PG_private` flag is set. If `PG_private` is set, the `private` field will contain a pointer to the `struct buffer_head`s. This is something that we will go into when we discuss page caches. 

The next two fields within the `struct page` of importance are `_mapcount` and `_refcount`. `_refcount` is exactly what it sounds like - the number of references held to the `struct page`. Typically, this is increased and decreased via the `get_page` and `put_page` functions, respectively. 

`_mapcount`, on the other hand, is a counter of how many times the page has been mapped to userspace (i.e., how many page tables the page is in). Put simply, `_refcount` is the number of references held to the `struct page`, and `_mapcount` is the number of virtual references held to the page. The page eviction code often uses these two variables to see when a page is safe to evict. These two fields will not be very important, but I felt that I should mention them because they are such ubiquitous variables.


## What's next

From here, I will try to explain the page cache and how it functions as well as the syscalls that will be important. 

## References
Page walk image - Intel Developer Manual Volume 3A page 4-20

Paging data structures image - Intel Developer Manual Volume 3A page 4-18

`struct page` - <https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L70>

`anon_vma` - <https://lwn.net/Articles/75405/>

`_mapcount` and `_refcount` - <https://stackoverflow.com/questions/48931940/how-do-pinned-pages-in-linux-present-or-actually-pin-themselves>

