---
title: Windows Subsystem for Linux Architecture
description: Detailed breakdown of the Windows Subsystem for Linux's architecture.
keywords: wsl, windows, windowssubsystem
author: scooley
ms.author: scooley
ms.date: 8/4/2017
ms.topic: article
ms.prod: windows-subsystem-for-linux
ms.service: windows-subsystem-for-linux
ms.assetid: e744587f-7450-4238-afd6-a36b2400bf24
---

# Architecture

This section documents the Windows Subsystem for Linux's underlying architecture.  It will touch on WSL's history, architecture, and core components.

The refrence documentation here assumes deep understanding of operating system architecture.
 
> Are you looking for a more general overview of WSL?  Read more [about WSL](../about.md).


## History of Windows Subsystems
Microsoft Windows NT was originally designed to support software written for a variety of platforms.  That allowed developers to run software on Windows without rewriting it, even if it had been written for OS/2 or Unix environments.

One way Windows NT did that was by running environment subsystems (like Win32 or POSIX) rather than making developers write software using kernel interfaces.  Each subsystem presented a different interface for applications to use without considering implementation details inside the kernel.

As an example, Windows NT 4.0 has four environmental subsystems: Win32, DOS, OS/2, and POSIX.

Early subsystems were implemented as user mode modules that issued NT system calls based on the API they presented to applications for that subsystem. All applications were PE/COFF executables, a set of libraries and services to implement the subsystem API and NTDLL to perform the NT system call. When a user mode application launched, the loader invoked the right subsystem to satisfy the application dependencies based on the executable header.

Later versions of subsystems replaced the POSIX layer to provide the Subsystem for Unix-based Applications (SUA).  This composed of user mode components to satisfy:

1. Process and signal management
2. Terminal management
3. System service requests and inter process communication

The primary role of SUA was to encourage applications to get ported to Windows without significant rewrites. This was achieved by implementing the POSIX user mode APIs using NT constructs. Given that these components were constructed in user mode, it was difficult to have semantic and performance parity for kernel mode system calls like fork(). This model required programs to be recompiled which meant continuous feature porting.  It was undenyably a maintenance burden.

Over time, those subsystems were retired. However, since the Windows NT Kernel was architected to allow new subsystem environments, we were able to use the initial investments made in this area to develop the Windows Subsystem for Linux.

## Windows Subsystem for Linux
WSL contains both user mode and kernel mode components. It is primarily comprised of:

1. User mode session manager service that handles the Linux instance life cycle
2. Pico provider drivers (lxss.sys, lxcore.sys) that emulate a Linux kernel by translating Linux syscalls
3. Pico processes that host the unmodified user mode Linux (e.g. /bin/bash)

It is the space between the user mode Linux binaries and the Windows kernel components where the magic happens. By placing unmodified Linux binaries in Pico processes we enable Linux system calls to be directed into the Windows kernel. The lxss.sys and lxcore.sys drivers translate the Linux system calls into NT APIs and emulate the Linux kernel.

![WSL Components](media/wsl-components.png)

## LXSS Manager Service

The LXSS Manager Service is a broker to the Linux subsystem driver it also lets wsl.exe invoke Linux binaries. Basically, it synchronizes install/uninstall, manages Linux instances, and prevents Linux binaries from launching when WSL is in the process of installing or uninstalling.

All Linux processes launched by a particular user go into a Linux instance. That instance is a data structure that keeps track of all LX processes, threads, and runtime state. The first time an NT process requests launching a Linux binary an instance is created.

Once the last NT client closes, the Linux instance is terminated. This includes processes that launched inside of the instance including daemons (e.g. the git credential cache).

## Pico Process
As part of Project Drawbridge, the Windows kernel introduced the concept of Pico processes and Pico drivers. Pico processes are OS processes without the trappings of OS services associated with subystems like a Win32 Process Environment Block (PEB). Furthermore, for a Pico process, system calls and user mode exceptions are dispatched to a paired driver.

Pico processes and drivers provide the foundation for the Windows Subsystem for Linux, which runs native unmodified Linux binaries by loading executable ELF binaries into a Pico process’s address space and executes them atop a Linux-compatible layer of syscalls.

## System Calls
WSL executes unmodified Linux ELF64 binaries by virtualizing a Linux kernel interface on top of the Windows NT kernel.  One of the kernel interfaces that it exposes are system calls (syscalls). A syscall is a service provided by the kernel that can be called from user mode.  Both the Linux kernel and Windows NT kernel expose several hundred syscalls to user mode, but they have different semantics and are generally not directly compatible. For example, the Linux kernel includes things like fork, open, and kill while the Windows NT kernel has the comparable NtCreateProcess, NtOpenFile, and NtTerminateProcess.

The Windows Subsystem for Linux includes kernel mode drivers (lxss.sys and lxcore.sys) that are responsible for handling Linux system call requests in coordination with the Windows NT kernel. The drivers do not contain code from the Linux kernel but are instead a clean room implementation of Linux-compatible kernel interfaces. On native Linux, when a syscall is made from a user mode executable it is handled by the Linux kernel. On WSL, when a syscall is made from the same executable the Windows NT kernel forwards the request to lxcore.sys.  Where possible, lxcore.sys translates the Linux syscall to the equivalent Windows NT call which in turn does the heavy lifting.  Where there is no reasonable mapping the Windows kernel mode driver must service the request directly.

As an example, the Linux fork() syscall has no direct equivalent call documented for Windows. When a fork system call is made to the Windows Subsystem for Linux, lxcore.sys does some of the initial work to prepare for copying the process. It then calls internal Windows NT kernel APIs to create the process with the correct semantics, and completes copying additional data for the new process.

## File system
File system support in WSL was designed to meet two goals.

Provide an environment that supports the full fidelity of Linux file systems
Allow interoperability with drives and files in Windows
The Windows Subsystem for Linux provides virtual file system support similar to the real Linux kernel. Two file systems are used to provide access to files on the users system: VolFs and DriveFs.

### VolFs
VolFs is a file system that provides full support for Linux file system features, including:

Linux permissions that can be modified through operations such as chmod and chroot
Symbolic links to other files
File names with characters that are not normally legal in Windows file names
Case sensitivity
Directories containing the Linux system, application files (/etc, /bin, /usr, etc.), and users Linux home folder, all use VolFs.

Interoperability between Windows applications and files in VolFs is not supported.

### DriveFs
DriveFs is the file system used for interoperability with Windows. It requires all files names to be legal Windows file names, uses Windows security, and does not support all the features of Linux file systems. Files are case sensitive and users cannot create files whose names differ only by case.

All fixed Windows volumes are mounted under /mnt/c, /mnt/d, etc., using DriveFs. This is where users can access all Windows files. This allows users to edit files with their favorite Windows editors such as Visual Studio Code, and manipulate them with open source tools in Bash using WSL at the same time.

 

In future blog posts we will provide additional information on the inner workings of these component areas. The next post will cover more details on the Pico Process which is a foundational building block of WSL.

 