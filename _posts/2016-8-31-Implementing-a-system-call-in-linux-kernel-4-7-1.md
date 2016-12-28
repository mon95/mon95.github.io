---
layout: post
title: Implementing a system call in Linux Kernel 4.7.1
---
![system call implementation](https://cdn-images-1.medium.com/max/800/1*4Tc89BJaeY-jAunLaeT9MA.png)

In the previous post, I had written about compiling and installing the Linux kernel from source. Now, let’s see how we can implement a non trivial system call on the newly installed kernel.

I decided to go with a non-trivial system call (as opposed to implementing a ‘hello world’ system call) as this could help you get started with reading Linux Source Code and understanding various headers, functions and data structures that are used internally.

<i>(Note: All of this could have been done in conjunction with what was talked about in the previous post. I’ve only put them in separate articles to avoid any sort of confusion. This article is about implementing your own system call, recompiling your kernel and seeing your system call in action)</i>i>

So, I’m going to walk you through the process of implementing a system call which iterates over all processes and prints its details that can be accessed via. the underlying ‘task_struct’ data structure, to the kernel log.

### Defining our system call:
Ensure that you are in the linux-4.7.1 directory.

Create a new directory, say ‘info’ and change to this directory. We’ll maintain the necessary header file(s) and implementation file(s) for the system call in this directory.

```
mkdir info
cd info/ 
```

Create a header file ‘processInfo.h’ that will contain the necessary function declarations, structure declarations, macros, etc. For this example, we will only be using it to declare the function prototype.

Include the following line in the header file. Here, [asmlinkage tells the compiler to look at the CPU’s stack for the function parameters, and, long is generally used as a return type in kernel space](https://www.quora.com/Linux-Kernel-What-does-asmlinkage-mean-in-the-definition-of-system-calls) for functions that return an int in user space.

```
asmlinkage long sys_listProcessInfo(void);
```

Now, let’s define our system call in ‘listProcessInfo.c’.

```
#include<linux/kernel.h>
#include<linux/init.h>
#include<linux/sched.h>
#include<linux/syscalls.h>
#include "processInfo.h"
asmlinkage long sys_listProcessInfo(void) {
    struct task_struct *proces;
 
    for_each_process(proces) {
 
    printk(
      "Process: %s\n \
       PID_Number: %ld\n \
       Process State: %ld\n \
       Priority: %ld\n \
       RT_Priority: %ld\n \
       Static Priority: %ld\n \
       Normal Priority: %ld\n", \
       proces->comm, \
       (long)task_pid_nr(proces), \
       (long)proces->state, \
       (long)proces->prio, \
       (long)proces->rt_priority, \
       (long)proces->static_prio, \
       (long)proces->normal_prio \
    );
  
  
   if(proces->parent) 
      printk(
        "Parent process: %s, \
         PID_Number: %ld", \ 
         proces->parent->comm, \
         (long)task_pid_nr(proces->parent) \
      );
  
   printk("\n\n");
  
  }
  
  return 0;
}
```

<i>(I used Linux Data Structures to get an idea of the kind of data structures being used. Note that this is based on an old 2.0.33 version. So, some of the fields have changed since then. For example, the ‘parent’ field (in the above code) was called ‘p_pptr’ in the older kernels. You’ll have to look into current source code to figure out the differences)</i>

Write a Makefile in the same directory(i.e., info/) with the following contents:

```
obj-y:=listProcessInfo.o
```

This is to ensure that the listProcessInfo.c file is compiled and included in the kernel source code.

Now, we’re all set to link our system call implementation with the existing kernel!

### Modifying necessary kernel files to integrate our system call:
Add the new ‘info’ directory to the kernel’s Makefile.

```
For this, open the kernel's Makefile (found in the linux-4.7.1 directory) and look for the following line:

core -y  += kernel/ mm/ fs/ ipc/ security/ crypto/ block/ 

(highlighted in the screenshot attached below) And, change it to include info/. 

It should read:
core -y  += kernel/ mm/ fs/ ipc/ security/ crypto/ block/ info/
```

![screenshot1](https://cdn-images-1.medium.com/max/800/1*pmxmVW9GyOkb4wEURsR-1g.png)

This tells the compiler that the source files of our new system call can be found in the info/ directory.

Now, we’ll have to alter the syscall_64.tbl.

A neat way to figure out where this file is present is to use the ‘find’ command on the terminal from the linux-4.7.1 directory.

```
find -name syscall_64.tbl   ### Should show the file's location
```

<i>(This is equivalent to using the find option(ctrl+F) in nautilus to look for where a specific file is located)</i>

In kernel 4.7.1, it is present in /arch/x86/entry/syscalls/syscall_64.tbl.

Now, edit the file as shown to include the new system call number and its entry point. Just note the system call number for reference. <i>(Ideally, we should be implementing a wrapper for our system call and will never be using the number directly. But, in this example, I’m going to use the system call number to test the system call)</i>

![screenshot2](https://cdn-images-1.medium.com/max/800/1*4Tc89BJaeY-jAunLaeT9MA.png)

```
318 common getrandom  sys_getrandom
319 common memfd_create  sys_memfd_create
320 common kexec_file_load  sys_kexec_file_load
321 common bpf   sys_bpf
322 64 execveat  stub_execveat
< Make your edit here >
#
# x32-specific system call numbers start at 512 to avoid cache impact
# for native 64-bit operation.
#
```

Finally, we’ll have to alter the syscalls.h file. Again, use ‘find’ look for where the syscalls.h file is present.

```
find -name syscalls.h
```

In kernel 4.7.1, it is found in /include/linux/.

Add the following line to the end of the file (before the #endif) as shown:

```
asmlinkage long sys_hello(void)
```

![screenshot3](https://cdn-images-1.medium.com/max/800/1*4AW2BLjsfXjAb5o2feF-CA.png)

And, we’re done!

### Recompile and Reboot:
To integrate the system call and to be able to actually use it, we’ll need to recompile the kernel as was outlined at the end of the previous post (see, ‘Important Note’).

For the sake of completeness, I have included the same here.

```
sudo make -j 4 && sudo make modules_install -j 4 && sudo make install -j 4
```

Once this is done, restart the system.

### Testing the system call:
To test the system call write a simple ‘test.c’ function (it can be placed in any directory) as follows:

```
#include <stdio.h>
#include <linux/kernel.h>
#include <sys/syscall.h>
#include <unistd.h>
int main()
{  
    printf("Invoking 'listProcessInfo' system call");
         
    long int ret_status = syscall(323); // 323 is the syscall number
         
    if(ret_status == 0) 
         printf("System call 'listProcessInfo' executed correctly. Use dmesg to check processInfo\n");
    
    else 
         printf("System call 'listProcessInfo' did not execute as expected\n");
          
     return 0;
}
```

Compile and execute this program. If it runs successfully, then, it should give the corresponding prompt and you can now use ‘dmesg’ to check the kernel log and actually verify if the process information has been logged.


```
dmesg   ### Check the kernel log to which we print the process info
```

And with that, we have successfully implemented a working system call that actually uses one of the internal kernel data structures!

<i>Note: The system call implemented does not take care of privilege checks, does not return any error codes on failure and does not do anything particularly useful for the user. So, it is actually far from being a well-designed system call! But, …</i>

The idea was to implement a system call in the new kernel, learn how to modify the various kernel files, quickly integrate the system call and check to see if it works.

Now, when working on the actual heavyweight stuff (designing and coming up with a clean and game-changing system call), you won’t need to worry about going wrong with the easier parts!