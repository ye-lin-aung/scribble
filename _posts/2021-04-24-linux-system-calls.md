---
title: Linux System Calls
date: '2021-04-24 21:42:10'
---

# What are system calls 
A systemcall is init location for karnel. A process can request kernel to perform kernel level actions by using system calls. Its like API for karnel. These APIs are basicing about scheduling processes, creating new processes, performing IO, piping processes and terminating processes. A system call changes the process state from user mode to kernel mode allowing CPU to able to interacted protected resources within kernel. Every system calls are fixed along kernel. Each call have their own unique number.

### Example sys calls table 

![System call table](/images/sys_call.png)


Don't worry, you don't need to remember all those numbers. You can just call by their function names like `open` or `fork` from your C program. The system calls tables are different based on their architecture.

## Things to know before further reading 

### Operating system and Kernel 

The major difference between OS and kernel is that kernel is the core of an OS. It manages resources such as process scheduling, memory management, file systems, process management, devices management and networking. It acts like the bridge between software and hardware. Operating system communicate the Kernel via system calls.  OS is another abstraction layer upon kernel. OS never directly interacts with hardware. It only request kernel via system calls to interact with hardware.

### Kernel mode and user mode 

Many processor architectures includes kernel mode and user mode. These are hardware instructions that can switch back and forward between one mode to other. Areas of virtual memory can be marked as being part of user space or kernel space. CPU can only access memory that is in user space in user mode. But in kernel mode, CPU can acess both user and kernel memory space. Some operations like memory management, device IO operations can be done only in kernel mode. By having different modes and different spaces. Operating system can ensure user processes are not able to access instructions and memory that are not belong to their user space which can give abnormalities in system. If a process from user mode needs to request for new process or interact with IO operations, a process must call system call which will switch into a kernel mode. 

### Traps and interrupts 

We can think traps and interrupts like signals.  Traps are fired by processes, whenever a trap is invoked, the trap handler process in kernel will switch the process from user mode to kernel mode and call system call corresponding to trap vector. Interrupts on the other hand are signal by Kernel. Interrupts are normally occured with timers. When ever the interrupt happens the current process will pause and the CPU will be scheduled for another process. You can read more about CPU scheduling in OS. Interrupts are mostly used in CPU scheduling in kernel. Traps and interrupts are initialized while the system is booted. It registers trap/interrupt table and listeners while the system is booted. Timer issues interrupts in short time manners up to 100usec. A timer device can be programmed to raise an interrupt every so many milliseconds; when the interrupt is raised, the currently running process is halted, and a pre-configured interrupt handler in the OS runs.


## Behind the scence 

Lets take a look at what happens if a C program calls a system call in x86-32 machine 

1. Application program call the function. (example `open`) 
2. The application pass the arguments of the function via a stack. 
3. System calls only read data from specific registers not from stack. The called function will pass the arguments to specific registers. 
4. The called function will copy the system call number ( lets call this A ) into the register. (%eax) 
5. The called function will trigger the trap  (int 0x80), which causes the processor to move from user mode to kernel mode and execute the code point located by location 0x80 of system trap vector. 

`int` here refers to `interrupt` not `integer` as in programming languages.  0x80 means `signal` which triggers signal to trap handler to switch between user mode to kernel mode. Find 0x80 under x86-32bit [here](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/errnos.md) .

6. The trap handler will invoke `system_call()` routine. 
7. Kernel will find system call (A) from %eax and get index from sys call table checks the validity of the call. Things like valid memory pointers of arguments and etc are checked here. 
8. Kernel will perform the system call like moving values at addresses, transfering data between user memory and kernel memory etc. 
9. Restore the register value from kernel and places return value from the system call (A) onto the stack.
10. Returns the function, returning to user mode. 
11. If the return value of the system call service routine indicated an error, the wrapper function sets the global variable errno using this value. The wrapper function then returns to the caller, providing an integer return value indicating the success or failure of the system call. You can use this `errno` to indicate what kind of error happened during system call. The errno returned is integer value. The returned integer can be matched with the error table to get more details about error.
 
![Behind the scence](/images/excution_of_system_call.PNG)

### Example errorno index table from linux

![System call table](/images/errorno.PNG)
