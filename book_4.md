# x86-64: CPU Virtualization with VT-x
Architectural support for virtualization.

## Road map
* CPU (ch 4)
* MMU (ch 5)
* I/O subsystem (ch 6)

VT-x is an Intel technology that virtualizes the CPU itself.
AMD-v is AMD's approach, which is architecturally similar to VT-x.


## VT-x Core Design

Intel added a new "root" mode, in which the Host OS and/or Hypervisor runs, and a
"non-root" mode which guest OSes and their user tasks run.

VT-x duplicates the entire processor state, which is necessary for backward
compatibility and full equivalence for virtualized workloads.

Design principle summary from the book:
```
In an architecture with root and non-root modes of execution
and a full duplicate of processor state, a hypervisor may be constructed
if all sensitive instructions (according to the non-virtualizable legacy architecture)
are root-mode privileged.

When executing in non-root mode, all root-mode-privileged instructions are either
(i) implemented by the processor, with the requirement that they operate exclusively
on the non-root duplicate of the processor or (ii) cause a trap.
```
Pretty cool.

Observations:

1. Unlike the original Popek and Goldberg theorem, this rephrasing does not take into
account whether instructions are privileged or not (w/r/t %cpl), but the orthogonal
question of whether they are root-mode privileged.

2. These traps are sufficient to meet the equivalence and safety criteria. This is
similar to the original theorem.

3. However, reducing transitions by implementing certain sensitive instructions in the
hardware is necessary to meet the performance criteria for virtualization.

### VACS
So, the entire processor state is "duplicated" / virtualized for the guest OS and
saved and restored on `#vmexit` and `#vmresume`. This state is the VMCS and is backed
by a region in memory.

However, the VMCS memory layout is not specified - the processor may even save
some of the state in the processor itself. So, the in-memory state of the
current VMCS is undetermined and the layout of the VMCS is undefined. The
hypervisor software must access portions of the guest state via the `vmread` and
`vmwrite` instructions.

See here regarding shadow page tables:
https://stackoverflow.com/a/9834730/8213783

## How it works
The reason for a `#vmexit` is stored in a dedicated register `vmcs.exit_reason`.

Reasons for exiting can be grouped into a few categories:

1. Any attempt to execute a root-mode-privileged instruction

2. The new `vmcall` instructions added to allow explicit transitions from non-root mode
   and root-mode.
   
3. Exceptions resulting from the execution of any innocuous instruction in non-root mode
   that cause a trap. i.e. page faults (`#PF`) caused by shadow paging, access to memory-mapped I/O,
   general-purpose faults (`#GP`).

4. EPT violations - a subset of page faults caused when extended page mapping is invalid.

5. Internal interrupts that occurred while the CPU was executing in non-root mode (i.e. disk I/O).

6. ISA extensions added by VT-x to support virtualization are control-sensitive and cause `#vmexit`
   with a distinct exit reason. This type of exit never occurs with "normal virtualization", but can
   occur with nested virtualization.

# Overview of KVM

* Type-2 hypervisor running on top of / integrated with Linux

* KVM relies on QEMU to emulate I/O. QEMU provides a userspace implementation of all I/O front-end
  device emulation and the Linux kernel responsible for the I/O backend (via normal system calls).
  
* KVM is responsible for multiplexing the CPU and MMU of the processor.

## The Kernel Module
The kernel module only handles the CPU and platform emulation. This includes:

* CPU emulation
* memory management
* MMU virtualization
* interrupt virtualization
* chipset emulation (APIC, IOAPIC)

Reminder: I/O is handled by QEMU in user space

### Emulation
Unfortunately the information in VMCS is often inadequate to handle the `#vmexit` without decoding the
instruction that caused it.

KVM therefore must include a general-purpose decoder and general-purpose emulator for all instructions.

## Role of the host operating system
The core KVM vm execution loop looks like this:

The *outer loop* is in user mode and repeatedly: 
* enters the KVM kernel module via an ioctl to the char device /dev/kvm

* the KVM kernel mod executes the guest code until
  i) the quest initiates I/O via an I/O instruction or memory mapped I/O; or
  ii) the host receives an external I/O or timer interrupt

* the QEMU device emulator then initiates the I/O (if necessary); and

* in the case of external I/O or timer interrupt, the outer loop may return back
  to the KVM mod without further side-effects. This step into user space is
  essential though since it gives the host OS the opportunity to make global
  scheduling decisions

The *inner loop* inside the KVM module repeatedly:
* restores the current state of the virtual CPU

* enters non-root mode using the `vmresume` instruction - after that the virtual machine
  executes in that mode until the next `#vmexit`

* handles the `#vmexit` according to the exit reason

* if the guest issued a programmed IO operation or mem-mapped IO instruction, break loop
  and return to userspace

* if the `#vmexit` was caused by an external event, break loop and return to user space
