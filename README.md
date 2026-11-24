# 4190.307 Operating Systems (Fall 2026)
# Project #5: deFAT File System
### Due: 11:59 PM, December 20 (Sunday)

## Introduction

Microsoft's early FAT file system has a unique file index structure based on the ___File Allocation Table (FAT)___. The goal of this project is to implement deFAT, a defragmentation-aware FAT file system, which modifies the file index structure of the existing `xv6` file system to resemble that of the FAT file system. You will make the new on-disk structures crash-consistent by routing every metadata update through the `xv6` logging layer, and you will implement a `defrag()` system call that reorganizes the on-disk layout to eliminate fragmentation.

## Background

### Microsoft FAT File System

The FAT file system, developed by Microsoft in 1977, is one of the earliest and simplest file systems. FAT uses a table at the beginning of a disk to manage files and directories. Each entry in the table corresponds to a cluster (i.e., a block), a fixed-size unit of disk storage. The table maintains pointers to the next cluster in a file, allowing for sequential access and easy file allocation.

FAT exists in several versions, including FAT12, FAT16, and FAT32, each extending the maximum storage capacity and improving performance. FAT32, for example, supports larger disks and smaller cluster sizes, making it suitable for a wider range of applications. Despite being somewhat outdated, FAT remains relevant in specific contexts, such as USB drives and memory cards, due to its broad operating system support and simplicity. For more information on the FAT file system, visit [this page](https://en.wikipedia.org/wiki/File_Allocation_Table).

### File Allocation Table (FAT)

The name of the FAT file system itself stems from a unique data structure called the File Allocation Table (FAT). This table, located at the beginning of the disk, contains information on the file index, specifically the locations of the blocks belonging to each file or directory. 
Assume that the first block number for the file `foo` is 2, and for the file `bar` is 5. In the FAT file system, this information on the first block number is recorded in the directory entry.

Let's consider the following FAT example. The FAT has an entry for each block, and each entry points to the next block number in the file. From the directory entry, we know that the file `foo` starts at block 2. Then, `FAT[2]` has the next block number for the file, which is 3 in this case. Similarly, the subsequent block numbers can be found by looking up the FAT in sequence: `FAT[3] = 4`, `FAT[4] = 7`, and `FAT[7] = 8`. Finally, `FAT[8]` has the value of -1 which denotes the end of the file. Therefore, we can see that the file `foo` is stored in five blocks: 2, 3, 4, 7, and 8. In the same way, we can find that the file `bar` is located at blocks 5, 6, 9, and 10. The FAT entry marked as zero indicates that the corresponding block has not been allocated to any file yet. Additionally, reserved blocks have the value of -2 in the corresponding FAT entries. The size of each FAT entry is 12 bits, 16 bits, and 32 bits in FAT12, FAT16, and FAT32, respectively.


```
FAT[0]  FAT[1]  FAT[2]  FAT[3]  FAT[4]  FAT[5]  FAT[6]  FAT[7]
 -2      -2       3       4       7       6       9       8 

FAT[8]  FAT[9]  FAT[10] FAT[11] FAT[12] FAT[13] FAT[14] FAT[15]
 -1       10     -1       0       0       0       0       0 

...
```

_Note that the above example uses the sentinel values of the original Microsoft FAT file system. The deFAT file system you are going to implement in this project uses different values; see the "Minor Changes from the FAT File System" section below._

## The deFAT File System

### The On-disk Layout

The following illustrates the on-disk layout of the current `xv6` file system when the file system size is set to 2000 blocks (`FSSIZE` in `kernel/param.h`). 

```
+---+---+-------+------+---+----------------------------------------+
| B | S |   L   |  I   | M |                D                       |
+---+---+-------+------+---+----------------------------------------+
```
* `B`: Boot block (1 block) — Not used
* `S`: Superblock (1 block)
* `L`: Log blocks (30 blocks) (cf. `LOGBLOCKS` @ `kernel/param.h`)
* `I`: Inode blocks (13 blocks) (cf. `NINODES` inodes @ `mkfs/mkfs.c`)
* `M`: Free bitmap blocks (1 block) (cf. `FSSIZE` blocks @ `kernel/param.h`)
* `D`: Data blocks (1954 blocks) 

In contrast, the on-disk layout of the new deFAT file system is as follows. After the Log (`L`) blocks, the deFAT file system has a region designated for FAT (`F`) blocks. FAT entries cover the entire file system blocks, with each FAT entry being 16 bits long, as in FAT16. Therefore, one FAT block is required for every 512 file system blocks. (Note: The block size is set to 1024 bytes in `xv6`. Refer to `BSIZE` in `kernel/fs.h`.)

Unlike the original `xv6` file system, the deFAT file system does not require the Free bitmap (`M`) blocks because the FAT blocks contain the information on the free data blocks. 

```
+---+---+-------+------+---+----------------------------------------+
| B | S |   L   |  F   | I |                D                       |
+---+---+-------+------+---+----------------------------------------+
```
* `B`: Boot block (1 block) — Not used
* `S`: Superblock (1 block) 
* `L`: Log blocks (43 blocks) (a header plus `LOGBLOCKS` @ `kernel/param.h`)
* `F`: FAT blocks (4 blocks) (cf. `FSSIZE` blocks @ `kernel/param.h`)
* `I`: Inode blocks (4 blocks) (cf. `NINODES` inodes @ `mkfs/mkfs.c`)
* `D`: Data blocks (1947 blocks)

The file system size (`FSSIZE` @ `kernel/param.h`), the number of inodes (`NINODES` @ `mkfs/mkfs.c`), and the number and size of the pre-installed programs (`UPROGS` @ `Makefile`) can all vary between the images used for grading. However, `FSSIZE` will never exceed `FSSIZE_MAX` = 2048 blocks, so the FAT region never occupies more than 4 blocks.

### Minor Changes from the FAT File System

1. The deFAT file system has a magic number `0x44464154`, where each byte represents 'D', 'F', 'A', and 'T', respectively.

2. In the original FAT file system, the File Allocation Table (FAT) is duplicated to provide redundancy and reliability, protecting against data corruption. In the deFAT file system, however, we maintain only one copy of the FAT blocks for simplicity.

3. Each FAT entry is encoded as an unsigned 16-bit integer, with the following values:
   * Positive values (> 0): denote the next block number
   * Zero (0): denotes the end of the chain
   * `0xFFFF`: indicates that the corresponding block is reserved (this applies to the entries for the boot block, superblock, log blocks, FAT blocks, and inode blocks)

   Note that this differs from a real FAT16 volume, where `0x0000` marks a *free* cluster and `0xFFF8`-`0xFFFF` marks the end of a chain. In deFAT, the free blocks are chained together (see 5 below), so a free block's entry has to hold the number of the next free block. Therefore, you __cannot__ tell from a single FAT entry whether a block is in use — an entry reading '57' is either a file block pointing at block 57 or a free block pointing at free block 57.

4. In the original FAT file system, the first block number is kept in the directory entry. However, in the deFAT file system, this information is stored in the inode's `startblk` field. An empty file (i.e., one with no data block allocated yet) has `startblk` of zero.

5. The unallocated (free) data blocks are also linked together via FAT entries, so that a block can be allocated and freed in constant time without searching. The head of the free block list is maintained in the `freehead` field of the superblock and the total number of free blocks in its `freeblks` field.

6. The FAT blocks and the superblock are ordinary disk blocks, reached through the buffer cache like any other metadata. ___Every___ modification to a FAT block or to the superblock must be performed on a buffer obtained with `bread()` and handed to `log_write()` inside the `begin_op()`/`end_op()` pair of the operation that caused it. Once an operation's transaction commits, its FAT and superblock changes are durable; if it does not commit, none of them are. (The `sync()` system call you already have waits for the log to commit. It is inherited from the base `xv6` and needs no change for this project.)

7. Because an inode no longer contains a fixed-size block address array, the maximum file size is no longer limited to `MAXFILE` blocks as in the original `xv6` file system. A single file in the deFAT file system can grow until it occupies the entire data region. (You will still find `MAXFILE` defined in `kernel/fs.h`; it is not a limit, only a size that `usertests` uses.)

### Superblocks and Inode for the deFAT file system

The superblock of the deFAT file system contains the following information.

```C
// @ kernel/fs.h
struct superblock {
  uint magic;        // Must be FSMAGIC
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
#ifdef SNU
  uint nfat;         // Number of FAT blocks
  uint fatstart;     // Block number of first FAT block
  uint freehead;     // Head of the free block list
  uint freeblks;     // Number of free data blocks
#else
  uint bmapstart;    // Block number of first free map block — Not needed for PA5
#endif
};
```

The inode of the deFAT file system contains the following information.

```C
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
#ifdef SNU
  uint startblk;        // First block number
#else
  uint addrs[NDIRECT+1];   // Data block addresses — Not needed for PA5
#endif
};
```

### Notations

The rest of this document uses the following terms.

* The __data region__ is the part of the file system that follows the inode blocks. It holds $D$ data blocks numbered $d$, $d+1$, $\ldots$, $d+D-1$, where $d$ is the number of the first data block and $D$ is the superblock's `nblocks`.
* A __file__ means any inode that is in use. Directories count as files here, and so does an inode that has been unlinked but is still open.
* The __chain__ of a file $f$ is the sequence of data blocks

  $$b_f[0], b_f[1], \ldots, b_f[n_f - 1]$$

  reached by starting at that inode's `startblk` and following FAT entries until `FAT_EOC` $(= 0)$. So $n_f$ is the number of blocks in the file $f$: a file of $s$ bytes has $n_f = \lceil s / \mathtt{BSIZE} \rceil$. An empty file has $n_f = 0$ and no chain at all. When only one file is under discussion, the subscript is dropped and its chain is written $b[0], b[1], \ldots, b[n-1]$.
* A __link__ of $f$ is a consecutive pair $(b_f[i], b_f[i+1])$ for $0 \le i < n_f - 1$. A file of $n_f$ blocks therefore has $n_f - 1$ links, and a file of zero or one block has none. $K$ is the __total number of links in the file system__, that is, the sum of $n_f - 1$ over every file with $n_f \ge 2$.
* $u$ is the number of data blocks __in use__, that is, the sum of $n_f$ over all files. The other $D - u$ data blocks are __free__ and are exactly the blocks on the free list.

## Problem specification

### 1. Modify the `mkfs` tool to create the correct on-disk image (20 points)

To set up the deFAT file system, you need to start by modifying the `mkfs/mkfs.c` file to generate the correct image file, `fs.img`. This image file should include the new FAT blocks and exclude the Free bitmap blocks. The FAT blocks must be positioned between the Log blocks and the Inode blocks. Additionally, you should correctly initialize the free block list and the corresponding superblock fields such as `freehead` and `freeblks`. Your `mkfs` must produce a correct image for any legal combination of `FSSIZE` (up to `FSSIZE_MAX`), `NINODES`, and `UPROGS`.


### 2. Replace the file index structure with FAT (30 points)

The current `xv6` file system uses 12 direct pointers and one indirect pointer within the inode to store the locations of data blocks associated with a file or directory. In the deFAT file system, however, each inode only contains a pointer (`startblk`) to the first data block. The subsequent block locations should be looked up in the FAT. Additionally, you need to modify the data block allocation and deallocation functions to utilize the new free list. When a data block is allocated or deallocated, ensure that the superblock's `freeblks` value is updated accordingly. The skeleton code prints this value whenever you press `^f` (`ctrl-f`) in the console. This value will be checked to determine whether your implementation has space leaks or not during various file system operations. 

Your implementation must work correctly on multi-processor systems with multiple processes accessing the file system simultaneously. In particular, the free block list — whose head and length live in the superblock and are shared by every file system operation — must stay consistent under concurrent allocation and deallocation: no block may be handed out to two files, no block may be lost from the list, and `freeblks` must always agree with the list's actual length. Your synchronization scheme must also be free of deadlock.

We provide you with a sample deFAT file system image, `fs-defat.img`, in the skeleton code so that you can begin Part 2 of this project before completing Part 1.

### 3. Make the deFAT file system crash-consistent (20 points)

In the original `xv6`, inode blocks and directory blocks survive a crash because every change to them is made inside a transaction and written to the log first (see Chap. 11 of the [xv6 book](https://mit-pdos.github.io/xv6-riscv-book/log.html)). In the deFAT file system, the FAT and the free-list fields of the superblock are metadata of exactly the same kind, and they must be given exactly the same treatment.

Your implementation must satisfy the following:
* __(a) Write-ahead__: Every change to a FAT block or to the superblock is made on a buffer returned by `bread()` and passed to `log_write()` before the enclosing `end_op()`. No FAT block or superblock may ever be written to the disk by a path that bypasses the log.
* __(b) Bounded transactions__: No file system operation may touch more than `MAXOPBLOCKS` distinct blocks in a single transaction. Pay particular attention to operations whose cost grows with file size, such as truncating or deleting a file that spans the entire data region.
* __(c) Atomicity__: After a crash at any point, and after the log has been replayed at the next boot, each file system operation must appear to have either completed entirely or not started at all.
* __(d) Consistency__: After recovery, the FAT must be well-formed: the chains reachable from the `startblk` of every allocated inode, together with the free list reachable from `freehead`, must partition the data region exactly. No block may appear in two chains, no block may be missing from all of them, and `freeblks` must equal the actual length of the free list.
* __(e) Recovery order__: The in-memory copy of the superblock must reflect the state of the file system *after* log recovery, not before it. Read `fsinit()` carefully.

During grading, we will run a workload of file system operations and terminate the emulator at a randomly chosen disk write. We then reboot on the same image and verify conditions (c) and (d) with a checker. The test is repeated at many different interruption points, including while a transaction is being committed and while the committed blocks are being copied from the log back to their home locations.

### 4. Implement the `defrag()` system call (20 points + up to 20 bonus points)

The FAT-based file index structure has one well-known weakness: because the location of the $n$-th block of a file can only be found by walking the chain from `startblk`, a fragmented file is expensive to access. Real FAT file systems address this with a periodic defragmentation pass that rewrites the on-disk layout so that each file occupies a consecutive run of blocks. 

Your final task is to implement the `defrag()` system call, which reorganizes the entire data region of the deFAT file system. When `defrag()` returns successfully, the file system must satisfy __all__ of the following conditions:

* __(a) Contiguity__: Every file occupies a consecutive run of blocks: $b_f[i+1] = b_f[i] + 1$ for every file $f$ and every $0 \le i < n_f - 1$.
* __(b) Compaction__: The $u$ blocks in use are packed at the beginning of the data region, that is, they are exactly the blocks $d, d+1, \ldots, d+u-1$, with no free block among them.
* __(c) Free list__: The free block list contains exactly the remaining data blocks, linked in ascending order of block number, with `freehead` pointing to the lowest-numbered free block.
* __(d) Data integrity__: The contents and the size of every file and directory are completely unchanged.
* __(e) No space leak__: The value of `freeblks` is identical to its value before the call.
* __(f) Crash safety__: In addition, `defrag()` is subject to the crash-consistency requirements of Part 3. `defrag()` will not fit in a single transaction, and it is not required to be atomic as a whole. It must, however, be carried out as a sequence of transactions such that a crash at any point leaves — after recovery — a file system that satisfies (d) and (e) and the consistency condition 3(d), even though it may be only partially defragmented. A crash during `defrag()` must never lose, duplicate, or corrupt a single block of user data.

Note that conditions (a) through (c) do __not__ constrain the relative order of files within the data region. You are free to choose which file is placed first, second, and so on; this freedom is the basis of the bonus points described below.

These conditions constrain only the state of the file system at the moment `defrag()` returns. How the free block list is organized during ordinary file system operations is entirely up to you.

You may assume that no other file system operation is in progress while `defrag()` is running. Open file descriptors, however, must remain valid: a process that had a file open before the call must be able to continue reading and writing it correctly afterwards. (Recall that a `struct file` holds a byte offset, not a block number.)


__SYNOPSIS__
```
    int defrag(void);
```

__DESCRIPTION__

The `defrag()` system call reorganizes the data region of the file system so that every file and directory occupies a consecutive run of data blocks, and so that all allocated data blocks are packed at the beginning of the data region.

__RETURN VALUE__

* On success, `defrag()` returns the number of data blocks that have been relocated (i.e., the number of blocks whose block number has changed).
* `defrag()` returns -1 if the file system cannot be defragmented.

#### Fragmentation metrics and bonus points 

We measure the result of your `defrag()` implementation with four quantities. $F$ and $G$ describe the state of the file system afterwards and are pass/fail; $M$ and $W$ describe what the call cost, and $W$ is what the bonus points are awarded on.

* __Fragmentation ($F$)__: the fraction of all links whose two blocks are not adjacent on the disk,

  $$F = \frac{1}{K} \sum_{\text{files } f} \lvert \\{ i : 0 \le i < n_f - 1 \text{ and } b_f[i+1] \ne b_f[i] + 1 \\} \rvert$$

  taking $F = 0$ when $K = 0$. A correctly defragmented file system has $F = 0$.

* __Free space fragmentation ($G$)__: $G = R / (D - u)$, where $R$ is the length of the longest run of consecutive free data blocks, taking $G = 1$ when $u = D$. A correctly compacted file system has $G = 1$.

* __Relocations ($M$)__: the number of data blocks whose block number changes during the call. This is exactly the value `defrag()` returns.

* __Write traffic ($W$)__: the total number of block writes that the call issues to the disk driver, counting the writes the logging layer makes on your behalf — both the copies into the log and the installs out of it — as well as the data copies themselves.

The 20 base points require $F = 0$, $G = 1$, and conditions (d) through (f) above, on every test image.

__The single metric you are competing on is $W$__, summed over test images of differing initial fragmentation, one of which is nearly full. Among the submissions that earn the full base points, the smallest $W$ wins. __The top five submissions will receive 20 bonus points, and the next five 10.__ Ties in $W$ are broken by $M$, the smaller winning; any tie remaining is broken by the upload time of the version you marked `FINAL`, the earlier winning.

You can measure all four quantities yourself. $M$ is what your own `defrag()` returns. For $W$, the skeleton counts every block write the disk driver issues in the global variable `diskwrites` (declared in `kernel/defs.h`). You can read it two ways: press `^w` (`ctrl-w`) in the console to print the running total, or read the variable directly from your own kernel code. For $F$ and $G$, the skeleton provides the `fragstat` tool, which inspects a file system image offline and also performs the consistency check.


### 5. Design document (10 points)

You need to prepare and submit the design document (in a single PDF file) for your implementation. Your design document should include the following:

1. New data structures
   * Describe any new data structures introduced for this project.
   * Explain the purpose and functionality of each data structure.
   * Provide insights into why these data structures were necessary and how they contribute to the implementation.
2. Algorithm design
   * Describe the algorithm used for implementing the deFAT file system.
   * Describe the strategy used to synchronize simultaneous accesses to the FAT and the free block list, and explain why it cannot deadlock.
   * Describe the algorithm used for your `defrag()` implementation, and explain how you decided the order in which files are placed in the data region.
   * Explain any corner cases you have considered and the strategies you used to address them.
   * Discuss any efforts you made to enhance the code efficiency, both in terms of time and space.
3. Crash consistency
   * Enumerate the blocks that your implementation adds to a transaction for each of the following operations: creating a file, appending one block to a file, creating a directory, and deleting a file that occupies the entire data region.
   * Describe what your `defrag()` leaves on the disk if the machine loses power in the middle of relocating a block, and argue that no data can be lost.
4. Testing and validation
   * Describe the testing strategy used to verify the implementation of the deFAT file system.
   * Discuss how you verify the correct handling of the corner cases highlighted in Section 2 and the crash-consistency conditions of Section 3.

## Restrictions

* Your implementation should pass `usertests` on multi-processor RISC-V systems (i.e., `CPUS` > 1).
* There should be no space leaks in the file system. 
* You only need to modify files in the `./kernel` and `./mkfs` directories. Any changes made outside these directories will be ignored during grading.
* Do not change `MAXOPBLOCKS`, `LOGBLOCKS`, or `NBUF` in `kernel/param.h`, and do not change the chunk size in `filewrite()`. They are fixed so that every submission is measured on the same workload.
* Do not modify `kernel/virtio_disk.c`. It holds the disk write counter that $W$ is measured with, and any change to the `diskwrites` variable will be treated as an integrity violation. Read the counter only; do not reset, change, or otherwise interfere with it.

## Skeleton code

The skeleton code for this project assignment (PA5) is available as a branch named `pa5`. Therefore, you should work on the `pa5` branch as follows:

```
$ cd xv6-riscv-snu
$ git fetch --all
$ git checkout pa5
```

After downloading, you must first set your `STUDENTID` in the `Makefile` again.

The `pa5` branch includes a sample deFAT file system image, `fs-defat.img`. Using this image file, you can start Part 2 without completing Part 1 of this project. If you want to use this image file, copy it to `fs.img` before running `xv6`.

The skeleton code provides the `fragstat` tool, which inspects a file system image offline and also performs the consistency check of Part 3 when invoked with `-c`:

```sh
$ ./mkfs/fragstat fs.img          # F, G, compaction and free-list order
$ ./mkfs/fragstat -v fs.img       # the same, plus a line per file
$ ./mkfs/fragstat -c fs.img       # the consistency check of Part 3 only
$ ./mkfs/fragstat -f name fs.img  # one file's layout, with its own F
```

The `-f` option prints the data blocks of a single file as a list of consecutive runs, followed by that file's own fragmentation. $A$ is the number of places where the chain is not consecutive — that is, the numerator of $F$ for that one file — so a file laid out in a single run has $A = 0$ and $F = 0$:

```
$ ./mkfs/fragstat -f cat fs.img
cat: 57-93 (37 blocks), F = 0.0000, A = 0

$ ./mkfs/fragstat -f foo fs.img
foo: 900-903, 910, 920-921 (7 blocks), F = 0.3333, A = 2

$ ./mkfs/fragstat -f bar fs.img
bar: 1450, 1449, 1448, 1432 (4 blocks), F = 1.0000, A = 3
```

The last example is worth a second look. The file `bar` occupies three adjacent blocks and one distant one, yet $F = 1$: its chain visits 1450, then 1449, then 1448, so __every__ link is broken. Condition 4(a) is about the order in which a file walks its blocks, not about the set of blocks it owns.

These are the same tool and the same counter we use for grading, so the numbers you see are the numbers you will be ranked on.

The system call number of `defrag()` is already assigned to 25 in the `kernel/syscall.h` file, and the user-level stub is already provided in the skeleton code.


Note: The skeleton code builds, but it cannot mount a file system yet. `make qemu` stops at `panic: invalid file system`, because the kernel expects a deFAT image while `mkfs` still produces the original `xv6` file system image. Copying `fs-defat.img` over `fs.img` gets you past that check and into the failures that Part 2 exists to fix.

## Tips

* Read Chap. 10 of the [xv6 book](https://mit-pdos.github.io/xv6-riscv-book/) to understand the file system in `xv6`, and Chap. 11 in particular for the logging layer.

* The logging layer is indifferent to what a block contains: `log_write()` takes a `struct buf *`, and a FAT block is just another block. You should not need to modify `kernel/log.c` at all. What you do need to get right is which buffers you hand to it, and how many.

* `log_write()` already absorbs repeated writes to the same block within one transaction. Allocating a run of consecutive blocks therefore writes fewer FAT blocks than allocating the same number of scattered ones.

* For your reference, the following roughly shows the amount of changes you need to make for this project assignment. Each `+` symbol indicates 1~10 lines of code that should be added, deleted, or altered.
```
 kernel/fs.h      | +++++++
 kernel/file.h    | +
 kernel/fs.c      | ++++++++++++++++++++++++++++++++++++++++++++++++++++
 mkfs/mkfs.c      | +++++++++++
```
  
## Hand in instructions

* First, make sure you are on the `pa5` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. Upload this file with your design document (in PDF format) to the submission server.
  
* The total number of submissions for this project assignment will be limited to 30. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Adding your new IP address is automated by a script that periodically checks the Google Form every 10 minutes.
     + If you cannot reach the server 10 minutes after submitting the Google Form, send the request again, as you might have sent the wrong IP address.
     + If you still cannot access the server after a while, that is likely due to an error in the automated process. The TAs will check if the script is properly running, but that is a ___manual___ process, so please do not expect it to be completed immediately.


## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delayed.
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
