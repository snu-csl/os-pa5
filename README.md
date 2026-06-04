# 4190.307 Operating Systems (Spring 2026)
# Project #5: CROFS: Compressed Read-Only File System
### Due: 11:59 PM, June 21 (Sunday)

## Introduction

In this project, you will implement **CROFS**, a compressed read-only file system for `xv6`.
CROFS is inspired by modern read-only compressed file systems such as [EROFS](https://docs.kernel.org/filesystems/erofs.html), but it is intentionally simplified for `xv6`. CROFS stores directories and metadata in an ordinary uncompressed form, while regular file contents may be compressed into independently readable chunks. The goal is to reduce the disk image size while preserving the standard `xv6` user interface. The system should boot, run the shell, execute user programs, traverse directories, and read files normally.

## Background

### The Original of `xv6` File System

The original `xv6` file system uses a simple on-disk layout:

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

The original `xv6` file system is writable. It supports allocation and freeing of blocks and inodes, and it uses the logging layer to make updates crash-consistent. Its file-system block size is 1 KiB (`BSIZE == 1024`). The buffer cache, disk driver, inode block mapping code, and many file-system helpers operate in units of 1 KiB blocks.

## CROFS

### CROFS Overview

CROFS is read-only. The entire image is produced ahead of time by a new host-side image builder called `mkcrofs` in `mkfs/mkcrofs.c`. The kernel does not allocate or free CROFS blocks. It only interprets CROFS metadata and returns the uncompressed bytes requested by user programs.

From the perspective of the kernel file interface, file data must be readable at arbitrary byte offsets. In the original `xv6` file system, each logical file block maps directly to one 1 KiB disk block. This makes random access simple: if a process reads from file offset `off`, the kernel computes the logical block number, locates the corresponding disk block, and reads that block.
However, compression breaks this one-to-one mapping. A 1 KiB region of uncompressed file data may become 300 bytes, 500 bytes, 1 KiB, or even slightly larger after compression. Therefore, CROFS must store additional metadata that maps logical file offsets to its compressed bytes on disk.

A CROFS implementation may internally compress larger or smaller regions of a file.
A large compression unit may improve the compression ratio because the compressor sees more repeated data. However, a large compression unit also increases the cost of a small random read: to return one byte or one 1 KiB block, the kernel may have to read and decompress a much larger compressed region.
A small compression unit usually gives faster random reads and reduces temporary memory usage, but it may reduce the compression ratio. 

This trade-off is part of the project. CROFS does **not** require a fixed compression unit. You may choose your own compression unit size, indexing scheme, and packing strategy. However, your implementation must still serve reads of any individual 1 KiB logical file block efficiently and must not require decoding a large file from the beginning.

### CROFS On-disk Layout 

CROFS deliberately leaves the data-region layout flexible so that you can choose your own compression format, indexing scheme, and block-packing strategy. At a high level, a CROFS image must follow this layout:

```
+-+-+-----+-------------------------------+
|B|S|  I  |             D                 |
+-+-+-----+-------------------------------+
```
* `B`: Boot block (block 0) -- Not used
* `S`: CROFS superblock (block 1) 
* `I`: Inode blocks (uncompressed)
* `D`: Data blocks (files, directories, and other metadata)
  
Only the boot block, superblock, and inode region have fixed locations. The size of the inode region is determined by the actual number of files and directories included in the image. After the inode region, you may freely organize the rest of the image. For example, regular files and directories may be stored either compressed or uncompressed, and implementations may place any necessary metadata.

### CROFS Superblock

You may freely extend the superblock, but the following fields are required for grading:


```C
// On-disk CROFS superblock structure (@ kernel/fs.h)
#define CROFS_MAGIC   0x43524653    // "CRFS"

struct superblock {
  uint magic;         // Must be CROFS_MAGIC
  uint size;          // Size of file system image (blocks)
  uint nblocks;       // Number of data blocks
  uint ninodes;       // Number of inodes.
  uint inodestart;    // Block number of first inode block
  uint datastart;     // Block number of first data block
  uint plainsize;     // Size of uncompressed file system image (blocks)
};

  // You may add more fields here...
};
```

The `size` field is important. It denotes the number of 1 KiB disk blocks actually used by the CROFS image, which is used for calculating the compression ratio.

The grading script will verify `size` in several ways:

1. The image builder (`mkcrofs`) must print `size` when building `fs.img`.
2. The CROFS superblock must contain the same `size` value.
3. The generated image should store all required blocks below `size`.
4. The disk driver checks that CROFS never reads a block number greater than or equal to `size`.

### CROFS Inode

The on-disk inode size is fixed to 64 bytes, as in the original `xv6` file system.
You may modify the inode format except for the first five fields (`type`, `major`, `minor`, `nlink`, and `size`), which are required for compatibility with the existing inode layer:

```C
// On-disk CROFS inode structure (@ kernel/fs.h)
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)

  // The remaining fields may be repurposed for CROFS
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

For regular files, `size` must be the uncompressed logical file size. System calls such as `read()`, `lseek()`, `fstat()`, and `exec()` should observe this uncompressed size.

### Compression Algorithms

You may use any of the existing compress/decompress algorithms such as [RLE](https://en.wikipedia.org/wiki/Run-length_encoding), [Huffman coding](https://en.wikipedia.org/wiki/Huffman_coding), [LZSS](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Storer%E2%80%93Szymanski), [LZ4](https://en.wikipedia.org/wiki/LZ4_(compression_algorithm)), etc.

A compression unit is the amount of original file data that your `mkcrofs` tool compresses together and that your kernel may need to decompress together. You may choose this unit freely and even vary it across files or regions if your metadata supports it.

The following rules apply:
* A read at any byte offset must return the same bytes as the original uncompressed file.
* A random read near the end of a large file must not require scanning or decompressing from the beginning of the file.
* Your metadata must let the kernel find the compression unit containing a requested offset efficiently.
* If compression makes a region larger, you should store that region uncompressed.
* Your design document must state the maximum compression-unit size used by your implementation.

### Interfacing with the Buffer Layer

CROFS must continue to use `xv6`'s existing buffer layer as the only mechanism for reading disk blocks. All on-disk compressed data must be obtained through `bread()`, and the return buffer must be released with `brelse()` when CROFS is done using it. This preserves `xv6`'s normal block-cache behavior: the buffer cache may keep the disk block in memory, synchronize access to it, and avoid repeated disk I/O for popular blocks. In `xv6`, `bread()` returns a locked buffer containing a disk block and `brelse()` releases it when the caller is finished.

For compressed files, the data stored in these `xv6` buffers is still the compressed on-disk representation, not the decompressed file contents. CROFS may decompress one or more compressed blocks or compression units in order to satisfy a read, but the decompressed output should be treated as temporary working data for that read operation.

After decompression, CROFS may copy the requested bytes from an internal decompression buffer into the user's destination buffer. Once the read operation no longer needs that temporary decompressed data, CROFS must release or reuse the internal buffer. CROFS must not keep a separate hidden cache of decompressed file data, decompressed compression units, or entire decompressed files.

The following designs are **not** acceptable:

* Reading compressed disk blocks by bypassing `bread()`.
* Keeping compressed disk data in a private CROFS cache instead of relying on the `xv6` buffer cache.
* Treating the temporary decompression buffer as a long-lived cache
* Keeping decompressed data for reuse across later reads.
* Keeping 1 KiB decompressed file blocks in a separate CROFS cache.
* Keeping the entire decompressed file or file system permanently in memory.
* Keeping hidden decompressed copies outside the lifetime of the read operation.

In short, `xv6`'s buffer cache may cache the **compressed disk blocks** returned by `bread()`. CROFS's decompressed data is only temporary read-time state and must not introduce another caching layer.

### Fast 1 KiB Random Reads

The grading script will measure random reads at 1 KiB granularity. Your CROFS design must handle this correctly and efficiently for every regular file, including files whose sizes are not multiples of 1024.

This does **not** mean your compression unit must be 1 KiB. It means your implementation must provide a fast path from a requested 1 KiB logical file block to the compressed region that contains it. Whole-file compression without an index is not acceptable for large files because it would make random reads require decoding from the beginning of the file. 

## Problem Specification

### Part 1. Implementing `lseek()` System Call (10 points)

Your first task is to implement the `lseek()` system call. The system call number is assigned to 30 in `kernel/syscall.h`.

__SYNOPSIS__

```C
int lseek(int fd, int offset, int whence);
```

__DESCRIPTION__

`lseek()` changes the current file offset of an open file descriptor.

`whence` must be one of:
```C
#define SEEK_SET   0   // new offset = offset
#define SEEK_CUR   1   // new offset = current offset + offset
#define SEEK_END   2   // new offset = file size + offset
```

For this project, `lseek()` is required only for regular files. You may return -1 for pipes and device files.

__RETURN VALUE__

* On success, `lseek()` returns the new file offset.
* On failure, `lseek()` returns -1.

__REQUIREMENTS__

* `lseek(fd, 0, SEEK_SET)` moves to the beginning of the file.
* `lseek(fd, 0, SEEK_CUR)` returns the current offset.
* `lseek(fd, 0, SEEK_END)` moves to the end of the file.
* Negative resulting offsets are invalid and must return -1.
* Seeking past the end of a file is allowed, but a later `read()` should return 0 if the offset is beyond EOF.

We need `lseek()` because the CROFS grading script will perform random reads by repeatedly seeking to different offsets and then calling `read()`.

### Part 2. Building CROFS Image with `mkcrofs` (30 points)
 
You will implement a host-side file system image builder called `mkcrofs`.
You are free to design the layout of the data-block region. However, `mkcrofs` must perform the following tasks:

* Create a bootable CROFS image named `fs.img`.
* Write the CROFS superblock to block 1.
* Create an inode region containing only files and directories included in the image.
* Preserve the original `xv6` directory-entry format.
* Store regular-file contents using your chosen compression scheme.
* Fall back to uncompressed storage when compression does not reduce space usage.
* Correctly populate device files, such as `/console`.
* Report `size` and other required superblock fields accurately.
* Ensure that no data is stored past the block indicated by `size`.
  
Also, your `mkcrofs` tool must print a summary of the generated CROFS image. The output must include the superblock fields `magic`, `size`, `nblocks`, `ninodes`, `inodestart`, `datastart`, `plainsize`, and the compression ratio, as shown below:
```
magic:              <CROFS magic number>
size:               <total number of 1 KiB blocks in the CROFS image>
nblocks:            <number of data blocks>
ninodes:            <number of inodes>
inodestart:         <first block of the inode region>
datastart:          <first block of the data-block region>
plainsize:          <estimated number of 1 KiB blocks for an uncompressed xv6-style image>
compression ratio:  <plainsize / size>
```

`plainsize` estimates how many 1 KiB blocks the same set of files and directories would occupy if stored uncompressed using the original `xv6` file-system inode indexing scheme. This estimate includes only the inode blocks and data blocks needed for the files and directories contained in the image. It does not include bitmap blocks, log blocks, or extra free blocks. In other words, `plainsize` represents the expected size of a compact, read-only, uncompressed image using the existing `xv6` file-system format. `plainsize` is used as the baseline for computing the compression ratio.

Your `mkcrofs` must be deterministic: building the same image twice should produce equivalent image-size statistics and pass the same tests.


### Part 3. Functional Correctness (20 points)

You need to extend the `xv6` kernel to recognize and mount CROFS. Your CROFS implementation must correctly mount and read a CROFS `fs.img`.
The kernel must:

* Read the CROFS superblock.
* Validate the magic number and basic file-system geometry.
* Locate and read inodes.
* Traverse directories using the original `xv6` directory-entry format.
* Open existing files.
* Read directories.
* Read compressed regular files.
* Support sequential reads.
* Support random reads through `lseek()`.
* Support `exec()` from CROFS regular files.
* Return correct metadata through `fstat()`.
* Correctly handle device files such as `/console`.
* Reject write operations cleanly.
* Avoid reading blocks outside the CROFS image size.

To receive the full credit for this part, your CROFS image must also achieve a minimum compression ratio of:
`plainsize / size >= 1.1`. 

### Part 4. Space-Performance Tradeoff (30 points + bonus 20 points)

In addition to correctness, your CROFS implementation will be evaluated on how effectively it improves both storage space and read performance. This will be based on the combined metric:

```
Rs * Rt
```
where:
```
Rs = plainsize / size
```
and:
```
Rt = uncompressed_read_time / crofs_read_time
```

`Rs` measures space improvement. A larger value means the CROFS image is smaller relative to the estimated compact uncompressed `xv6`-style image.
`Rt` measures read-time improvement. We will evaluate this using 1 KiB random reads across a collection of files. The baseline is the estimated time to read the corresponding uncompressed 1 KiB blocks directly from disk. The CROFS time includes both the time to read compressed or raw blocks from disk and the time spent decompressing data.

To estimate `Rt`, we will measure:
* the amount of data actually read from disk, and
* the cycles spent in decompression.

Disk transfer time will be estimated using an assumed random-read performance of 10,000 IOPS (I/O Operations per Second), which represents UFS-class storage. Decompression time will be estimated from cycle counts, assuming a 2 GHz CPU.
Cycle counts are measured using the RISC-V `rdcycle` instruction inside the kernel. For more consistent measurements, QEMU will be run with `-icount shift=0` option. With this option, QEMU's `rdcycle` values roughly track the number of executed instructions during the measured interval. These numbers are not intended to represent real hardware performance, but they provide a consistent basis for comparing submissions under the same environment.

The following restrictions apply:
* The kernel must not decompress the entire image at boot.
* The kernel must not keep a full uncompressed copy of every file.
* The kernel may not keep hidden per-inode, per-file, per-process, or global decompressed copies.

The skeleton code includes the `bflush()` and `crofs_stats()` system calls to flush the disk buffers and collect various statistics for grading.
The final ranking will be computed only among submissions that satisfy the correctness in Part 3. 
The exact performance scoring thresholds will be calibrated after validating that the metric is stable and meaningful across workloads. 


### Part 5. Design Document (10 points)

Along with your code, submit a design document in a single PDF file. Your document should include the following sections.

1. On-disk layout
   * Explain your CROFS superblock.
   * Explain your inode format.
   * Explain how compressed file data and indexes are represented.
   * Explain how `size` is computed.
2. Compression design
   * State your compression algorithm.
   * State your compression-unit policy.
   * Explain how random reads locate the required compressed region.
   * Explain the trade-off between compression ratio and decompression time.
3. Kernel integration
   * Explain your `lseek()` implementation.
   * Explain how `read()` works for compressed files.
   * Explain how `exec()` works on CROFS.
   * Explain your memory usage.
 4. Testing and validation
   * Outline the test cases you created to validate your implementation, if any.
   * Describe corner cases you considered.
   * Explain which part of the project consumed most of your time and why.

 
## Restrictions

* CROFS is read-only. System calls that create, delete, link, unlink, truncate, or write regular files on CROFS must fail cleanly rather than panic.
* Writes to device files such as `/console` should still work through the normal device driver.
* The shell, `exec()`, directory traversal, and ordinary read-only file operations must work on CROFS.
* Tests that require file-system mutation are not required to succeed on CROFS.
* A data block contains data from only one file; CROFS must not pack data from multiple files into the same block.
* Do not keep the entire decompressed file system, entire decompressed files, or unbounded private decompression caches in memory.
* A random read of a 1 KiB logical block must not require decoding the file from offset 0.
* The kernel must not read CROFS blocks greater than or equal to `size`.
* The image builder must be deterministic and must report accurate image-size statistics.
* Please use `qemu` version 8.2.0 or later. To check your `qemu` version, run: `$ qemu-system-riscv64 --version`
* You only need to change the following files: `./mkfs/mkcrofs.c`, `./kernel/fs.h`, `./kernel/fs.c`, `./kernel/sysfile.c`, `./kernel/crofs.h`, and `./kernel/crofs.c` files. Any other changes will be ignored during grading.

## Skeleton Code

The skeleton code for this project assignment (PA5) is available as a branch named `pa5`. Therefore, you should work on the `pa5` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ cd xv6-riscv-snu
$ git checkout pa5
```
After downloading, set your `STUDENTID` in the `Makefile` again.

In the skeleton code, `xv6`'s original writable file-system layout is replaced with a simpler read-only CROFS layout. The superblock is extended with CROFS metadata such as `CROFS_MAGIC`, and `plainsize`, and the free-block bitmap and log-related fields are removed from the CROFS layout. 
The inode format is kept compatible in size, but the existing `addrs[]` area may be repurposed to store compressed-file indexing metadata. 

The skeleton introduces `crofs.c` and `crofs.h` as the main implementation points for CROFS. You are expected to implement the CROFS read path and decompression logic, including:
```C
crofs_init();
crofs_readi_file();
crofs_decompress();
```

The file-system hooks that connect `xv6` to your implementation are placed in a separate file, `fshook.c`. In particular, `readi_file()` and the wrapper around `crofs_decompress` are defined outside the editable files. You should therefore implement the corresponding CROFS functions in `crofs.c`, but must not modify the hook code in `fshook.c`.

The decompression wrapper, `decompress()`, must be used instead of calling the actual decompression routine, `crofs_decompress()`, directly. The wrapper records various decompression statistics required by the grading script.

The buffer-cache layer is also instrumented for grading. `bread()` records disk-block read statistics, and a `bflush()` system call is added to invalidate cached buffers between benchmark runs. A `crofs_stats()` system call is also added to expose statistics, including the number of disk blocks read, decompression calls, decompressed bytes, and decompression cycles. 

The `mkfs` tool is renamed to `mkcrofs`, and modified to generate a CROFS image. It removes the log and bitmap regions, computes the number of inode blocks dynamically, and records both the compressed image size and the estimated uncompressed size. You should fill in the missing code that compresses file contents and writes the compressed representation into the image. In addition, you must create the `/console` device entry directly in the image instead of relying on `mknod()` at boot time.

## Tips

* Read Chap. 10 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2026-1/book-riscv-rev5.pdf) to understand the file system implementation in `xv6`.

* For your reference, the following roughly shows the required code changes; each `+` denotes about 1~10 lines to add, remove, or modify.
  ```
  mkfs/mkcrofs.c       |  +++++++++++++++++++++++++++++++++++
  kernel/fs.h          |  +
  kernel/fs.c          |  ++++
  kernel/sysfile.c     |  ++++
  kernel/crofs.h       |  +++++++
  kernel/crofs.c       |  +++++++++++++++++++++++++++++++++++++++++++++++
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


