# 4190.307 Operating Systems (Fall 2025)
# Project #5: xFFS: Faster File System for xv6
### Due: 11:59 PM, December 21 (Sunday)

## Introduction

Inspired by the classic BSD Fast File System (FFS), xFFS extends the original `xv6` file system with the concept of ___block groups___ (a.k.a. _cylinder groups_). This architectural change co-locates related data and metadata on the disk to reduce disk seek latency and boost file system throughput.
The goal of this project is to implement block groups and a locality-aware allocation policy in `xv6`, while preserving the original on-disk layout as much as possible.

## Background

### The On-disk Layout of `xv6` File System

The following illustrates the on-disk layout of the current `xv6` file system.

```
+-+-+---+-----+-+--------------------------------------------------------+
|B|S| L |  I  |M|                         D                              |
+-+-+---+-----+-+--------------------------------------------------------+
```
* `B`: Boot block (1 block) -- Not used
* `S`: Superblock (1 block) 
* `L`: Log blocks (size derived from `LOGBLOCKS` @ `kernel/param.h`)
* `I`: Inode blocks (size derived from `NINODES` @ `mkfs/mkfs.c`)
* `M`: Data bitmap blocks (size derived from `FSSIZE` @ `kernel/param.h`)
* `D`: Data blocks 

## xFFS
 
### The On-disk Layout 

At the front of the disk, xFFS keeps the same header region as the original `xv6`: a boot block, a superblock, and a sequence of log blocks. xFFS introduces one additional header block, the Global Descriptor Table (GDT) block, which records summary information of each block group. 
Beyond this header region, the disk is divided into fixed-size __block groups__, each containing its own bitmap, inode blocks, and data area. This layout aims to keep related metadata and data close together within a block group, improving locality and reducing seek distance.

```
+-+-+---+-+-+---+---------+-+---+---------+---------------+-+---+---------+
|B|S| L |G|M| I |    D    |M| I |    D    |       ...     |M| I |    D    |
+-+-+---+-+-+---+---------+-+---+---------+---------------+-+---+---------+
          |----- bg 0 ----|----- bg 1 ----|               |----- bg 9 ----|
```

For this project, the disk size is fixed at 20,520 blocks (1 KiB per block): the first 40 blocks form the header region, and the remaining 20,480 blocks are divided into 10 block groups of 2,048 blocks each. The detailed layout is as follows.

* `B`: Boot block (1 block) -- Not used
* `S`: Superblock (1 block)
* `L`: Log blocks (37 blocks) (= `LOGBLOCKS` @ `kernel/param.h` + log header block)
* `G`: GDT block (1 block) (= `GDTBLOCKS` @ `kernel/param.h`)

Each block group is 2,048 blocks and contains:
* `M`: Data bitmap block (1 block) (= `BITMAP_PER_BG` @ `kernel/param.h`)
* `I`: Inode blocks (8 blocks) (= `INODE_PER_BG` @ `kernel/param.h`)
* `D`: Data blocks (2,039 blocks) (= `(BLOCKS_PER_BG`-`BITMAP_PER_BG`-`INODE_PER_BG`) @ `kernel/param.h`)

### Superblock 

The superblock is the file system's on-disk "header": it defines the global geometry and parameters the kernel needs to interpret the disk. At boot, the kernel reads the superblock to locate and size every region (log, GDT, inodes, bitmaps, and data). In xFFS, it also encodes the block group configuration, enabling the allocator to make locality-aware allocation decisions.

The `struct superblock` is defined in `kernel/fs.h` as follows. You may add additional fields you find useful, as long as the total superblock size fits within one block (1,024 bytes). 

```C
// In kernel/fs.h
struct superblock {
   uint magic;               // must be FSMAGIC of xFFS: 0x000F0F05
   uint size;                // size of file system image (in blocks)
   uint nblocks;             // number of data blocks
   uint ninodes;             // number of inodes
   uint nlog;                // number of log blocks
   uint logstart;            // block number of first log block
#ifdef SNU
   uint nbg;                 // number of block groups
   uint nblks_per_bg;        // block group size in blocks
   uint nblks_inode_per_bg;  // number of inode blocks per block group
   uint nblks_bitmap_per_bg; // number of bitmap blocks per block group
   
   // <you may add your own fields>
   
#else
   ...
#endif
};
```

### Global Descriptor Table (GDT) 

The GDT is a small on-disk directory of block groups. It stores per-group statistics, such as free-block and free-inode counts, that can guide locality-aware allocation.

```C
// In kernel/fs.h
struct gdt_entry {
   uint freeinodes;          // number of free inodes
   uint freeblocks;          // number of free data blocks
   
   // <you may add your own fields>
   
};

struct gdt_t {
  struct gdt_entry bg[NBG]; 
};
```

### Data Bitmap Blocks

xFFS uses one data bitmap block per block group. With a 1,024-byte block size, a bitmap block can represent 8,192 blocks (1 bit per block). Because each block group in xFFS is only 2,048 blocks in size, the bitmap needs just 2,048 bits (one quarter of the bitmap block). The remaining space is left unused.

### Inode Blocks

Each block group reserves 8 blocks for inodes. With a block size of 1,024 bytes and an inode size of 64 bytes (see `struct dinode` in `kernel/fs.h`), each block holds 16 inodes, yielding 128 inodes per block group.

We slightly extend `xv6`'s inode format to support much larger files, as shown below. In the original `xv6`, an inode stores 12 direct pointers plus one single indirect pointer. xFFS trades one direct slot for an extra level of indirection, giving each inode 11 direct, 1 single-indirect, and 1 double-indirect pointers. Small files still live entirely in the direct slots with zero extra lookups. When a file grows beyond those, the single-indirect block supplies up to 256 additional data-block addresses. For truly large files, the double-indirect pointer fans out to 256 indirect pointer blocks, each of which can reference 256 data blocks, for a theoretical maximum of 11 + 256 + 256 * 256 = 65,308 data blocks (about 64.3 MiB). For this project, however, we intentionally cap the maximum file size (`MAXFILE` in `kernel/fs.h`) to 11 + 256 + 1024 data blocks (= 1,321,984 bytes) to reduce the execution time of certain test cases in `usertests`. 

```C
// In kernel/fs.h
#define NINDIRECT  (BSIZE / sizeof(uint))
#ifdef SNU
#define NDIRECT    11
#define MAXFILE    (NDIRECT + NINDIRECT + NINDIRECT * 4)
#else
#define NDIRECT    12
#define MAXFILE    (NDIRECT + NINDIRECT)
#endif

// In kernel/file.h
struct inode {
  uint dev;               // Device number
  uint inum;              // Inode number
  int ref;                // Reference counter
  struct sleeplock lock;  // protects everthing below here
  int valid;              // inode has been read from disk?

  short type;             // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
#ifdef SNU
  uint addrs[NDIRECT+2];   // Data block addresses
                           // (11 direct, 1 indirect, 1 double-indirect)
#else
  uint addrs[NDIRECT+1];   // Data block addresses
                           // (12 direct, 1 indirect)
#endif
};
```

## Problem Specification

### Part 1. Extend `mkfs` to build an xFFS disk image with large-file support (30 points)

Your first task involves extending the command-line utility `mkfs` (@ `mkfs/mkfs.c)` to properly format an xFFS file system.
The `mkfs` utility creates and initializes a file system on a disk image or storage device. It sets up essential file system structures, including the boot block, superblock, inode blocks, bitmap blocks, and data blocks. It also populates the file system image with required directories (including `/`) and initial system files so that the file system is mountable and usable from first boot. 

For xFFS specifically, you will modify `mkfs` to emit the xFFS header region (`| B | S | L | G |`), partition the remainder of the disk into fixed-size block groups, and fill in all on-disk metadata accordingly. That includes writing a superblock that records block-group geometry, constructing the Global Descriptor Table (GDT) with per-group summaries, initializing each group's bitmap and inode area, and placing the root directory and starter files. 

In addition, `mkfs` must support the double-indirect pointer for large files: the provided skeleton code includes a text file `kjv.txt` (the first two chapters of the King James Version of the English Bible), sized 406,932 bytes, which exceeds what 11 direct pointers plus a single indirect block can address. You must therefore allocate and link the double-indirect structure (the top-level double-indirect block, the necessary second-level indirect blocks, and the referenced data blocks).

Your implementation should validate that regions don't overlap, that counts match the declared geometry, and that the resulting image boots and mounts with xFFS enabled.


### Part 2. Extend the `xv6` kernel to mount and operate on xFFS (30 points)

In this part, you will extend the `xv6` kernel to understand and operate on xFFS on-disk structures. On mount, the kernel should read the superblock, recognize the xFFS magic number, and parse the GDT to learn each block group's information. Your implementation should keep the existing `xv6` interface intact (same system calls and directory semantics) but use the xFFS geometry when allocating inodes/blocks, reading/writing file data, and updating bitmaps. Be sure to implement the double-indirect addressing so that large files can be read and written correctly. 

For allocation and free, you may keep `xv6`'s simple first-fit policy for now: scan for the first free inode or data block across the entire file system (or group-by-group in order) and mark it in the corresponding per-group bitmap; implementing a sophisticated locality-aware allocator is deferred to Part 3. What matters here is correctness with xFFS's on-disk data structures. 

### Part 3. Implement a block-group-aware allocation policy (30 points)

So far, xFFS understands block groups but still allocates like the original `xv6` file system, scanning for the first free inode or data block. This naïve, global first-fit policy scatters related metadata and data, inflating the logical seek distance and undermining the whole point of block groups. 

In this part, you will design and implement your own block-group-aware allocator that keeps files near their parent directories and concentrates a directory's files within the same group whenever possible. The necessary per-group information, such as free data blocks, free inodes, etc., can be maintained in the GDT so the kernel can make fast, global decisions without walking every bitmap. For inspiration, you may consider the spirit of the [original FFS's heuristics](https://pages.cs.wisc.edu/~remzi/OSTEP/file-ffs.pdf):
  * For directories, find the block group with a low number of allocated directories and a high number of free inodes, and put the directory data and inode in that group.
  * For files, allocate the data blocks of a file in the same group as its inode. Also, place all files in a directory in the same group as that directory.
    
A similar strategy is also being used by Linux's Ext4 file system. 

As part of this task, you are also required to implement a new system call, `sync()`, to persist the GDT's per-group summaries. At mount time, the kernel reads the on-disk GDT into the in-memory `gdt` structure (`kernel/fs.c`). As files and directories are created or deleted, the corresponding GDT entries in memory are updated to reflect changes in free-inode and free-block counts, etc. 
The `sync()` system call writes these updated GDT entries back to the GDT block on disk, making them durable before you exit `xv6` (e.g., Ctrl-a x). For simplicity, we do not journal GDT updates with the `xv6`'s Logging Layer (see Chap. 10.4 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2025-2/book-riscv-rev5.pdf)), and you do not need to handle sudden power failures during `sync()`. The system call number of `sync()` has already been assigned as 51 in the `./kernel/syscall.h` file.

__SYNOPSYS__
```
  void sync(void);
```

__DESCRIPTION__

  This `sync()` system call writes the current in-memory GDT contents to the on-disk GDT block.

__RETURN VALUE__

* Always succeeds.
  

We will evaluate your allocator with a seek-distance metric that approximates head movement by grouping data blocks into tracks of 32 blocks each. On every actual device I/O, we add the absolute difference between the current block's track and the previous block's track:

```C
// kernel/virtio_disk.c
#define TRACK(b)  ((b) / 32)
#define ABS(a)    (((a) >= 0)? (a) : -(a))

uint32 d = ABS(TRACK(b->blockno) - TRACK(dd_last_blk));
dd_seek_dist += d;
dd_seek_dist_total += d;
dd_last_blk = b->blockno;
```

We will use a mix of sequential and random file accesses, plus metadata-heavy workloads. To receive full credit, your policy should reduce total seek distance by at least 20% relative to the baseline first-fit allocator. To minimize cache effects, the skeleton code provides the `bdrop()` system call that discards all cached blocks with `refcnt == 0`, forcing subsequent accesses to re-read those blocks from disk. `bdrop()` will be invoked before running any performance measurement so that results reflect on-disk access patterns rather than cache hits. 

### Part 4. Design Document (10 points)

Along with your code, submit a design document in a single PDF file. Your document should include the following sections.

1. New data structures
   * Provide details about any newly introduced data structures or modifications made to existing ones (in particular,  `superblock`, `gdt_entry`, etc.)
   * Explain why these data structures/modifications are necessary and how they contribute to the implementation.
2. Algorithm design
   * Describe the changes you made to `mkfs` to produce an xFFS image.
   * Explain how you support large files via the double-indirect pointer.
   * Provide an overview of your block-group-aware allocation policy (e.g., target group selection for directories/files, fallback behavior, etc.)
   * Describe any corner cases you considered and the strategies you used to address them.
   * Discuss any optimizations you applied to improve code efficiency, both in terms of time and space.
3. Testing and validation
   * Outline the test cases you created to validate your implementation, if any.
   * Describe how you verified the correct handling of the corner cases mentioned in Section 2.

### **Bonus** (up to an additional 20 points)

The top ten submissions with the smallest total seek distance for a given set of workloads will receive an additional 20 bonus points; the next ten will earn 10 bonus points. Only the submissions that (1) mount and operate on xFFS correctly (Part 1 & 2), (2) implement the block-group-aware allocator (Part 3), and (3) pass `usertests` on a multi-core RISC-V machine are eligible.
 
## Restrictions
* Once a file or a directory is created, its inode and data blocks must not be relocated.
* Your implementation must ensure that file system operations, such as file creation, deletion, and traversal, work seamlessly on multi-core RISC-V systems with the new on-disk layout introduced by xFFS. The correctness of your implementation will be validated using `usertests`.
* For Parts 1 and 2, we will build and run your code without special compiler options. For Part 3, however, we will compile with the `-DPART3` flag. Please wrap any Part 3-specific changes within `#ifdef PART3` ... `#endif`. If your Part 3 allocator works correctly, you do not need to keep the original allocator working, and you may omit `PART3` flag entirely.
* Please use `qemu` version 8.2.0 or later. To check your `qemu` version, run: `$ qemu-system-riscv64 --version`
* You only need to change the following files: `mkfs/mkfs.c`, `kernel/fs.h`, and `kernel/fs.c`. Any other changes will be ignored during grading.
   
## Skeleton Code

The skeleton code for this project assignment (PA5) is available as a branch named `pa5`. Therefore, you should work on the `pa5` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ git checkout pa5
```
After downloading, you must first set your `STUDENTID` in the `Makefile` again.

We briefly summarize the PA5 modifications below.  
(Note: the provided skeleton currently fails to build `fs.img` because `kjv.txt` is too large for the original allocator; your xFFS changes, including double-indirect support, are required for the image to build successfully.)

### xFFS Layout Parameters

xFFS parameters related to the file system layout are defined in `kernel/param.h`:

```C
// kernel/param.h
#define MAXOPBLOCKS  12  // max # of blocks any FS op writes
#define LOGBLOCKS    (MAXOPBLOCKS*3)  // max data blocks in on-disk log
#define GDTBLOCKS     1  // max GDT blocks
#define NBG          10  // number of block groups
#define BITMAP_PER_BG 1
#define INODE_PER_BG  8
#define BLOCKS_PER_BG 2048 // size of block group in blocks
#define FSSIZE         (BLOCKS_PER_BG * NBG + LOGBLOCKS + GDTBLOCKS + 3)
                       // +1 for boot
                       // +1 for superblock
                       // +1 for logblock header
```

In the original `xv6`, `mkfs/mkfs.c` uses `NINODES` to determine the required number of inode blocks.
In xFFS, each block group has a fixed number of inode blocks (`INODE_PER_BG` blocks). Hence, you need to adjust `NINODES` accordingly.

The file system magic number is defined as `0x000F0F05` (yes, it means "xFFS" :) in `kernel/fs.h`.

### Seek Distance

We track seek distance in `virtio_disk_rw()` (`kernel/virtio_disk.c`) using two counters: a cumulative total (`dd_seek_dist_total`) and a recent value (`dd_seek_dist`). Both are printed when you press `^p` (ctrl-p), alongside `xv6`'s process list. The recent counter resets to 0 every time you press `^p`, allowing you to measure the delta caused by a specific command. The total counter continues to accumulate throughout the entire boot session.
For example, when building `fs.img` without the `kjv.txt` file in the skeleton, you might see something like:

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic  -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$                             <--- ^p here
1 sleep  init
2 sleep  sh
recent seek distance:	6
total seek distance:	6

$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2425
cat            2 3 36184
echo           2 4 35064
forktest       2 5 17064
grep           2 6 39624
init           2 7 35528
kill           2 8 34992
ln             2 9 34816
ls             2 10 38144
mkdir          2 11 35056
rm             2 12 35048
sh             2 13 57872
stressfs       2 14 35928
usertests      2 15 184408
grind          2 16 50800
wc             2 17 37128
zombie         2 18 34424
logstress      2 19 36952
forphan        2 20 35808
dorphan        2 21 35256
console        3 22 0
$                             <--- ^p here
1 sleep  init
2 sleep  sh
recent seek distance:	2
total seek distance:	8
```

### The `bdrop()` and `sync()` System Calls

The `bdrop()` system call is implemented in `bdrop() @ kernel/bio.c`, which invalidates all buffer-cache entries whose reference count is zero. 
`bdrop()` also resets the last head position (`dd_last_blk` in `kernel/virtio_disk.c`) to block 0, ensuring subsequent measurements start from a known position and remain reproducible.

You need to complete `sys_sync()` in `kernel/sysfile.c` to flush the in-memory GDT structure to the on-disk GDT block. 

For convenience, the shell (`sh`) has been extended to recognize `bdrop` and `sync` as built-in commands, so you can invoke them directly at the shell prompt without launching external binaries.


### The `fstest` utility

The skeleton includes a user-level utility called `fstest` (in `user/fstest.c`), which is a tiny program that exercises xFFS reads and writes with three file sizes: `fsmall` (5 blocks), `fmedium` (20 blocks), and `flarge` (300 blocks). These sizes are chosen so that the first fits entirely in direct blocks, the second spills into the single-indirect block, and the third requires the double-indirect block. Each read or write path calls `bdrop()` before opening the file, invalidating cached buffers so that subsequent I/O actually hits the disk rather than the buffer cache.

Run `fstest` with no arguments to write and then read all three files in order.
```sh
$ fstest       # write "fsmall", "fmedium", and "flarge" and then read them back in order
```

You can target a single file by index (0 = small, 1 = medium, 2 = large). For example, the following writes and then reads only `flarge`.
```sh
$ fstest 2     # write "flarge" and then read it back.
```

You can also request an operation to a single file explicitly:
```sh
$ fstest w 1   # write only "fmedium"
$ fstest r 0   # read only "fsmall"
```

## Tips
  
* Read Chap. 10 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2025-2/book-riscv-rev5.pdf) to understand the file system implementation in `xv6`.

* For your reference, the following roughly shows the required code changes; each `+` denotes about 1~10 lines to add, remove, or modify.
  ```
  mkfs/mkfs.c          |  +++++++++
  kernel/defs.h        |  +
  kernel/fs.h          |  ++
  kernel/fs.c          |  ++++++++++++++++++
  kernel/sysfile.c     |  +
  ```
  

## Hand in instructions

* First, make sure you are on the `pa5` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. In addition, you must also upload your design document as a PDF file for this project assignment.

* The total number of submissions for this project assignment will be limited to 30. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Now, adding your new IP address is automated by a script that periodically checks the Google Form at minutes 0, 20, and 40 during the hours between 09:00 and 00:40 the following day, and at minute 0 every hour between 01:00 and 09:00.
     + If you cannot reach the server a minute after the update time, check your IP address, as you might have sent the wrong IP address.
     + If you still cannot access the server after some time, it is likely due to an error in the automated process. The TAs will verify whether the script is running correctly, but since this check must be performed __manually__, please understand that it may not be completed immediately.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delayed.
* __You can use up to _3 slip days_ during this semester__. If your submission is delayed by one day and you decide to use one slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use on the QnA board of the submission server before the next project assignment is announced. Once slip days have been used, they cannot be canceled later, so saving them for later projects is highly recommended!
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)


