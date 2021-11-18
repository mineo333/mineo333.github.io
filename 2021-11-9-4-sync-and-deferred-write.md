---
title: 4 - sync and Deferred Write
date: 2021-11-8 15:40:00
categories: [QogChamp]
tags: [Linux Kernel Exploitation, Syscall]
---

In this article, I want to describe the `sync` syscall. By describing the `sync`, I hope to describe how deferred write actually works in Linux. This will give insight into how Qogchamp works.

## How to read this article
This article goes step by step through the `msync` and `fsync` syscalls. I HIGHLY recommend that you follow along on your own as there is a minimal amount of code. In order to follow along either start at the entrypoints mentioned, or, if you're skipping around, use [this](https://elixir.bootlin.com/linux/v5.14/source) wonderful website to search look up the symbols.

## Entrypoint
In order to begin analyzing `sync`, it is important to find the entrypoint. Luckily, we can use [this](https://filippo.io/linux-syscall-table/) wonderful site which describes all the entrypoints for each syscall as well as their registers. 

Looking at this site, we find 3 kinds of `sync` - `msync`, `fsync`, and plain `sync`. For ease, we will analyze both `msync` and `fsync`. The reason we won't analyze `sync` is because it wakes up the flusher threads which makes it somewhat difficult to analyze.

## msync vs fsync

I briefly want to go over the difference between `msync` and `fsync` at a high level before we get into the specifics. 

For those unaware, there are two ways to open and modify files: `write` and `mmap`. 

In the case of `write`, we simply open the file using `open` and then we use `write` along with the returned file descriptor. In the case of `write`, as we will see, the changes are written directly to the backing `struct address_space` of the inode. In order to write the contents of the written file back to the disk, we use `fsync` which takes in the file descriptor recieved from `open`.

In the case of `mmap` we map the file into the address space of the process. Meaning, we can access and modify portions of the file in the same way we might access portions of stack or heap. These modifications are then written back to the page cache. `msync` is used to write back these changes to the disk. 

## fsync

Using the site mentioned above, we can find the entry point of `fsync` which happens to be in fs/sync.c which can be found [here](https://elixir.bootlin.com/linux/v5.14/source/fs/sync.c#L230). In order to find the actual syscall within the file we can look for the `SYSCALL_DEFINEx` macro where x is the number of arguments in the syscall.  

Quickly, we find that we enter a function called `do_fsync` with the `fd` and another argument `datasync` being 0. `datasync` is only set to 1 if `fdatasync` is called. 

Within `do_fsync` we call `fdget` which returns a `struct fd`. For those familiar with Linux file operations might recognize this function. Essentially, within the process control block of Linux - `struct task_struct` there is an array of `struct fd`s which contains information on opened files. Using this integer as an index into this array, we get the relevant `struct fd`. If you're particularly interested in these functions I recommend reading [this](https://www.kernel.org/doc/Documentation/filesystems/files.txt).

Using this `struct fd`, assuming the backing file exists, we enter another function known as `vfs_fsync`. Anyone familiar with Linux file operations might be familiar with the `vfs_` prefix. This is derived the idea of the virtual file system. In the Linux Kernel, we have a generic high-level file-system which functions as the abstraction above concrete file systems (i.e. ext4). As we'll see, these `vfs_` functions typically eventually call filesystems-specific functions. 

In `vfs_fsync` we call `vfs_fsync_range` which is used to sync portions of a file. We use 0 and LLONG_MAX as the start and end respectively to sync the entire file. From here, we already move into filesystem-specific operations. 

In `vfs_fsync_range`, we begin with two if statements. The first if-statement checks if the file system has an fsync member function. If not, then it errors. This happens because the virtual file system needs a concrete `fsync` function provided by the filesystem in order to perform `fsync`. The second if statement is a little more peculiar. The second if statement is where `datasync` becomes useful. If `datasync` is 0 (Which it is), then we essentially update the inode metadata and write that new metadata to the disk. When that is complete, we finally call the filesystem specific `fsync`.

Because of how many filesystems there are, I am only going to focus on ext4's implementation which can be found [here](https://elixir.bootlin.com/linux/v5.14/source/fs/ext4/file.c#L912). Looking at the `fsync` field, we find that it points to `ext4_sync_file`. From here, we can start the filesystem specific portion of fsync. 

## FS-Specific fsync

Here we start with `ext4_sync_file`. I should note that this is not the ONLY form of `fsync` as every filesystem (And block device) should have its own implementation. However, because ext4 is the most common, I'll be doing this. Note that because ext4 is a journaling file system, a lot of the code in the ext4 implementation is journal management, and because journaling is irrelavent for this article, I will be skipping over that code. 

In `ext4_sync_file`, we start out by checking if the superblock of the filesystem is read only. If it is then we leave (`goto out`). This is fairly self-explanatory as we can't write to a read-only filesystem. 

After that check is completed, we immediately jump into `file_write_and_wait_range` which actually happens to be a VFS function. In `file_write_and_wait_range` we come across a familiar data structure: `address_space` which as you might recall is the page cache data structure. As might be obvious, what we are going to write the pages from `address_space`. 

The first thing that we check in `file_write_and_wait_range` is if the page-cache has any pages at all. This is done by running the function `mapping_needs_writeback`. This is an important check because if the page cache has no pages, then what's the point of starting writeback. 

If the page cache for the inode does need writeback then we run `__filemap_fdatawrite_range`. This is run with 4 parameters. The first 3 are self-explantory. The last one is interesting however: `WB_SYNC_ALL`. What this flag does is that if a page in the page cache is already under writeback, then it should wait for the writeback to finish and then redo the writeback. The alternative to this is `WB_SYNC_NONE` which does not wait. This distinction is best described in the documentation for `__filemap_fdatawrite_range` which says that "The difference between \[ `WB_SYNC_NONE` and `WB_SYNC_ALL` \] is that if a dirty page/buffer is encountered, it must be waited upon, and not just skipped over"

Looking into `__filemap_fdatawrite_range` we see that we setup something called a `writeback_control`. This struct essentially decides how writeback will be done. There are 4 fields, but one that is new: `nr_to_write`. This represents the number of pages to writeback. This is set to `LONG_MAX` indicating that we should writeback all pages (Which makes sense). Once that is setup, we jump into `filemap_fdatawrite_wbc`. 

`filemap_fdatawrite_wbc` according to documentation is the function that calls "writepages on the mapping using the provided wbc to control the writeout". This is where the fun really begins. In this function, the first thing we do is check if the page cache even has the capability to writeback and, more importantly, if the page cache is dirty. If the page cache is not dirty then the function returns immediately. This is **extremely** important for later. If these two checks pass, then we call `do_writepages` which actually does the writeback. 

Looking into `do_writepages`, we start with checking check if the number of pages to write is greater than 0. If not, then there is nothing to write then return. From here, we move into an infinite loop. Within this infinite loop, the exit code is particularly interesting. Looking at the conditional, we see that the code will only exit if `(ret != -ENOMEM) || (wbc->sync_mode != WB_SYNC_ALL)` or, in other words, the `writepages` did not fail OR we don't have to sync everything. 

Within `do_writepages`, we arrive a fork in the road. If the filesystem has an implementation for `writepages` in its `address_space_operations` then use that. If not, then use a `generic_writepages`. Looking into ext4's `address_space_operations`, we see that ext4 has an implementation for `writepages` in `ext4_writepages`. So, we jump to that function. 

Looking into `ext4_writepages`, we see a massive block of code. However, upon closer analysis it becomes obvious that this is for dealing with special cases of writeback - specifically in three cases: cyclic writeback, inline file writeback, and non-journaled writeback. The reason that this code happens here is because, generally, the standard execution path for ext4 incorporates journaling.  However, because ext4 is generally journaled for all normal files, it can safely be assumed that we can take the standard execution path. 

Assuming we are performing the standard writeback procedure, we start at line 2671. Here, we first check if the file it being journaled. If it is (Which we are assuming), then we jump to `generic_writepages`. 

Within `generic_writepages`, we start by looking checking if the filesystem's`address_space_operations` has a `writepage` implementation. If it doesn't then leave. This `writepage` function is what actually writes the new file buffer to the disk. So, without a `writepage` this endeavor is basically impossible. We then run a function called `blk_start_plug`. According to the documentation, all this does is warm up the block device and block io layer and warns them that they will be receiving IO requests shortly. From here, we jump to `write_cache_pages`. However, before we go into `write_cache_pages` we need to analyze one of the parameters - specifically `__writepage`. 

`write_cache_pages` requires a parameter that is of type `writepage_t`. This is an alias for a function point to a `writepage` function. This will serve as the means by which `write_cache_pages` actually executes the write. In the case of this particular call, `__writepage` serves as the generic wrapper for all fs-specific `writepage` implementations. Looking into `__writepage`, we quickly find that all it basically does is call the fs-specific `writepage` implementation from `address_space_operations`. I just thought I'd mention that in case any of you were wondering.

Anyway, looking into `write_cache_pages` we find once again a very large block of code. However, once again, much of this code is dealing with edge cases which we don't particularly care about. The first real line of consequence (And perhaps the most important line in the entire `fsync` implementation) happens on line 2197. This is the line where the magic of QogChamp really comes together. Assuming that the sync mode is `WB_SYNC_ALL` (Which it is), then we will jump into `tag_pages_for_writeback`. 

In `tag_pages_for_writeback` the kernel essentially searches through the page cache and tags all the dirty pages with a special tag that says that it needs to be written back. How does it do this? The answer hearkens back to the `i_pages` xarray structure within the `address_space` object. What it does is that it uses a special macro to find all entires that are marked with `PAGECACHE_TAG_DIRTY`. Every entry marked with this tag is then marked with `PAGECACHE_TAG_TOWRITE` indicating that it should be written to the disk. There are two questions posed from this. First, why do this? The main reason for doing this is written in the documentation for `tag_pages_for_writeback`. We do this additional marking in order to prevent livelocking which could be caused by a process writing to the pages as we're trying to do the writeback. Second, where does the `PAGECACHE_TAG_DIRTY` tag come from and when is it set? This is the answer to this question holds the key to why QogChamp works. This topic will be discussed further as we discuss the `mmap` and `write`implementations. 

Once `tag_pages_for_writeback` returns, it can be assumed that all the pages ready for writeback are tagged and ready to go. From here, its just a matter of iteratng through the pages and sending them on their way to the disk. In order to writeback, the code does three checks. First it checks if the page has been truncated under us. If so, then leave because a truncated page cannot be written back. Second it checks if the page is no longer dirty. If the page is no longer dirty then the page has been written back by someone else. Finally, it checks if the page is already currently under writeback. If it is, then it wait for it to not be under writeback and then either continues or re-does the write depending on whether `WB_SYNC_ALL` or `WB_SYNC_ALL` is used. 

If these three checks are satisfied, the dirty flag is cleared via `clear_page_dirty_for_io` and then finally we call writepage function that was passed in earlier (`__writepage`). This calls the fs-specific `writepage` which in this case is `ext4_writepage`.

I'm not going to go over `ext4_writepage`, but essentially what it does is that it passes the page down the block device driver which then pushes the page to the disk. This completes the journey of the average `fsync`.


## msync
Luckily, the `msync` implementation is not too different from the `fsync`, and eventually we jump to the  

## Additional comments



## References
ext4's `address_space_operations` - <https://elixir.bootlin.com/linux/v5.14/source/fs/ext4/inode.c#L3648>






