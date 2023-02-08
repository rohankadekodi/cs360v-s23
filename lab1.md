# Project 1  Lab 1

# Part 1: Introduction

In the previous lab, you worked with QEMU and KVM to become more familiar with virtualization. In this and the subsequent labs, you will implement part of a paravirtual hypervisor within the JOS operating system. The labs cover bootstrapping a guest OS, programming extended page tables, emulating privileged instructions, and using hypercalls to implement hard drive emulation. By the end of Project 1, you wil be able to launch a JOS-in-JOS environment.

You will do this lab with your Project 1 group. 

This README series contains some background related to Project 1. Reading this document will help you understand the pieces you will be implementing on a high level as you work on Project 1.

The README series is broken down into 4 parts:
1. [Bootloader](bootloader.md) explains what happens when you boot JOS (in our case, create or boot the guest).
2. [Virtual Memory](virtual_memory.md) explains how JOS manages virtual memory. This part also contains details about segmentation and paging, which are a good background for the project in general.
3. [Environments](environments.md) provides information on the environment structure. Environments in JOS are similar to Linux processes and are also used to manage and run guests.
4. [File System](file_system.md) which will help you understand the last few labs, where we handle vmcalls related to reading and writing of data to a disk.

**Due date: February 2, 2023**

# Part 2: Setup

Please see Canvas for information on how to access course servers.

We recommend running all projects on the course servers since they are already set up properly to run the assignments, but if you have a personal computer with x86 hardware (*not* AMD or a newer Mac) you may be able to run the projects locally or within a different hypervisor like VirtualBox. See [the tools page](tools.md) for information on how to set up the labs from scratch. Course staff will only be able to provide limited support for alternative environments.

The servers are only accessible via a gateway server, `pig.csres.utexas.edu`. We have set up a user on this server that all students in this class will use. In order to access your group's assigned server for the course, you need to first SSH into the gateway server, then to the assigned course server. You can add entries as follows to your SSH config file (`~/.ssh/config` for Linux and MacOS) to simplify this process:
```
Host pig
   User cs360v-s23
   HostName pig.csres.utexas.edu
   IdentityFile <path to private key file>

Host cs360v
   User <group username>
   HostName <course server>
   ProxyJump pig
   IdentityFile <path to private key file>
```
After adding these entries, you can SSH to the course server by running `ssh cs360v`. 

Clone JOS onto your server from the provided Github Classroom repository.

Compile JOS using `make` and clear out the repo using `make clean`. We recommend using `make clean` frequently; previous students have found that the Makefile does not always automatically detect changes to JOS.

You can run the VMM from the shell by typing:
```
make run-vmm-nox
```
You should see something like this:
```
kernel panic on CPU 0 at ../vmm/vmx.c:65: vmx_check_support not implemented

Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```
Exit JOS by pressing `Ctrl-A`, followed by `X`. Release `Ctrl` and `A` before pressing `X`.

## GDB

Next, we will start GDB. GDB is an indispensible tool for systems programming and debugging and will be very useful throughout this course. See [the tools page](tools.md#GDB) for more information about using GDB.

You need to use a **patched** version of GDB for these assignments. We have already set this version of GDB up on the lab servers. Using other GDB versions will result in errors.

SSH to the course server, navigate to the JOS directory, and run the following command: 
``` 
make run-vmm-nox-gdb
```
This command starts JOS with support for GDB debugging. It will look like JOS is hanging, but it is really just waiting for input from GDB.

Now, to start GDB, open a new terminal window, SSH to the course server, and run from the JOS project directory:
```
gdb_7.7
```
Note that you will get subtle errors if you run GDB from a different directory.

Normally, you would now specify an executable, but because JOS is an operating system kernel running in a VM, we have to use **remote debugging** via GDB. The JOS project is set up to take care of this for you. When you start GDB, it will automatically run `target remote localhost:<port number>` to attempt to connect to your VM. When you build JOS and run `make run-vmm-nox-gdb`, the build system attempts to automatically select a unique port number and copies it to the `.gdbinit` file so that GDB automatically connects to the VM. We suggest always running `make run-vmm-nox-gdb` before starting GDB so that the port number is set up properly. 

On the terminal running GDB, you should see something like the following:
```
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

The target architecture is assumed to be i8086
[f000:fff0]    0xffff0:	ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
Hardware assisted breakpoint 1 at 0x1000e5: file kern/bootstrap.S, line 147.
```
If you *don't* see this, you may need to run `echo "set auto-load safe-path /" >> $HOME/.gdbinit` and start over.

You can now continue execution with `c` (continue), `n` (next), or `s` (step). See the GDB reference for further information.

# Part 3: Pre-lab questions

For the rest of this lab, you will need to read part of the Intel Software Developer's Manual. This manual can be dense and uses unfamiliar notation in many places. [Chapter 1 of this manual](https://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.html) provides useful information about how to read this notation. 

Please write up the answers to these questions in a Markdown document.

1. Based on what you know about JOS so far, will we be building a Type 1 or Type 2 hypervisor? Explain your reasoning.
2. The JOS hypervisor is paravirtualized. What are the benefits of paravirtualization over other virtualization techniques? What are the disadvantages?
3. In the second coding exercise for this lab, we will be using the `cpuid()` function to check several features of the CPU. We will use GDB to understand what this function does. 
   
   a. `cpuid()` is implemented in inc/x86.h using the `asm` keyword. What does this keyword mean? What does `volatile` mean here? How are the arguments to the `cpuid()` function used? (Hint: look up GCC's documentation on Extended Asm.)

   b. Set a breakpoint at the `cpuid()` call in `vmx_check_support()` in vmm/vmx.c and step past the `asm volatile` call. Make sure the first argument to this `cpuid()` call is 0 before stepping into it. What are the values of the `eax, ebx, ecx, edx` variables in hexadecimal at this point? Hint: [you can print program variables in GDB](https://sourceware.org/gdb/current/onlinedocs/gdb/Variables.html).

   c. Convert the contents of ebx, ecx, and edx from part (b) to strings using an ASCII table or converter. Using [the CPUID wikipedia page](https://en.wikipedia.org/wiki/CPUID#EAX=1:_Processor_Info_and_Feature_Bits) as a reference, explain what these values tell us about our CPU.

# Part 4: Coding exercise

## Part A: Track number of runs for an environment

JOS environments are roughly equivalent to Linux processes. We introduce the term "environment" instead of the traditional term "process" to stress the point that JOS environments do not provide the same semantics as UNIX processes, even though they are comparable. See the [Environments README](environments.md) for more information.

We want to be able to keep track of the number of times each environment has been run. JOS' `Env` struct has a `env_runs` field, but the current version of JOS does not use or update it properly. We have marked several places in the code with hints starting with `// HINT LAB 1`. Your task is to find all of these places and modify JOS to use this field according to the hints.

## Part B: Checking for VMX and EPT support

Next, we need to check whether our CPU supports VMX and extended paging. In the pre-lab, we examined how to use `cpuid()` to obtain information about the processor. You will now use this function and a similar function, `read_msr()` to check whether the CPU supports these virtualization features. To understand how to implement these checks, read Chapters 23.6, 24.6.2, and Appendices A.3.2 and A.3.3 of the [Intel Software Developer's Manual](https://www.cs.utexas.edu/~vijay/cs378-f17/projects/64-ia-32-architectures-software-developer-vol-3c-part-3-manual.pdf). 

Now, implement the `vmx_check_support()` and `vmx_check_ept()` functions in vmm/vmx.c.

### Checking your code

Project 1 includes an autograder, `gradeproject.py`, that we will use to check submissions and that you can use to check your work. The autograder cannot check the `env_runs` field or whether `vmx_check_ept()` is implemented correctly, but it does check that `vmx_check_support()` is implemented properly. 

To run all tests, run `python ./gradeproject.py`. To run specific test(s), add the names of the tests (in quotes, separated by spaces) to the end of the command. At the end of this lab, tests "VMX not supported" and "VMX supported" should pass; all other tests are expected to fail.

If these functions are properly implemented, an attempt to start the VMM will not panic the kernel, but will fail because the VMM can't map guest bootloader and kernel into the VM. The error will look something like this:
```
Error copying page into the guest - 4294967289
```

# Grading Rubric

Total points: 20

Each prelab question is worth 2 points (6 total).
Tracking the number of runs is worth 6 points.
`vmx_check_support()` and `vmx_check_ept()` are each worth 4 points.

# Submission details

**Due date: February 2, 2023**

Assignments will be submitted using GitHub Classroom. You will receive an invite link for the assignment when it is released. You may make as many branches and commits as you would like while working on the lab. When you are ready to submit, make sure all changes have been pushed to the main branch of your group's repository in the classroom. GitHub Classroom will snapshot the repository at the due date, and course staff will leave feedback and grade via a pull request.

Within your repository, please create a Markdown file for the pre-lab answers and ensure that all files required to run the assignment are present. Please indicate if you are using any slip days in the Markdown file.


# Contact details

Post on Piazza if you have questions or trouble with the assignment. Whenever possible, please post publicly - chances are, other students have the same question or may be able to help. You may post your question anonymously. If you want to share code with course staff, use a private Piazza question. 
