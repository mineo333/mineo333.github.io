---
title: A Look At ELFs
date: 2022-03-05 23:40:00
categories: [Random]
tags: [Random]
---

In this article, I want to discuss ELFs, how they work, what they do, and how they're structured. 

## Introduction

The purpose of this article is to discuss the basics of Executable Linked Format (ELF). This includes things such as the ELF header, ELF program headers, and ELF sections. 

 This article will use two main resources: GEF (A PWN extension of GDB), the System V documentation which can be found [here](https://docs.oracle.com/cd/E19120-01/open.solaris/819-0690/6n33n7fcb/index.html), Cutter (Radare2 GUI), the `readelf` utility, and `include/uapi/linux/elf.h`. 

To understand this article, a strong understanding of C and its datatypes is necessary. 
An understanding of x86-64 assembly will also be helpful. 

This is an incredibly dense article, so I recommend reading it multiple times as well as trying out the exercises that I perform yourself. 

## What the fuck is an ELF?

An ELF also known as the **Executable Linked Format** is the standard format for Unix executables. The ELF format is important because it fundamentally defines how programs are stored and interpreted on Unix systems. So, by having a thorough understanding of the ELF format you will fundamentally understand how programs are stored on the disk as well as how they are loaded into memory at runtime. 

## Some History

The ELF specification is a small portion of the System V ABI. The System V ABI is the specification for the original Unix operating system and almost every Unix and Unix-like operating system adheres to it. 

The ELF was actually not the first executable format for Unix. The first executable format was actually aout. However, aout was later phased out because it could not handle additional sections beyond .text, .data, and .bss

## Caveat

This article **DOES NOT** deal with PIE executables (ELF type = `ET_DYN`). That will be in a different article as many fields, especially in the program header, take on vastly different meanings when dealing with PIEs. 

## The Loading Process

Before we even discuss what an ELF is, it is important to understand how a program is loaded into memory at a high level. 

In order to run a program, we usually use the `fork` -> `execve` flow where a process will fork itself to create a new process and call the `execve` system call to replace the process image. The step we are interested in is the `execve` step. 

When `execve` is called, there are two steps performed. The first step is obviously mmaping the executable into memory. The second step, however, is slightly more mysterious.

 ELFs are what we call dynamically linked meaning that certain symbols are resovled at runtime instead of at compile-time. For example, definitions for functions like `read`, `write`, and `printf` are not actually present your binary. Instead, if you look in your binary, you will find that definitions will point to thunk functions like the one seen below:
 
```
int printf (const char *format);
endbr64
bnd jmp qword [printf] ; 0x3f60
```

```
;-- printf:
0x00003f60      .qword 0x0000000000004060
```
 
The actual definitions of the functions are located in a shared library (Most often libc) which is mmap'd at runtime. However, what mmaps these shared libraries? The answer is a program called an *interpreter* (Most often `/lib64/ld-linux-x86-64.so.2`). Interpreters are supplementary programs that are `execve` mmaps alongside the actual binary to perform dynamic linking. 

In fact, the `execve` syscall almost always passes control to the interpreter instead of the binary itself. This is so that the interpreter has the opportunity to map the shared libraries into memory before the actual program runs. 

This behavior can actually be seen by running strace on a dynamically linked program as seen below.

![shoff](/assets/img/ELF/strace.png)

Notice how `openat` opens libc and gives an fd of 3. This fd is then used in the three highlighted mmap calls. Also, note that the mmap calls are done with `MAP_DENYWRITE` which essentially prevents someone from running `mprotect` on the mapping and clobbering libc. 

With all of this in mind, we can begin discussing what is actually in an ELF. 


## ELF Data Types

Before we get into the meat of this article, we also need to discuss the various ELF datatypes. ELFs use many confusing typedefs.

The following are the relevant typedefs:

```c
typedef __u64	Elf64_Addr;
typedef __u16	Elf64_Half;
typedef __s16	Elf64_SHalf;
typedef __u64	Elf64_Off;
typedef __s32	Elf64_Sword;
typedef __u32	Elf64_Word;
typedef __u64	Elf64_Xword;
typedef __s64	Elf64_Sxword;
```
`__sx` is a signed type where `x` is the number of bits.

`__ux` is an unsigned type where `x` is the number of bits.



## The ELF Header

The ELF header is at the core of what an ELF is as the first thing an interpreter will read is the ELF header.

The purpose of an ELF header is to tell the interpreter where key headers are in the file (i.e. program and section headers). 

So, what does the ELF header look like? Well it looks like this:

```c
#define EI_NIDENT	16

typedef struct elf64_hdr {
  unsigned char	e_ident[EI_NDENT];	/* ELF "magic number" */
  Elf64_Half e_type;
  Elf64_Half e_machine;
  Elf64_Word e_version;
  Elf64_Addr e_entry;		/* Entry point virtual address */
  Elf64_Off e_phoff;		/* Program header table file offset */
  Elf64_Off e_shoff;		/* Section header table file offset */
  Elf64_Word e_flags;
  Elf64_Half e_ehsize;
  Elf64_Half e_phentsize;
  Elf64_Half e_phnum;
  Elf64_Half e_shentsize;
  Elf64_Half e_shnum;
  Elf64_Half e_shstrndx;
} Elf64_Ehdr;
```

Source: `include/uapi/linux/elf.h` Line 222

What the fuck does this even mean? To break this down, we can reference the trusty ELF reference specs. 

- The first field is `e_ident` or the ELF identification array. This array contains various useful pieces of information including the magic bytes, endianess, and more. The exact array can be found in Table 7-3 in the System V documentation. 

- The second field is `e_type` or the type of file. The number in this field determines what kind of executable this is (i.e. object file, shared object file, executable, PIE executable).

- The third field is `e_machine` or the machine type. This simply tells you the architecture that this executable was complied for.

- The fourth field is `e_version` or the version of the file. This is typically `0x1` as any value <0x1 is invalid.

- The fifth field is `e_entry` or the entry point. This is the address of the first address in the file. After the program has been loaded into memory, the kernel will transfer execution to the program by setting the program counter to this address (Or the equivalent virtual address).

- The sixth field is `e_phoff` or the offset of the program headers. This tells the ELF interpreter where the program headers are located in the file. We will talk about program headers later into this article.  

- The seventh field is `e_shoff` or the offset of the section headers. This tells the ELF interpreter where the section headers are located in the file. 

- The eighth field is `e_flags` or the flags field. This contains various flags that are used by the loader and the kernel. 

- The ninth field is `e_ehsize` or the size of the ELF header. This is literally the size of the ELF header in bytes. 

- The tenth and eleventh fields are `e_phentsize` and `e_phnum` or the size of each program header in bytes and the number of program headers respectively. It follows that `e_phentsize`*`e_phnum` = total size of all program headers.

- The twelfth and thirteenth fields are `e_shentsize` and `e_shnum` or the size of each section header bytes and the number of section headers respectively. It follows that `e_shentsize`*`e_shnum` = the total size of all section headers. 

- The fourteenth and final field is `e_shstrndx` or the index of the section header string table in the section headers. This essentially allows the loader to find the names of each section. We will see this field be used when we talk about sections in ELFs.

All of this information can be seen in a nice way via the `readelf` utility on Linux. By running `readelf -h [executable name]`, it will show a nice view of the ELF header and what all the fields mean. For example:

EXAMPLE OF `readelf -h` HERE

Using the ELF header, an interpreter is able to find all the information it needs about a binary. One major piece of information is found in the section headers. 


## Sections and Section Headers

ELFs contain all kinds of data whether that be code, data, symbols, strings, or anything else. However, how does a reader of the ELF know what part is what? This is where sections become important. Sections describe exactly that - they tell readers where what is. 

For example, suppose we are trying to find where our executable code is located (`.text`). Without some kind metadata, finding that specific data is impossible. Via sections, we solve that problem and provide readers with the proper metadata to find the information they need. 

So, what the hell does this metadata look like? This can be seen below:

```c
typedef struct {
        Elf64_Word      sh_name;
        Elf64_Word      sh_type;
        Elf64_Xword     sh_flags;
        Elf64_Addr      sh_addr;
        Elf64_Off       sh_offset;
        Elf64_Xword     sh_size;
        Elf64_Word      sh_link;
        Elf64_Word      sh_info;
        Elf64_Xword     sh_addralign;
        Elf64_Xword     sh_entsize;
} Elf64_Shdr;
```

Let's discuss the fields.

- The first field is `sh_name` or the section name. This is actually an offset in bytes into the section header string table (Described by `e_shstrndx` in the ELF header). The string at the `sh_name` offset in the section header string table is the name of the section defined by this header. 

- The second field is `sh_type` or the section type. The `sh_type` field essentially explains what the purpose of this section is. We will discuss the various section types later. 

- The third field is `sh_flags` or symbol flags. These flags are typically used to describe certain attributes about a section. There are a few important flags. One major flag that is used very often is `SHF_ALLOC` which indicates that this particular section will be loaded into memory. Another flag is `SHF_WRITE` which indicates that a section will be writable when loaded into memory. A third common flag is `SHF_EXECINSTR` which indicates that the section has executable instructions. 

- The fourth field is `sh_addr`. This field contains the address at which this section will be loaded **in memory**. So, if you run something like `p {char[sh_size]} [sh_addr]` in gdb, you should get the contents of this section. 

- The fifth field is `sh_offset`. This field contains the offset of this section **in the file**. So, if you run `xxd -s [sh_offset] [exec_name]` you should see the contents of this section. 

- The sixth and seventh fields are `sh_link` and `sh_info` respectively. Both of these fields contain information whose meaning depends on the `sh_type`. 

- The eighth field is `sh_addralign` which contains the alignment of `sh_addr`. This is to say that `sh_addr % sh_addralign == 0`. 

- The ninth and final field is `sh_entsize`. Some sections, such as symbol tables, hold entries of constant size. This field holds that constant size. 

One important thing to discuss is how sections headers are stored in ELF files. Section headers are essentially stored in an array. The start of this array is located at an offset of `e_shoff` with a size of `e_shnum`. This information will become important later.

All this information is great. However, what can sections actually be used for? In order to understand this question we must analyze `sh_type` more carefully. 

`sh_type` can take on many values. However, by far the most common is `SHT_PROGBITS` or program bits section. This simply represents a section that contains executable-specific data. Many of the common sections that we know and love fall under this type such as `.text`, `.data`, `.bss`, `.rodata`, and more.

Another very common type of section is `SHT_STRTAB` or a string table. There are many use cases for string tables however by far the most common is `.shstrtab`.

`.shstrtab` is the name of section header string table. This section is a string table which contains the names of all the sections. For example, the names of sections such as `.text`, `.data`, and `.bss` are stored here. This section is indexed by the `sh_name` field. As we discussed when talking about the header, this field may be found via the `e_shstrndx` field in the header which gives the index of 

Another extremely common string table is `.dynstr`. `.dynstr` describes the names of symbols that need to be dynamically linked. This includes names like `printf`, `scanf`, and many more common functions. We will talk more about `.dynstr` when we get into dynamic linking. 

Another common section type is `.symtab` (`sh_type = SHT_SYMTAB`) or a symbol table. We will talk about symbol tables later, but in essence, these symbol tables contain information on symbols such as functions and variables within the file. Having a string table is very important for both compile-time linking as well as dynamic linking. When we talk about dynamic linking we will also talk about a stripped down version of the symbol table called `.dynsym`.

Another useful question to ask is how are sections stored. As we discussed while discussing the ELF header, the `e_shoff` field describes the offset of the section headers into the file. However, what is actually located at this offset?

At `e_shoff` bytes into the file, there is an array of `Elf64_Shdr`. The length of this array is determined by `e_shnum`.

## Using what we know so far...

To review, I want to do a brief demonstration showing how the information in the ELF header as well as the section headers may be used to navigate the file using only `xxd`. In this demonstration, we will attempt to find `.text`. This is a pretty good simulation of how a tool like `readelf` looks through an ELF file. 

In order to perform this demonstration, I will be using a simple C program that I made which can be found [here](https://github.com/mineo333/ELF-testing). We will specifically be using the `no-pie` version for simplicity. 

In order to locate the text section, we first need to find the section headers as we can only find `.text` if we know how to find the sections. As you may recall, this information is located in the ELF header with the `e_shoff` field. In order to locate the ELF header, we need only run `xxd -l 64 no-pie` as the ELF header is the first 64 bytes of the file. The output can be seen below:

![ELF Header](/assets/img/ELF/elf_hdr_no_pie.png)

So, this is great, but where in these bytes is `e_shoff`. In order to find that information, we may use a tool like pahole to determine the offset of `e_shoff`. 

```c
struct elf64_hdr {
	unsigned char              e_ident[16];          /*     0    16 */
	Elf64_Half                 e_type;               /*    16     2 */
	Elf64_Half                 e_machine;            /*    18     2 */
	Elf64_Word                 e_version;            /*    20     4 */
	Elf64_Addr                 e_entry;              /*    24     8 */
	Elf64_Off                  e_phoff;              /*    32     8 */
	Elf64_Off                  e_shoff;              /*    40     8 */
	Elf64_Word                 e_flags;              /*    48     4 */
	Elf64_Half                 e_ehsize;             /*    52     2 */
	Elf64_Half                 e_phentsize;          /*    54     2 */
	Elf64_Half                 e_phnum;              /*    56     2 */
	Elf64_Half                 e_shentsize;          /*    58     2 */
	Elf64_Half                 e_shnum;              /*    60     2 */
	Elf64_Half                 e_shstrndx;           /*    62     2 */

	/* size: 64, cachelines: 1, members: 14 */
};
```
From the pahole output, we can see that the offset of `e_shoff` is 40 bytes into the header with a size of 8. Thus, we may find `e_shoff` by simply doing running `xxd -l 8 -s 40 no-pie`. The output for this can be seen below:

![Offset of Section Headers](/assets/img/ELF/e_shoff_no_pie.png)

Because the file is in little-endian, we can determine that the section headers are `0x36c0` bytes into the file. 

However, this is not sufficient information to navigate an ELF as we also need the number of section headers as well as the size of each section header. Luck for us, this information can be found with the `e_shnum` and `e_shentsize` fields respectively. 

Much like with `e_shoff`, we can use the pahole output to determine that the offsets of `e_shnum` and `e_shentsize` are 60 and 58 with a size of 2 each. With this information, we may get the actual values by running `xxd -s 58 -l 4 no-pie`. This nets an output where the first two bytes are `e_shentsize` and the last 2 bytes are `e_shnum`. The output for this can be seen below:

![Number of Section Headers and Size](/assets/img/ELF/sh_num_size_no_pie.png)

Because the file is in little endian, we can determine that `e_shentsize = 0x40` and `e_shnum = 0x1f`. 

With this, we now have all the information we need to traverse this file. Specifically, we know that the section headers are located `0x36c0` bytes into the file and size of the section headers is `e_shentsize * e_shnum = 0x40 * 0x1f = 0x7c0`. With these two pieces of information we can dump the section headers using the command `xxd -s 0x36c0 -l 0x7c0 -c 0x40 no-pie`. The (partial) output of this can be seen below:

![Section Headers](/assets/img/ELF/section_headers_no_pie.png)

So, this is great, but how do we know which one is `.text`? The answer lies in the `sh_name` field of the section header. The `sh_name` field tells us the exactly where the name of any given section is in the section header string table. So, if we find the `sh_name` associated with `.text`, we can identify which section header is actually `.text`! 

However, how do we find the section header string table? In order to find the section header string table, we need to revisit the header. The header has a crucial piece of information that describes the section header string table: `e_shstrndx`. The `e_shstrndx` tells us index of the section header string table in the section headers. 

Going back and looking at the header, we see that `e_shstrndx` is located at an offset of 62 with a size of 2. So, Running `xxd -s 62 -l 2 no-pie` we get the following:

![shstrndx](/assets/img/ELF/e_strshndx_no_pie.png)

In little endian, we see that `e_shstrndx = 0x1e = 30`. Thus, we now know that the 31st section header is that of the section header string table. Doing some simple math, we realize that the 31st section header is `30*0x40 = 0x780` bytes into the section header array. We learned earlier that `e_shoff = 0x36c0`. So, the the section header string table's section header is located at `0x36c0 + 0x780 = 0x3e40`. We can look at that using the following command: `xxd -s 0x3e40 -l 0x40 no-pie`. This gives the following:

![section header string table's section header](/assets/img/ELF/str_section_hdr_no_pie.png)

Now, we have the section header string table's section header. However, once again we are faced with the problem of how do we use these bytes.

In order to find where the actual section header string table is, we need two pieces of information from section header: the offset of the section into the file and the length of the section. This will tell us exactly where section header string table is. To find what the offset and length are, we may use the pahole command much like we did with the ELF header. Below is the pahole output for `elf64_shdr`:

```c
struct elf64_shdr {
	Elf64_Word                 sh_name;              /*     0     4 */
	Elf64_Word                 sh_type;              /*     4     4 */
	Elf64_Xword                sh_flags;             /*     8     8 */
	Elf64_Addr                 sh_addr;              /*    16     8 */
	Elf64_Off                  sh_offset;            /*    24     8 */
	Elf64_Xword                sh_size;              /*    32     8 */
	Elf64_Word                 sh_link;              /*    40     4 */
	Elf64_Word                 sh_info;              /*    44     4 */
	Elf64_Xword                sh_addralign;         /*    48     8 */
	Elf64_Xword                sh_entsize;           /*    56     8 */

	/* size: 64, cachelines: 1, members: 10 */
};

```

As can be seen from the pahole output, `sh_offset` is at an offset of 24 with a size of 8 and `sh_size` is at an offset of 32 with a size of 8.

We can get `sh_offset` by running `xxd -s 0x3e58 -l 0x8 no-pie` and we can get `sh_size` by running `xxd -s 0x3e60 -l 0x8 no-pie`. Running these commands, we get the following:

![offset and size](/assets/img/ELF/sh_strtab_off_size_no_pie.png)

In little endian, we see that `sh_offset = 0x35a0` and `sh_size = 0x011f`. Now, we know exactly where the section header string table. Specifically, it is at an offset of `0x35a0` bytes with a length of `0x11f` bytes. We may use `xxd -s 0x35a0 -l 0x11f no-pie` to print it out:

![section header string table](/assets/img/ELF/shstrtab_no_pie.png)

Now, we are able to find the `sh_name` associated with `.text`. As you may recall `sh_name` is an offset in bytes into the section header string table. So, we just need to find the offset of `.text`. Doing some counting, you will eventually figure out that `.text` is at an offset of `0xb0 = 352`. 

With this, we now know that the section header associated with `.text` has an `sh_name` of `0xb0`. So, we just need to look through the `0x1f` section headers we found earlier and find the one with an `sh_name` of `0xb0`. 

Looking through the section headers, we realize that the 15th section header is the one we are looking for. We may print this out using `xxd -s 0x3a80 -l 0x40 -c 0x40 no-pie`. This results in the following:

![section header string table](/assets/img/ELF/text_shhdr_no_pie.png)




## Program Headers

Sections allow us to find what we need in the ELF. However, it doesn't tell us certain information. For example, how do we let the interpreter know that we shouldn't load `.shstrtab` or `.debug` into memory. How do we let the interpreter know that `.text` should be executable? The section header simply does not provide this kind of metadata.

Enter: program headers. Program headers provide this exact kind of metadata. Program headers define what are known as "segments". These essentially segments are essentially "blocks" of the file that will be loaded together by the kernel.  

 The format for a program header can be seen below:

```c
typedef struct elf64_phdr {
  Elf64_Word p_type;
  Elf64_Word p_flags;
  Elf64_Off p_offset;		/* Segment file offset */
  Elf64_Addr p_vaddr;		/* Segment virtual address */
  Elf64_Addr p_paddr;		/* Segment physical address */
  Elf64_Xword p_filesz;		/* Segment size in file */
  Elf64_Xword p_memsz;		/* Segment size in memory */
  Elf64_Xword p_align;		/* Segment alignment, file & memory */
} Elf64_Phdr;
```

Let's discuss the fields of program headers:

- The first field is `p_type` or the type of the program header. The type of the program header determines how it is handled by the interpreter. We will discuss the various program headers in depth later.

- The second field is `p_flags` or the flags of the program headers. 

- The third field is `p_offset` or the offset of the segment in the file

- The fourth field is `p_vaddr` or the virtual address that this segment will be loaded at. The address is always aligned at a `p_align`-boundary.

- The fifth field is `p_paddr` or the physical address that this segment will be loaded at. Due to virtual address and paging in modern operating systems, the kernel cannot and does not provide any guarantees about the PFN where this segment will be loaded (Nor does it really matter). As such this field is really only used when physical addressing actually matters (i.e. in certain embedded systems). In the (very likely) case that it doesn't matter, `p_paddr = p_vaddr`. 

- The sixth field is `p_filesz` or the size of this segment in the file. 

- The seventh field is `p_memsz` or the size of this segment in memory. Usually, `p_filesz = p_memsz`. Like `p_paddr` this is also usually wrong 

- The eighth and final field is `p_align` or the alignment of the load address. For loadable segments this is typically page size (Usually `0x1000`).

## Using Program Headers


## The interpreter

As we discussed at the beginning of the articles, one of the key aspects of an ELF is the need for an interpreter. To understand the interpreter, we need to hit kernelspace as well as glibc. 

In kernelspace, I want to take a good look at the ELF loader and how it interacts with the interpreter. The ELF loader is located in `fs/binfmt_elf.c`. The entrypoint into the ELF loader is `load_elf_binary`. This can be found [here](https://elixir.bootlin.com/linux/v5.14/source/fs/binfmt_elf.c#L823). 

In order to understand `load_elf_binary`, it is important to discuss how we get to this function. `load_elf_binary` is run as a result of running `execve`. When `execve`, 







## References

`include/uapi/linux/elf.h` - <https://elixir.bootlin.com/linux/v5.14/source/include/uapi/linux/elf.h>








