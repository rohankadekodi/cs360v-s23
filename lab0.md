# Project 1 Lab 0

# Part 1: Introduction

The goal of this lab is to familiarize you QEMU and KVM (two popular pieces of virtualization software). For the rest of Project 1, you will build part of a paravirtual hypervisor, which will run in a QEMU/KVM virtual machine. You may also finding the tracing tools and VM setup instructions presented in this lab useful in Project 2. 

You will complete this and the other Project 1 labs in a group of 2-4 students. Please sign up for an existing Project 1 group on the "People" tab on Canvas. You will complete all Project 1 labs with the same group. You will submit the assignment through GitHub classroom (see Canvas for the invite link).

**Due date: January 24, 2023**

## Lab machine access

For this lab, we will use the department's public lab machines. The lab requires KVM, a Linux kernel module that accelerates virtualization, which is only set up on the following department machines:

- key
- cyan
- magenta
- yellow

Note that these machines are set up for remote access only. See [the department documenation](https://www.cs.utexas.edu/facilities-documentation/ssh-keys-cs-mac-and-linux) for help setting up remote access to these machines.

**IMPORTANT**: If this lab will cause you to exceed your storage quota (10GB for undergrads), you may store files in /var/local. Please read /var/local/README before using this space and do not store anything private or important in this folder.

You may also use a personal computer for this lab. You will need a machine running a Linux distribution with QEMU version >= 2.11 and the KVM module loaded. Course staff will not be able to assist with environments other than the department lab machines.

Note that for subsequent labs, we will use different servers that are set up for this course. You will receive instructions on how to access those servers when we begin Lab 1.

# Part 1.5: Prepping for Lab 1
You will work on subsequent labs on a separate set of servers set up for this course. You need to provide some information to course staff to gain access to these servers. **Please do this as early as possible - you will not be able to start Lab 1 until you provide this information.** 

1. On your own computer, run `ssh-keygen -t ecdsa`. Follow the prompts to create public and private SSH keys. Be sure to password-protect your keys.
2. Log into [stache.utexas.edu](stache.utexas.edu) using your UT EID. Stache is a website provided by UT to safely share private information like SSH keys. See https://sites.cns.utexas.edu/oit-blog/blog/what-stache-and-how-use-it-share-confidential-information for more information.
3. Create a new entry and copy the public key you just created (it will have the suffix `.pub`) into the "Secret" field. 
4. Share your key with course staff using the "Add a person" search bar on the right side of the window. 

**Note: each student should create their own SSH key**. 

# Part 3: Setting up a VM

The first part of this lab involves setting up a VM for use with QEMU, a popular hypervisor. Since virtual machines have their own operating system kernels, we need to install an operating system in our virtual machine before we can do anything interesting with it. 

The following steps should all be run on a public department machine. If you have a personal computer with an Intel processor and a Linux distribution installed, you may be able to run this lab locally, but course staff will only be able to provide limited help with technical issues related to other environments.

Installing a new operating system requires interacting with an installer GUI. This can be done remotely using X11 forwarding over SSH. To set up X11 forwarding, add `-X -C` to your SSH command. If you are using an SSH config file, add the following lines to the configuration for the lab machine you are acessing:
```
ForwardX11 yes
Compression yes
```
We recommend working in this lab from within GDC to reduce latency between the remote server running the VM and your personal computer. If you cannot physically access GDC or otherwise have latency issues, please contact course staff.

**Note: this part of the assignment can be done with or without KVM.** KVM can be disabled by removing `-enable-kvm` from the commands. The VM will run more slowly, but the steps for this part of the lab will remain the same.

1. Download the Ubuntu Desktop 22.04 ISO using
```
wget https://releases.ubuntu.com/22.04.1/ubuntu-22.04.1-desktop-amd64.iso?_ga=2.1007156.421406901.1670951282-1358803642.1670951282
```
This ISO file contains the installer for Ubuntu 22.04. 

2. Run the following command to create a VM image file:
```
qemu-img create -f qcow2 lab0.img 10G
```
`qemu-img` is a utility for creating disk images compatible with QEMU. A disk image is a file that is structured like a disk volume or storage device. We will use this disk image as a virtualized hard drive: we'll install Ubuntu on it, and QEMU will store files created within the VM within this file. The `-f` argument specifies the format for the image file; we're using qcow2, a format for copy-on-write image files supported by QEMU.

3. Boot the VM with the Ubuntu ISO using the following command:
```
qemu-system-x86_64 -boot d -cdrom <path to Ubuntu ISO> -m 8G -enable-kvm -hda lab0.img
```
Breaking down this command:
- `-boot` argument lets us specify the boot order (the order in which the computer checks devices for an OS). `d` tells QEMU to look at CD-ROMs first. 
- `-cdrom` indicates that a certain file (in this case, our Ubuntu ISO) should be treated as a CD-ROM. Since we indicated that CD-ROMS are first in the boot order, QEMU will look at this ISO first.
- `-m 8G` indicates that the VM is allowed to use at most 8GB of host memory. 
- `-enable-kvm` enables virtualization support from KVM, a Linux kernel module that accelerates virtualization. 
- `-hda lab0.img` tells QEMU to use lab0.img as a virtual hard drive.

The VM will take some time to start up. It will open a new window on your local machine and it may appear to hang on a blank screen for a few minutes. Eventually, you will see a window presenting options to try or install Ubuntu. Follow the prompts to install Ubuntu, using default installation options.


4. Once Ubuntu has been installed, close the window or use Ctrl-C to quit the current QEMU instance. We'll restart QEMU with a different command that boots from our disk image rather than the Ubuntu ISO.

Run
```
qemu-system-x86_64 -hda lab0.img -m 8G -enable-kvm 
```

Edit the file /etc/default/grub using your preferred text editor (it will require sudo) and change `GRUB_CMDLINE_LINUX=""` to `GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"` and run `sudo update-grub`. This will allow us to interact with the VM without the GUI, which is often more convenient. You can continue to work with the VM via the GUI, but we can now use the following command to start the VM and interact with it entirely over the command line:
```
qemu-system-x86_64 -hda lab0.img -m 8G -enable-kvm -nographic
```

# Part 4: Questions

Please write up answers to these questions in a Markdown document and push it to your group's Github Classroom repository for Lab 0 prior to the deadline. There is no code deliverable for this assignment.

1. Run `lscpu` on both the host and the guest with KVM enabled. Briefly describe some of the differences that you see. Is the guest aware that it is being virtualized? How do you know?
2. Now boot the guest without KVM (remove `-enable-kvm` from the command) and run `lscpu` again. What has changed? Is the guest aware that it is being virtualized? How do you know?
3. Within both KVM and non-KVM VMs, run `systemd-analyze` and record the total startup time. Which VM has a longer startup time? Why?

For the next few questions, we'll use some tracing functionality built into QEMU. This tracing infrastructure was introduced to aid in debugging and performance profiling, but we'll use it to better understand how VMs work with QEMU and KVM. See the QEMU tracing documentation for more info: https://qemu-project.gitlab.io/qemu/devel/tracing.html

4. For this question, we'll need to use the GUI. Remove `-nographic` from the QEMU command (you can add it back after this question) and enable KVM. Add `--trace ps2_put_keycode,file=$HOME/qemu.out` to the QEMU command and start the VM. This will record each time a function `ps2_put_keycode` is called within QEMU and log it to the file `$HOME/qemu.out`. To view contents of this file as they are appended, run `tail -f $HOME/qemu.out` from a different terminal window.

    Wait for the VM to boot. Figure out what triggers the `ps2_put_keycode` event by interacting with the VM (there may be a slight delay between the triggering event and the logs appearing in the file). What triggers this event? Why does QEMU handle this event?

5. Now, trace the `kvm_run_exit` event by adding `--trace kvm_run_exit,file=$HOME/qemu.out` to the QEMU command. This lets us see when and why the VM exits to KVM. Make sure KVM is enabled. What reasons do you see? Use https://github.com/qemu/qemu/blob/master/linux-headers/linux/kvm.h to interpret them.

6. Trace the `translate_block` event with and without KVM. How does the output change with KVM vs. without? Why do you think this is? (Hint: think about the name of the event, and read the first few sections of https://www.qemu.org/docs/master/devel/tcg.html and https://lwn.net/Articles/705160/)


Now we'll use `strace`, a tool to record system calls and signals used within a program.

7. Use `strace` to trace only `ioctl` calls made by the VM (read the `strace` man page [here](https://man7.org/linux/man-pages/man1/strace.1.html) for help). We suggest redirecting the output (note that `strace` prints to stderr, not stdout) to a separate file. `ioctl` is a system call that allows a userspace program (like QEMU) to communicate with a kernel module (like KVM) and to perform operations that are not expressed by regular system calls. See [the ioctl man page](https://man7.org/linux/man-pages/man2/ioctl.2.html) for more information.

    Boot the VM with KVM enabled - you should see lots of `ioctl` calls where the request type (the second argument) begins with `KVM_`. Pick out five different `KVM_` request types (check the top of the log file) and, using the [KVM API documentation](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt) as a reference, briefly describe what those requests do.

# Grading rubric

Total points: 20
Questions 1 through 4 are worth 2 points each.
Questions 5 through 7 are worth 4 points each.

# Submission details

**Due date: January 24, 2023**

Assignments will be submitted using GitHub Classroom. See Canvas for the invite link. You may make as many branches and commits as you would like while working on the lab. When you are ready to submit, make sure all changes have been pushed to the main branch of your group's repository in the classroom. GitHub Classroom will snapshot the repository at the due date, and course staff will leave feedback via a pull request.

Please indicate if you are using any slip days in the Markdown file with the answers to the lab questions.

# Contact details

Post on Piazza if you have questions or trouble with the assignment. whenever possible, please post publicly - chances are, other students have the same question or may be able to help. You may post your question anonymously. If you want to share code with course staff, use a private Piazza question. 

