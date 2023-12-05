# 4190.307 Operating Systems (Fall 2023)
# Project #5: Native thread support
### Due: 11:59 PM, December 22 (Friday)

## Introduction

The current version of the `xv6` kernel is designed to support old-fashioned processes only.  This project involves extending the `xv6` kernel to incorporate native thread support. The primary goal is to enable user processes to create and manage user-level threads. Our threading system adheres to the 1:1 threading model. This means that for each user-level thread created, a corresponding kernel thread should be created within the kernel. You must ensure the integration of this new feature does not disrupt the existing process management functionalities of the `xv6` kernel.

## Problem specification

### Part 1: Preparing the `xv6` kernel for native thread support (40 points)

As a first step, you need to separate the notion of "process" and "thread" within the `xv6` kernel. A ***thread*** is a lightweight unit of execution within a process, sharing the process's resources but operating independently in terms of program counter, registers, stack, etc. On the other hand, a ***process*** is a self-contained execution environment, typically comprising its own memory space, program code, and other resources. We can view the "process" in the current `xv6` kernel as the process with only one thread. 

Our goal is to make the process have more than one thread. To achieve this, we must first separate the intertwined structures of processes and threads in `xv6`. Presently, `struct proc` in `xv6` holds all the information pertinent to both a process and its thread. Your task is to isolate the data structures required for each "thread" from those used by the "process". For this purpose, we introduce `struct thread`, a new data structure to store thread-specific information. Meanwhile, any process-wide data will remain in `struct proc`. Since each thread exists within a process, the `struct proc` will include a `struct thread` instance corresponding to every thread that is part of the process.

The code fragment below illustrates the relationship between these data structures. Note that the value of `NTH` is set to 4 in `Makefile`. You can test your code by setting the value of `NTH` to one for Part 1.
```cpp
struct thread {
  int   tid;    // thread id
  int   state;  // thread state
  ...
};

struct proc {
  int   pid;    // process id
  ...

  struct thread thr[NTH];
};
```

Modifying `struct proc` in this way will break the kernel code in a number of places. For example, since a thread is the basic unit of scheduling, the kernel must keep track of the state of each thread and the scheduler is responsible for selecting the next thread to be executed. You must ensure that the `xv6` kernel code functions correctly, even after `struct thread` has been separated from `struct proc`. For this part, you can assume that the number of threads per process (`NTH`) is fixed to one. Since each process has only one (default) thread, any existing program should run correctly on the new `xv6` kernel, including `usertests`. In the second part of this assignment, the value of `NTH` will be set to more than one. Hence, you should design your data structures and modify the kernel accordingly to cope with the scenarios where a process has multiple threads. 

### Part 2: Supporting user-level threads (50 points)

#### 2.1 SNU Threads (sthreads) APIs

In the second part of this project, your task is to enable user-level threads (called sthreads) by implementing the following APIs. Since our sthreads are based on the 1:1 model, each of these will be implemented as a system call. The system call numbers for these functions have been pre-assigned, ranging from 24 to 27 (refer to `./kernel/syscall.h` for details). For Part 2, please change the value of `NTH` to more than one in `Makefile`.

```cpp
int  sthread_self(void);
int  sthread_create(void (*func)(), void *arg);
void sthread_exit(int ret);
int  sthread_join(int tid, int *retp);
```

---
__SYNOPSIS__
```cpp
    int sthread_self(void);
```
__DESCRIPTION__

The `sthread_self()` function returns the ID of the calling thread, which is represented as an integer type.

__RETURN VALUE__

* This function always succeeds, returning the calling thread's ID.

---
__SYNOPSIS__
```cpp
    int sthread_create(void (*func)(), void *arg);
```
__DESCRIPTION__

The `sthread_create()` function creates a new thread within the calling process. The new thread begins execution at `func()`, with `arg` provided as the sole argument to `func()`. 

The thread ID (`t->tid`) of the newly created thread is assigned using the following formula: 
```cpp
struct proc *p = ...;
struct thread *t = ...;

t->tid = p->pid * 100 + n
```
where `p->pid` represents the process ID of the process to which the thread belongs, and `n` is a monotonically increasing number that starts from 0 within that process. For example, in a process with a PID of 3, the *default thread* -- which is the 
 thread initially created during `fork()` -- will be assigned a thread ID of 300. If this default thread subsequently creates another thread using `sthread_create()`, the newly created thread will have a thread ID of 301. 


__RETURN VALUE__

* On success, `sthread_create()` returns the thread ID of the newly created thread.
* On error, `sthread_create()` returns -1.

---
__SYNOPSIS__
```cpp
    void sthread_exit(int retval);
```
__DESCRIPTION__

The `sthread_exit()` function terminates the calling thread and returns a value via `retval` that is available to another thread in the same process that calls `sthread_join()`. If the last thread in a process executes `sthread_exit()`, the associated process should also terminate, freeing up all resources allocated to that process.

__RETURN VALUE__

* This function does not return to the caller.

---
__SYNOPSIS__
```cpp
    int sthread_join(int tid, int *retval);
```
__DESCRIPTION__

The `sthread_join()` function waits for the thread specified by `tid` to terminate. If that thread has already terminated, then `sthread_join()` returns immediately. If `retval` is not NULL (0), then `sthread_join()` copies the exit status of the target thread (i.e., the value that the target thread supplied to `sthread_exit()`) into the location pointed to by `retval`. If multiple threads simultaneously try to join with the same thread, the results are undefined. 

__RETURN VALUE__

* On success, `sthread_join()` returns 0.
* If the target thread has already terminated or the target thread is not found, then `sthread_join()` returns -1.


#### 2.2 Trapframe Handling

The current `xv6` uses a fixed memory region in the user virtual address space to preserve the user context across traps such as system calls and interrupts. However, when a process has multiple threads, it becomes essential to maintain the user context of ***each individual thread*** across traps. Consequently, a significant challenge in implementing multi-thread support in `xv6` lies in ensuring that each thread operates with its own dedicated trapframe. 

We solve this problem by saving the address of the corresponding trapframe to the `sscratch` register whenever a thread returns to the user space. The skeleton code for this project is slightly modified to illustrate this strategy. As you can see below, we have updated the `usertrapret()` (@ `./kernel/trap.c`) function to now pass the trapframe address as a second argument to the `userret()` (@ `./kernel/trampoline.S`) function, utilizing the `a1` register for this purpose. At the very beginning of the `userret()` function, the address is saved to the `sscratch` register.
```cpp
@ ./kernel/trap.c
void usertrapret(void)
{
  ...
  ((void (*)(uint64,uint64))trampoline_userret)(satp, TRAPFRAME);
}
```
```sh
@ ./kernel/trampoline.S
.globl userret
userret:
      # switch to the user page table
      sfence.vma zero, zero
      csrw satp, a0
      sfence.vma zero, zero

      # save the trapframe address (in a1) to sscratch
      csrw sscratch, a1
      mv a0, a1

      # restore all but a0 from TRAPFRAME
      ld ra, 40(a0)
      ld sp, 48(a0)
      ...

      sret
```

When a trap occurs in the user space, the control is transferred to the `uservec()` (@ `./kernel/trampoline.S`) function. Previously, `xv6` has simply initialized the `a0` register with the constant address `TRAPFRAME` after backing up the previous value of the `a0` register to `sscratch`. Now, since the trapframe address for the currently running thread is stored in the `sscratch` register, we need to swap the value of the `sscratch` register and the `a0` register atomically. Fortunately, RISC-V provides the `csrrw` instruction to do this; if we execute the `csrrw a0, sscratch, a0` instruction, the value in `sscratch` is put into `a0`, while the old value of `a0` is stored in `sscratch`, simultaneously. After this instruction, we can freely use the trapframe (pointed to by the `a0` register) to save the user registers used by the current thread. 
```sh
@ ./kernel/trampoline.S
.globl uservec
uservec:
      # swap a0 with sscratch
      csrrw a0, sscratch, a0

      # save the user registers in TRAPFRAME
      sd ra, 40(a0)
      sd sp, 48(a0)
      ...

      # jump to usertrap(), which does not return  
      jt t0
```

Now, all you need to do is to pass the correct trapframe address at the end of the `usertrapret()` function allocated for the thread currently in execution.

#### 2.3 Interactions with Process-oriented System Calls

* `fork()`: If one of the threads invokes `fork()`, only the thread that made the call is duplicated in the new process, thereby becoming the default thread in that process.
 
* `exec()`: If one of the threads executes `exec()`, only the thread that initiated the call will survive, becoming the default thread in that process. This thread starts its execution from the entry point of the new program, while all the other threads in the process are terminated.
 
* `exit()`: If any thread within a process calls `exit()`, all the threads are terminated, and the associated process is subsequently removed from the system. A process can also be terminated when its last thread calls the `sthread_exit()` function. In this scenario, the behavior should be identical to that of the process executing `exit(-1)`.

* `kill()`: If any process is killed by another process via `kill()`, all the threads within the process are terminated, and the associated process is subsequently removed from the system.
 
### 3. Design document (10 points)

You need to prepare and submit the design document (in a single PDF file) for your implementation. Your design document should include the following:

* What information is maintained in the `struct thread` and why?
* Are there any new variables introduced in the `struct proc`? Why?
* Are there any new variables introduced in the `struct thread`? Why?
* How are the trapframes, user stacks, and kernel stacks managed?
* Show the pseudocode for `sthread_create()` and `sthread_exit()`
* Show the pseudocode for `sthread_join()` and how to read the value returned by `sthread_exit()`
* If you modify the existing system calls such as `fork()`, `exec()`, `exit()`, `kill()`, `wait()`, etc., explain why
* What was the hardest part of this project?

## Restrictions

* Your implementation should work on multi-core systems. The `CPUS` variable that represents the number of CPUs in the target QEMU machine emulator is already set to 4 in the `Makefile`. If your implementation works on a single core, but fails on a multi-core system, you will only get half of the points.
* Submitting either the unmodified or slightly modified `xv6` code might allow you to pass the test cases in Part 1 of this project. Any attempt to do so will result in a penalty score of -10 points. Please note that TAs will manually review your code to ensure that your implementation is in line with the intended purpose of this project assignment.
* Do not add any other system calls.
* You only need to modify those files in the `./kernel` directory. Changes to other files will be ignored during grading. 

## Skeleton code

The skeleton code for this project assignment (PA5) is available as a branch named `pa5`. Therefore, you should work on the `pa5` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ cd xv6-riscv-snu
$ git checkout pa5
```
After downloading, you have to set your `STUDENTID` in the `Makefile` again.

The skeleton code includes four user-level programs (`thread1` ~ `thread4`) whose source code is available in `./user/thread1.c` ~ `./user/thread4.c`, respectively. You can use these programs to test your implementation.


## Sample Outputs

The following shows the sample output of each test program provided in the skeleton code.

### 1. `thread1`

The default thread (tid 300) in the `thread1` program creates a thread (tid 301) and performs `sthread_join()` until it finishes its execution. The default thread passes the value `0xdeadbeef` as an argument, while the thread 301 exits with the value `0x900dbeef`.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 1 starting
hart 3 starting
hart 2 starting
init: starting sh
$ thread1
tid 301: got 0xDEADBEEF
tid 300: got 0x900DBEEF
$
```

### 2. `thread2`

The `thread2` program tests a situation where the default thread (tid 300) is terminated before the newly created thread (tid 301). When the thread 301 is terminated, the associated process should be removed from the system. 

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
hart 3 starting
init: starting sh
$ thread2
Thread 300 is exiting
Thread 301 is exiting
$
```

### 3. `thread3`

In `threaad3`, the default thread (tid 300) creates three threads (tid 301, 302, and 303). Each thread is terminated in the order of tid 300 -> 302 -> 303 -> 301.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 3 starting
hart 2 starting
hart 1 starting
init: starting sh
$ thread3
Thread 300 is exiting
Thread 302 is exiting
Thread 303 is exiting
Thread 301 is exiting
$
```

### 4. `thread4`

In `thread4`, the default thread (tid 300) of the process (pid 3) forks a child process (pid 4). The default thread (tid 400) of the child process creates another thread (tid 401), and the thread performs the `exec()` system call to run the `ls /` command. When the `exec()` system call is invoked, the default thread (tid 400) in the child process should be terminated. The parent process (pid 3) waits until the child process is terminated, receiving its exit status in the `ret` variable. This will be the return value of running the `ls /` command, which is always zero. 

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
hart 3 starting
init: starting sh
$ thread4
.              1 1 1024
..             1 1 1024
README         2 2 2305
cat            2 3 33256
echo           2 4 32136
forktest       2 5 16264
grep           2 6 36696
init           2 7 32608
kill           2 8 32056
ln             2 9 31880
ls             2 10 35200
mkdir          2 11 32120
rm             2 12 32104
sh             2 13 54688
stressfs       2 14 33000
usertests      2 15 180744
grind          2 16 47864
wc             2 17 34184
zombie         2 18 31472
thread1        2 19 32712
thread2        2 20 32352
thread3        2 21 32528
thread4        2 22 32552
console        3 23 0
ret = 0
$
```

## Tips

* Read Chap. 2, 3, 4, 6, and 7 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2023-2/book-riscv-rev3.pdf) to understand process management and trap handling in `xv6`. 

## Hand in instructions

* First, make sure you are on the `pa5` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. Upload this file to the submission server.
* Please remove all the debugging outputs before you submit.
* The total number of submissions for this project assignment will be limited to 30. Only the version marked `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Note that adding your new IP address is a ___manual___ process, so please do not expect it to be completed immediately.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delay.
* __You can use up to _3 slip days_ during this semester__. If your submission is delayed by one day and you decide to use one slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use in the QnA board of the submission server right after each submission. Once slip days have been used, they cannot be canceled later, so saving them for later projects is highly recommended!
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)



<!--
### Multithreads-aware `xv6`

* Alloc & free thread structs
* Preallocated kernel stacks in kernel virtual address space (per-proc -> per-thread)
* Map kernel stack for the default thread
* userinit()
* fork(), exec(), wait(), exit(), kill(): need to be changed
* Default thread is not necessarily the thread 0. (thread 2 can invoke exec())
* schedule(): thread is a unit of scheduling

### Multiple threads support (up to NTH)

* int thr_create(void (*func)(), void *arg)  
  On success, returns thread ID (tid), Otherwise returns -1
* void thr_exit(int retval) 
* in thr_join(int tid, int *retval) 
  On success, returns 0. Otherwise returns -1

* How can each thread have its own trapframe?
* How to find it on trap?
* How to use more than one register to do that? (problem: we only have one spare register (a0) after swapping with scratch)
-->
