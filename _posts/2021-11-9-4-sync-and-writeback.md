---
title: 4 - sync and Writeback
date: 2021-11-20 22:40:00
categories: [QogChamp]
tags: [Linux Kernel Exploitation, Syscall]
---

In this article, I want to describe the `sync` syscall. By describing the `sync`, I hope to describe how changes to files are transferred from the page cache to the disk. This will give critical insight into how Qogchamp works.

## How to read this article
This article goes step by step through the `msync` and `fsync` syscalls. I HIGHLY recommend that you follow along on your own as I don't put much of the actual code in this article. In order to follow along either start at the entrypoints mentioned, or, if you're skipping around, use [this](https://elixir.bootlin.com/linux/v5.14/source) wonderful website to search look up the symbols.

## Entrypoint
In order to begin analyzing `sync`, it is important to find the entrypoint. Luckily, we can use [this](https://filippo.io/linux-syscall-table/) wonderful site that describes all the entrypoints for each syscall and their registers. 

Looking at this site, we find three kinds of `sync` - `msync`, `fsync`, and plain `sync`. For ease, we will analyze both `msync` and `fsync`. We won't analyze `sync` because it wakes up the flusher threads, which makes it somewhat difficult to analyze.

## msync vs. fsync

I briefly want to go over the difference between `msync` and `fsync` at a high level before we get into the specifics. 

There are two ways to open and modify files for those unaware: `write` and `mmap`. 

In the case of `write`, we simply open the file using `open`, and then we use `write` along with the returned file descriptor. In the case of `write`, as we will see, the changes are written directly to the backing `struct address_space` of the inode. In order manually to force the contents of the written file back to the disk, we use `fsync`, which takes in the file descriptor received from `open`.

In the case of `mmap`, we map the file into the process's address space. Meaning, we can access and modify portions of the file in the same way we might access portions of stack or heap. These modifications are then written back to the page cache. `msync` is used to force back those changes to the disk. 

## fsync

Using the site mentioned above, we can find the entry point of `fsync`, which happens to be in fs/sync.c which can be found [here](https://elixir.bootlin.com/linux/v5.14/source/fs/sync.c#L230). In order to find the actual syscall within the file, we can look for the `SYSCALL_DEFINEx` macro.

Quickly, we find that we enter a function called `do_fsync` with the `fd` and another argument `datasync` being 0. `datasync` is only set to 1 if `fdatasync` is called. 

Within `do_fsync` we call `fdget` which returns a `struct fd`. Those familiar with Linux file operations might recognize this function. Essentially, within the process control block of Linux - `struct task_struct` there is an array of `struct fd`s containing information on opened files. Using this integer as an index into this array, we get the relevant `struct fd`. If you're particularly interested in these functions, I recommend reading [this](https://www.kernel.org/doc/Documentation/file-systems/files.txt).

Using this `struct fd`, we enter another function known as `vfs_fsync`. Anyone familiar with Linux file operations might be familiar with the `vfs_` prefix. This is derived the idea of the virtual file system. In the Linux Kernel, we have a generic high-level file-system which functions as the abstraction above concrete file systems (i.e. ext4). As we'll see, these `vfs_` functions typically eventually call file-systems-specific functions. 

In `vfs_fsync` we call `vfs_fsync_range` which is used to sync portions of a file. We use 0 and `LLONG_MAX` as the start and end, respectively, to sync the entire file. From here, we move into file-system-specific operations. 

In `vfs_fsync_range`, we begin with two if-statements. The first if-statement checks if the file system has a fsync member function. If not, then it errors. This happens because the virtual file system needs a concrete `fsync` function provided by the file-system in order to perform `fsync`. 

The second if-statement is a little more peculiar. The second if-statement is where `datasync` becomes useful. If `datasync` is 0 (Which it is), then we essentially update the inode metadata and write that new metadata to the disk. If `datasync` is 1, then we do not update the metadata, and we only sync the contents of the file. 

When those two checks are complete, we finally call the file-system-specific `fsync`.

Unfortunately, because of how many file-systems there are, I will not be able to cover the `fsync` implementation for every single one. So, I will only focus on ext4's implementation, which can be found [here](https://elixir.bootlin.com/linux/v5.14/source/fs/ext4/file.c#L912). Looking at the `fsync` field, we find that it points to `ext4_sync_file`. This is what is called by `vfs_fsync_range`.

## File-system-Specific fsync
In `ext4_sync_file`, we start out by checking if the superblock of the file-system is read only. If it is then we leave (`goto out`). This is fairly self-explanatory as we can't write to a read-only file-system. 

After that check is completed, we immediately jump into `file_write_and_wait_range`, which actually happens to be a VFS function. In `file_write_and_wait_range`, we come across a familiar data structure: `address_space`, which, as you might recall, is the page cache data structure. As might be obvious, we are going to write the pages from `address_space`. 

The first thing that we check in `file_write_and_wait_range` is if the page-cache has any pages at all. This is done by running the function `mapping_needs_writeback`. This is an important check because if the page cache has no pages, then what's the point of starting writeback?

If the page cache for the inode does indeed have pages, we run `__filemap_fdatawrite_range`. `__filemap_fdatawrite_range` is run with four parameters. The first three are self-explanatory. The last one is interesting, however: `WB_SYNC_ALL`. What this flag does is that if a page in the page cache is already under writeback, then it should wait for the writeback to finish and then redo the writeback. The alternative to this is `WB_SYNC_NONE`, which does not wait for writeback to complete and instead skips any page currently under writeback. This distinction is best described in the documentation for `__filemap_fdatawrite_range`, which says that "The difference between \[ `WB_SYNC_NONE` and `WB_SYNC_ALL` \] is that if a dirty page/buffer is encountered, it must be waited upon, and not just skipped over." We will see these flags in the code down the road. 

Looking into `__filemap_fdatawrite_range` we see that we setup something called a `writeback_control`. This struct essentially decides how writeback will be done. There are 4 fields populated, but one that is new: `nr_to_write`. This represents the number of pages to writeback. This is set to `LONG_MAX`, indicating that we should writeback all pages (Which makes sense). Once that is set up, we jump into `do_writepages`. 

Within `do_writepages`, we arrive a fork in the road. If the file-system has an implementation for `writepages` in its `address_space_operations` then use that. If not, then use a `generic_writepages`. Looking into ext4's `address_space_operations`, we see that ext4 has an implementation for `writepages` in `ext4_writepages`. So, we jump to that function. 

## writepages

Upon reading `ext4_writepages`, we see a daunting block of code. However, upon closer analysis, it becomes obvious that much of this code is for dealing with special cases of writeback - specifically three cases: cyclic writeback, inline file writeback, and non-journaled writeback. However, for ease, we will assume that we take the standard writeback procedure which uses journaling

Assuming we are performing journaling, we start at line 2671. Here, we first check if the file is being journaled. If it is (Which we are assuming), then we jump to `generic_writepages`. 

Within `generic_writepages`, we start by looking checking if the file-system's `address_space_operations` has a `writepage` implementation. If it doesn't then leave. This `writepage` function is what actually writes the new file buffer to the disk. So, without a `writepage` it is physically impossible to push data to the disk. If `writepage` exists, we then run a function called `blk_start_plug`. According to the documentation, all this does is warm up the block device and block io layer and warn them that they will be receiving IO requests shortly. From here, we jump to `write_cache_pages`. However, before we go into `write_cache_pages`, we need to analyze one of the parameters - specifically `__writepage`. 

`write_cache_pages` requires a parameter that is of type `writepage_t`. This is a pointer to a `writepage` function.  In this case, we use `__writepage` which essentially serves as a wrapper of the FS-specific implementation of `writepage`. 

Looking into `write_cache_pages` we find a very large block of code. However, once again, much of this code is dealing with edge cases which we don't particularly care about. The first real line of consequence (And perhaps the most important line in the entire `fsync` implementation) happens on line 2197. This is the line where the magic of QogChamp is defined. Assuming that the sync mode is `WB_SYNC_ALL` (Which it is), then we will jump into `tag_pages_for_writeback`. 

In `tag_pages_for_writeback`, the kernel essentially searches through the page cache and tags all the dirty pages with a special tag that says that it needs to be written back. How does it do this? The answer hearkens back to the `i_pages` xarray structure within the `address_space` object. What it does is that it uses a special macro to find all entires that are marked with `PAGECACHE_TAG_DIRTY`. Every entry marked with this tag is then marked with `PAGECACHE_TAG_TOWRITE`, indicating that it should be written to the disk. 

There are two questions posed from this. First, why do this? The main reason for doing this is written in the documentation for `tag_pages_for_writeback`. We do this additional marking in order to prevent livelocking, which could be caused by a process writing to the pages as we're trying to do the writeback. Second, where does the `PAGECACHE_TAG_DIRTY` tag come from, and when is it set? This is the answer to this question that holds the key to why QogChamp works. This topic will be discussed further as we discuss the `mmap` and `write` implementations. 

Once `tag_pages_for_writeback` returns, it can be assumed that all the pages ready for writeback are tagged and ready to go. From here, it's just a matter of iterating through the pages and sending them on their way to the disk. To writeback, the code does three checks:

1. It checks if the page has been truncated under us. If so, then leave because a truncated page cannot be written back. This is accomplished by checking if the `address_space` associated with the page is still valid. 

2. It checks if the page is no longer dirty. If the page is no longer dirty, then the page has been written back by someone else.

3. It checks if the page is already currently under writeback. If it is, then it waits for it to not be under writeback and then either continues or redoes the write depending on whether `WB_SYNC_ALL` or `WB_SYNC_NONE` is used. 

Once these three checks are satisfied, the dirty flag is cleared via `clear_page_dirty_for_io`, and then finally, we call writepage function that was passed in earlier (`__writepage`). As I said earlier, `__writepage` serves as a wrapper for the FS-specific `writepage` which in the case of ext4 is `ext4_writepage`.

I'm not going to go over `ext4_writepage`, but essentially what it does is that it passes the page down the block device driver, which then pushes the page to the disk via DMA. This completes the journey of the average `fsync`.


## msync
Luckily, the `msync` implementation is not too different from the `fsync`. 

Much like with `fsync`, we at the entry for `msync` which can be found [here](https://elixir.bootlin.com/linux/v5.14/source/mm/msync.c#L32). 

`msync` starts out by performing some checks. There are a variety of checks here. We first check the flags and if any flags that are not `MS_ASYNC`, `MS_SYNC`, or `MS_INVALIDATE` are set then return immediately. Second we check to see if both `MS_ASYNC` and `MS_SYNC` are set. If they are then return. Finally, we check if the inputted address and offset are valid. 

If these basic checks pass then we can start `msync`. `msync` starts out with `find_vma` which finds the `vm_area_struct` that contains the `start`. Now `find_vma` does the search in a particularly interesting (And quite perplexing) way. Within each `vm_area_struct` there are fields called `vm_start` and `vm_end`. `vm_start` represents the beginning virtual address of the vma and `vm_end` represents the ending virtual address of the vma. Now instead of checking that `vm_start` <= `start` < `vm_end`, it instead checks that `start < vm_end` and returns the first vma that satisfies that condition. So, we need to do some additional checking within `msync`. Specifically, we make sure that `start >= vm_start`. 

If all these checks are satisfied, then we can actually begin. We start by calculating the portion of the file that we want to sync which is then stored in `fstart` and `fend`. With these in hand we perform yet another which checks if 1. we have a file, 2. we are syncing synchronously, and 3. the mapping is shared. If these three conditions are satisfied then we pass `fstart` and `fend` into `vfs_fsync_range` and we continue with the same `fsync` routine. Note that in the case of `fsync` we passed in 0 and `LLONG_MAX`, but this time we are passing in only `fstart` and `fend` as we are syncing only the portion of the file specified by the user.
 

## Additional comments

As you might have realized, Linux employs a very modular system when it comes to implementation. As we talked about `fsync` we were interleaving between different "layers" of the kernel. Initially, we start out at the virtual file system (vfs) level which called the file-system-specific function which then called the block-device-specific functions.

 This layering idea is something that is common across the entire linux kernel whether it be networking, file-systems, or anything else and is something that one should get used to.

## What's Next

From here, we are going to discuss how `PAGECACHE_TAG_DIRTY` is set by discussing `mmap`, `write`, and page faulting. 


## References
ext4's `address_space_operations` - <https://elixir.bootlin.com/linux/v5.14/source/fs/ext4/inode.c#L3648>

`find_vma` - <https://elixir.bootlin.com/linux/v5.14/source/mm/mmap.c#L2301>    






