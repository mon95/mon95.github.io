---
layout: post
title: How to build and install the latest Linux kernel from source
---
![A map of the Linux kernel](/images/bp1pic1.gif)

I just finished my first assignment for a course on Advanced Operating Systems. And I decided to document my approach for building the Linux kernel from source and implementing my own system call.

There are a number of blogs that already tell you how to go about doing this, but some of them are outdated, and some seem unnecessarily complicated. My goal is to present a straightforward approach for doing this, which should hopefully help you save a lot of time.

Compiling the Linux Kernel from source can seem like a daunting task, even to someone who’s pretty comfortable with computers in general. It can also get really irritating if you aren’t following the right instructions.

So, here’s a guide to help you through the process of building the kernel from source, and it’s a guide that works! You will not have to worry about messing up your system or wasting your time.

### Why build the kernel from source?
If you plan to work on the internals of the Linux kernel or change its behavior, you’ll need to recompile the kernel on your system.

Here are a few specific cases where you’ll need to know how to work with the kernel’s source code:

1. You want to write a really cool ‘Hello world’ program. (Each time you [implement your own system call](https://mon95.github.io/Implementing-a-system-call-in-linux-kernel-4-7-1/) or modify kernel source code, you will need to recompile the kernel to implement the changes)

2. You want to enable experimental features on your kernel that are not enabled by default (or, disable default features that you don’t want)

3. You want to debug kernel source code, enable support for a new piece of hardware, or make modifications to its existing configurations

4. You’re doing a course on Advanced Operating Systems and have no choice but to do this!

In each of the above situations, learning how to build the kernel from source will come in handy.

### What you’ll need
A Linux based Operating System (I tried this on Ubuntu 14.04 LTS and the instructions written here are for the same).

You will need to install a few packages before you can get started. Use the following commands for the same.

```
sudo apt-get update

sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc
```

You will also need up to at least 12 GB of free space on disk, an internet connection to download the source code, and a lot of time (about 45 to 90 minutes).

### Downloading and extracting the latest kernel source
To check your current kernel version, open the terminal and type:

```
uname -r
```

Go to kernel.org and download the latest stable version. At the time of writing this, the latest stable kernel version was 4.7.1, and I will refer to the same in this article. (Note: Try to avoid downloading source from other websites)

Change to the directory where the file was downloaded and extract using:
  
```
tar xf linux-4.7.1.tar.xz
```

Change to the extracted linux-4.7.1 directory.

```
cd linux-4.7.1
```

It should contain folders called arch, fs, crypto, etc.

### Configuring and Compiling:
Before compiling the kernel, we need to configure which modules are to be included and which ones are to be left out.

There are many ways to go about doing this.

An easy and straightforward way to do this is to first copy your existing kernel config file and then use ‘menuconfig’ to make changes (if necessary). This is the fastest way to do it and probably, the safest.

```
cp /boot/config-$(uname -r) .config   

make menuconfig
```

This is the part where you could end up removing support for a device driver or do something of the sort which will eventually result in a broken kernel. If you are unsure about making changes, just save and exit.

Note: One of the alternatives to menuconfig is an interactive command line interface accessible using ‘make config’. This helps you configure everything from scratch. <b>Do not use this.</b> You will be asked over a thousand yes/no questions about enabling or disabling modules, which I promise is no fun whatsoever. I did try this out once and somehow managed to mess up the display driver configurations.

gconfig and xconfig are alternate GUI based configuration tools that you could use. I haven’t tried these myself. For this, you’ll need to use make gconfig (or make xconfig) instead of make menuconfig.

Now, we’re all set!

To compile the kernel and its modules, we use the <b>make</b> command.

This is followed by using <b>make modules_install</b> to install the kernel modules.

Finally, we use <b>make install</b> to copy the kernel and the .config file to the /boot folder and to generate the system.map file (which is a symbol table used by the kernel).

These three steps put together usually take up a lot of time. Use the following command to perform the above tasks:

```
sudo make -j 4 && sudo make modules_install -j 4 && sudo make install -j 4
```

Note: I have used the -j option to specify the number of cores to be used. This tends to speed up the process considerably. You can use nproc to check the number of processing units available. In my case, it was 4 cores.

Ideally, you shouldn’t need sudo privileges, but, I was running into problems when I didn’t run it with sudo privileges.

### Final steps
Once the kernel and its modules are compiled and installed, we want to be using the new kernel the next time we boot up.

For this to happen, we need to use the following command:

```
update-initramfs -c -k 4.7.1   
```

Then, use the following command, which automatically looks for the kernels present in the /boot folder and adds them to the grub’s config file.

```
update-grub  
```

Now, restart the system and you should see that the new kernel is added to the boot loader entries.

On following the instructions, assuming there’s enough space available on disk and the current kernel configuration works fine, you shouldn’t encounter any problems. Note that you could always use the old kernel version in case of any problem and try the whole thing again!

The command uname -r should now show you the current kernel version being used.

### An important note
The above steps are needed to build the kernel from source, for the first time. Once, this is done at least once and a new kernel image is ready, making changes and writing our own modules is simple. You will only be using the steps listed under <b>Configuring and Compiling</b> each time something new is to be implemented or configured differently.

Meaning, just remember the following:

```
cp /boot/config-$(uname -r) .config

make menuconfig

sudo make -j 4 && sudo make modules_install -j 4 && sudo make install -j 4
```

I must give credit to the following worthwhile resources — they were hugely helpful with this task: Ramkitech.com, askubuntu.com, kernel.org and cyberciti.biz
