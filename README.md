# 4190.307 Operating Systems (Fall 2024)
# Project #5: FullFS: A File System with Full-Path Indexing
### Due: 11:59 PM, December 22 (Sunday)

## Introduction

Full-path indexing is a file system technique where directories store the complete path and associated inode information for each file or subdirectory. The goal of this project is to explore the implementation of a full-path indexing mechanism in the `xv6` file system, focusing on its impact on directory structure management and pathname resolution efficiency.

## Background

### Traditional Pathname Lookup

In traditional UNIX operating systems, pathname lookup follows a hierarchical process to resolve a file or directory path. The kernel starts at the root directory (`"/"`) and traverses each component of the given pathname sequentially. For absolute paths, the traversal begins at the root, while for relative paths, it starts at the current working directory. At each step, the system accesses the inode of the current directory, searches its directory entries for the next component, and retrieves the corresponding inode. This process continues recursively until the final component is resolved. While this approach is straightforward and aligns with the hierarchical structure of UNIX file systems, it can be inefficient for deep directory trees or repeated lookups of the same path, as it requires multiple disk accesses and inode reads. 

### Full-path Indexing

Full-path indexing is a file system technique where directories maintain the full path and corresponding inode information for each file or subdirectory. Instead of traversing the file hierarchy recursively to resolve each component of a pathname, this approach allows direct lookup of any file or directory using its full path. By embedding the entire path, the file system eliminates the need for iterative searches through intermediate directory layers, significantly reducing the overhead of path resolution in deep or complex directory structures.

This method improves performance for file lookups and operations that frequently access nested files. However, full-path indexing can increase storage overhead within directory entries and necessitates more complex handling of path changes, such as when files are moved or renamed. Despite these trade-offs, full-path indexing is particularly useful in applications where fast and efficient file resolution outweighs the cost of additional storage or implementation complexity.
For more detailed information about full-path indexing, please refer to this [paper](https://www.usenix.org/conference/fast18/presentation/zhan).

## FullFS Design

### Directory Structure

In contrast to traditional UNIX file systems, where each directory maintains its own directory file, FullFS employs a simplified structure with a single global directory named `"/"`. All files and subdirectories in the system are stored within this single root directory, eliminating the hierarchical structure commonly found in conventional file systems. 

The root directory in FullFS is implemented as a centralized collection of directory entries, as outlined below. Each entry represents a file or a subdirectory, with the `name` field storing its full path and the `inum` field corresponding to the associated inode number. The maximum length for a pathname, including the trailing null character (`'\0'`), is limited to `MAXPATH` (106 bytes). Each directory entry is fixed at 128 bytes in size. The remaining space in the entry, labeled as `padding`, is reserved for optional use and can be customized to fit the requirements of your implementation. Note that the `padding` field can be used as desired, but the `inum` and `name` fields are fixed: the first 2 bytes are reserved for `inum`, and the last `MAXPATH` bytes are reserved for the full pathname. The inode number for the root directory is fixed at a constant value of 1, which is defined as `ROOTINO` in the `./kernel/fs.h` file.

```C
// @ kernel/param.h
#define MAXPATH   106

// @ kernel/fs.h
struct dirent {
  ushort inum;          // inode number
  char padding[20];     // reserved space for custom implementation
  char name[MAXPATH];   // full pathname
};
```   

Here’s how the root directory would look in FullFS for the given example with three directories (`/bin`, `/home`, and `/home/jinsoo`) and two files (`/README` and `/bin/ls`):

  | inode (`inum`) | pathname (`name`)    |
  |:---------------|:---------------------|
  | 1              | /                    |
  | 2              | /bin                 |
  | 3              | /home                |
  | 4              | /bin/ls              |
  | 5              | /home/jinsoo         |
  | 6              | /README              |

### Subdirectories

In FullFS, although subdirectories do not maintain their own directory entries or allocated data blocks, they still require corresponding entries in the root directory. 
This design ensures that operations such as file/directory creation or traversal are valid only when the parent directory exists.
For example, a file creation request for `/bin/ls` can proceed only if the root directory includes an entry for `/bin`. These entries serve as placeholders for subdirectories and enforce hierarchical path constraints. 

Unlike traditional file systems, where subdirectories include `.` and `..` entries to facilitate easy lookup of the current and parent directories, FullFS omits these entries entirely. Consequently, every directory in FullFS has a link count of 1, reflecting its single entry in the root directory. Note that `xv6` does not allow hard links to directories.

Despite this simplification, FullFS must dynamically parse and resolve each component of a given path to maintain backward compatibility with expected file system behaviors. For instance, complex pathnames such as `/home/jinsoo/../.././bin/ls` must be correctly interpreted by resolving all current (`.`) and parent (`..`) directory components in the context of the file system.
Similarly, FullFS must support shell commands like `$ cd ..` and `$ ../cat ../README`, which rely on the correct interpretation of relative paths, even in the absence of `.` and `..` directory entries.

## Problem Specification

### Part 1. Modifying the `mkfs` utility for FullFS (10 points)

Your first task is to modify the command-line utility `mkfs` to support the initialization of the FullFS file system.
The `mkfs` utility is responsible for creating and initializing the file system on a storage device or disk image. It sets up essential file system structures, including the boot block, superblock, inode blocks, data blocks, and bitmap blocks for tracking inode and data block allocation. 
Additionally, `mkfs` populates the file system with necessary directories (including the root directory) and system files, ensuring that the file system is ready for use upon mounting. 

Specifically, you need to modify the root directory to store all entries in a single, centralized structure, as required by FullFS. 
Furthermore, traditional directory entries such as `.` and `..` must be excluded from the root directory.

### Part 2. Implementing Full-Path Indexing (60 points)

Your second task is to implement FullFS, a modified version of the `xv6` file system that supports the full-path indexing mechanism.
This involves restructuring the file system so that all system calls operate correctly with the new directory structure.
Each entry in the root directory should contain the full path of the file or subdirectory, adhering to the `MAXPATH` limit of 106 bytes (including the null terminator). 
To complete this task, you need to implement path resolution logic capable of handling both absolute paths (e.g., `/bin/ls`) and relative paths (e.g., `../bin/ls`), even though traditional `.` and `..` entries are not explicitly stored in the directory.

In order to parse relative paths correctly within system call handlers, each process should maintain information about its current working directory. This directory is initially inherited from the parent process during `fork()` and can be updated later using the `chdir()` system call. 

As part of this task, you are required to implement a new system call, `pwd()`, which returns the absolute pathname of the current working directory for the calling process. This pathname is crucial for both system calls and shell commands, as it provides the necessary context for dynamically resolving relative paths. The system call number of `pwd()` has already been assigned as 23 in the `./kernel/syscall.h` file.

__SYNOPSYS__
```
  int pwd(char *buf);
```

__DESCRIPTION__

  This system call returns a null-terminated string containing the absolute pathname of the current working directory for the calling process. 
  The pathname is stored in the buffer provided by the argument `buf`, which must be able to hold a maximum of `MAXPATH` bytes, including the null terminator.

__RETURN VALUE__

* On success, returns 0.
* If the argument `buf` is NULL or points to an invalid address, returns -1.

Your implementation must ensure that file system operations, such as file creation, deletion, and traversal, work seamlessly on multi-core systems with the new directory structure introduced by FullFS. The correctness of your implementation will be validated by passing the `usertests` suite with the `-q` option. Note that some irrelevant test cases in the `usertests` suite have been disabled in the provided skeleton code.

### Part 3. Modifying the `ls` command for FullFS (20 points)

This part involves modifying the `ls` command to ensure it exhibits the same behavior as it would in a traditional file system, even though FullFS has a single, centralized directory structure. 
Specifically, the `ls` command must correctly interpret and display the contents of directories as if they existed hierarchically. For example, executing `$ ls /home` should display the entries for the files and directories within the `/home` directory. This requires parsing the full-path entries in the root directory to identify entries that logically belong to the `/home` directory. 

Note that we have slightly modified the output format of the `ls` command so that the pathname is located at the end of the output. Additionally, you are required to implement the `$ ls //` command, which should display all valid entries stored in the root directory (refer to [Examples](#Examples) for details).

### Part 4. Design Document (10 points)

Prepare a design document detailing your implementation in a single PDF file. Your document should include the following sections:

1. New data structures
   * Provide details about any newly introduced data structures or modifications made to existing ones.
   * Explain why these data structures/modifications were necessary and how they contribute to the implementation.
2. Algorithm design
   * Provide an overview of the overall flow of the directory lookup process.
   * Describe any corner cases you considered and the strategies you used to address them.
   * Discuss any optimizations you applied to improve code efficiency, both in terms of time and space.
3. Testing and validation
   * Outline the test cases you created to validate your implementation, if any.
   * Describe how you verified the correct handling of the corner cases mentioned in Section 2.

### **Bonus** (up to an additional 20 points)

As a bonus, we will evaluate the average execution time of the `open()` system call in a large directory. The performance will be measured on a single processor system using the `rdtime()` system call with the QEMU option `-icount shift=0`. The top five fastest implementations will receive an additional 20 bonus points, while the next five will earn 10 bonus points. You may use hashing to improve directory lookup performance; however, at least one byte-to-byte comparison of pathnames is necessary to confirm results and avoid false positives caused by hash collisions. Any submissions that omit this comparison will not be eligible for bonus points. 


## Restrictions

* Please use `qemu` version 8.2.0 or later. To check your `qemu` version, run: `$ qemu-system-riscv64 --version`

## Skeleton Code

The skeleton code for this project assignment (PA5) is available as a branch named `pa5`. Therefore, you should work on the `pa5` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ git checkout pa5
```
After downloading, you must first set your `STUDENTID` in the `Makefile` again.

The `pa5` branch includes two user-level program, `pwd` and `fsperf`, with their source code located in `./user/pwd.c` and `./user/fsperf.c`, respectively. 
The `pwd` program simply executes the `pwd()` system call and prints the resulting absolute pathname of the current working directory. 
The `fsperf` program is designed to evaluate the file system performance. It creates 150 files in the `/fstests/snu/operating/systems/2024/fall/project5` directory, measures the time taken to perform a series of `open()` system calls on these files, and cleans up the test files. Please note that a program different from `fsperf` will be used to measure the performance of the `open()` system call for the bonus points.


## Examples

The following demonstrates the output of the `ls` command when the system is booted. Note that the `ls` command's output format has been slightly modified. The format is structured as follows:
```
[type] [inum] [link] [size] [name]
```
 * `type`: indicates whether the entry is a directory (1), a file (2), or a device (3).
 * `inum`: represents the inode number associated with the file or directory.
 * `link`: displays the link count of the file or directory.
 * `size`: shows the size of the file or directory in bytes.
 * `name`: provides the name of the file or directory.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -nographic  -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -smp 3

xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ ls                                <-- list files in the current directory 
2 2 1 2403 README
2 3 1 35264 cat
2 4 1 34144 echo
2 5 1 16336 forktest
2 6 1 38696 grep
2 7 1 34616 init
2 8 1 34080 kill
2 9 1 33904 ln
2 10 1 49552 ls
2 11 1 34144 mkdir
2 12 1 34128 rm
2 13 1 56712 sh
2 14 1 35008 stressfs
2 15 1 178336 usertests
2 16 1 49928 grind
2 17 1 36208 wc
2 18 1 33496 zombie
2 19 1 34592 pwd
2 20 1 41344 fsperf
3 21 1 0 console
$ pwd                               <-- show the current directory 
/
$ ls README                         <-- list a file `README`
2 2 1 2403 README
$ mkdir home                        <- make a new directory /home
$ mkdir home/jinsoo                 <- make a new directory /home/jinsoo
$ cat README > home/jinsoo/README2  <- copy a file to /home/jinsoo
$ cd home                           <- change the current directory to /home
$ ls                                <- ls is not in the current directory
exec ls failed
$ ../ls                             <- list files in the current directory
1 23 1 0 jinsoo
$ cd jinsoo                         <- change the current directory to /home/jinsoo
$ ../../pwd                         <- show the current directory
/home/jinsoo
$ /ls                               <- list files in the current directory
2 24 1 2403 README2
$ cd ..                             <- change to the parent directory
$ /ls                               <- list files in the current directory
1 23 1 0 jinsoo
$ cd ..                             <- change to the parent directory
$ ls //                             <- show all the files/directories
1 1 1 3072 /
2 2 1 2403 /README
2 3 1 35264 /cat
2 4 1 34144 /echo
2 5 1 16336 /forktest
2 6 1 38696 /grep
2 7 1 34616 /init
2 8 1 34080 /kill
2 9 1 33904 /ln
2 10 1 49552 /ls
2 11 1 34144 /mkdir
2 12 1 34128 /rm
2 13 1 56712 /sh
2 14 1 35008 /stressfs
2 15 1 178336 /usertests
2 16 1 49928 /grind
2 17 1 36208 /wc
2 18 1 33496 /zombie
2 19 1 34592 /pwd
2 20 1 41344 /fsperf
3 21 1 0 /console
1 22 1 0 /home
1 23 1 0 /home/jinsoo
2 24 1 2403 /home/jinsoo/README2
```

Below is an example execution of the `fsperf` program. Note that running the `$ make perf` command sets `CPUS` to 1 and adds the QEMU option `-icount shift=0`.

```
$ make perf
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -nographic  -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -smp 1 -icount shift=0

xv6 kernel is booting

init: starting sh
$ fsperf
Usage: fsperf [[c]reate|[o]pen|[d]elete]
$ fsperf c
creating files...
$ fsperf o
opening files...
open: 34567 cycles/file
$ fsperf d
deleting files...
$
```

## Tips

* Read Chap. 8 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2024-2/book-riscv-rev4.pdf) to understand the file system in `xv6`.
  
* For your reference, the following roughly shows the amount of changes you need to make for this project assignment. Each `+` symbol indicates 1~10 lines of code that should be added, deleted, or altered.
   ```
   kernel/defs.h        |  +
   kernel/exec.c        |  +
   kernel/fs.h          |  +
   kernel/fs.c          |  ++++++
   kernel/proc.c        |  +
   kernel/sysfile.c     |  +++++++++++++
   mkfs/mkfs.c          |  ++++
   user/ls.c            |  +++++++++++++++
   ```

## Hand in instructions

* First, make sure you are on the `pa5` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. Upload this file to the submission server. Additionally, your design document should be uploaded as the report for this project assignment.

* The total number of submissions for this project assignment will be limited to 50. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button.

* The `sys` server will no longer accept submissions after three days (72 hours) past the deadline.
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Now, adding your new IP address is automated by a script that periodically checks the Google Form at minutes 0, 20, and 40 during the hours between 09:00 and 00:40 the following day, and at minute 0 every hour between 01:00 and 09:00.
     + If you cannot reach the server a minute after the update time, check your IP address, as you might have sent the wrong IP address.
     + If you still cannot access the server after a while, it is likely due to an error in the automated process. The TAs will check if the script is running properly, but since that is a ___manual___ process, please do not expect it to be completed immediately.

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


