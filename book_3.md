# Virtualization without Architectural Support
This chapter presents 3 systems - DISCO, VMware Workstation, and Xen - which were
developed without / before hardware support for virtualization.

# Disco
Architected around the trap-and-emulate approach formalized by Popek & Goldberg.

Disco runs in kernel mode; guest OSes (designed to run in kernel mode) run in
supervisor mode, and applications run in user mode.

* Equivalence
  - Disco didn't attempt to run binary-compatible kernels as guests.
  - MIPS instructions are location-sensitive due to hard-coded virtual
    memory segment ranges.
  - As a result, guest OSes needed to be re-compiled to run on Disco

* Safety
  - Disco isolated VMs without assumptions about the guest.
  - Relied on virtual memory and MIPS' three privilege levels (kernel, supervisor, user)
  - Supervisor code can't issue privileged instructions - thus triggering traps

* Performance
  - Popek and Goldberg assumed traps were rare
  - Doesn't hold on RISC arch's like MIPS
  - To address this, Disco introduced special memory pages to virtualize
    read-only privileged registers, hypercalls, and a larger TLB

## The L2TLB
MIPS uses a software loaded TLB - TLB misses are handled by the OS in software.
Since the page table format is defined by the OS, not the arch, an OS agnostic
hypervisor is incapable of handling TLB misses directly.

Instead, it needs to transfer execution to the guest's TLB miss handler.
To maintain performance, Disco added a software implemented TLB (L2TLB) that
ran on each TLB miss and checked to see if it had a matching virtual address
it could add to the physical TLB.

The L2TLB replaces the original TLB in the architecture of the virtual machine.

# VMWare Workstation - x86-32
Architected as:

- a pure virtualization solution compatible with existing unmodified guest OSes
- a type-2 hypervisor for Linux and Windows host OSes

* Equivalence
  - x86-32 has 17 virtualization-sensitive non-privileged instructions, so trap-and-emulate is out
  - VMWare combined direct execution with dynamic binary translation (DBT)
  - DBT is an efficient form of emulation

* Safety
  - Made extensive use of segment truncation to isolate virtual machines
  - Only supported a subset of x86-32 necessary to run supported guest OSes
  - Unsupported requests abort execution

* Performance
  - Aimed to run relevant workloads at near native speeds & worst case as if
    they were running natively on 1-gen older hardware.

## Dynamic Binary Translation (DBT)
* An efficient form of emulation
* Rather than interpret instructions one-by-one, compile groups of instructions,
  often basic blocks, into fragments of executable code.
* Translated code is stored in a large buffer called the translation cache so that
  it can be reused multiple times.
* Includes optimizations like *chaining* to jump between compiled fragments.

VMware used DBT so that instead of executing or interpreting the original VM instructions,
the processor natively executed the corresponding translated sequence from the translation cache.

### Address Space
Challenge to ensure the VMM could share an address pace safely with the virtual machine
without being visible to the virtual machine, and to execute with minimal performance
overhead.

VMWare's VMM used segmentation (ONLY) for protection. It statically divided the linear address space
into two regions - one for the VM and one for the VMM.

The VM segments were truncated by the VMM to ensure they did not overlap with the VMM.

* Guest user code ran in %cpl=3
* Guest OS code ran in %cpl=1
* VMM ran %cpl=0

### World Switch
Like a context switch - but EVERYTHING is saved during transitions between
the OS and the VMM.

# XEN - Paravirtualization
Xen used paravirtualization to "undefine" the 17 sensitive x86 instructions and replace
them with hypercalls and additional memory-mapped system registers.

The Xen hypervisor relied on this modified x86-32 arch to run virtual machines
using *direct execution** only.

Xen exposed a simplified MMU arch that directly, but safely, exposes host-physical memory
to the guest OS.

  - The guest OS combines virtual-to-guest-physical mappings with guest-physical to
    host-physical information readable from the hypervisor to generate virtual-to-host-
    physical mappings, which it passes to the hypervisor for validation.

## Xen's para interface

## Memory
- Segmentation :: Cannot install fully privileged segment descriptors and cannot overlap
  with the top end of the linear address space.

- Paging :: Guest OS has direct read access to hardware PT; updates batched and validated
  by the hypervisor. A 'domain' may be allocated discontinuous host physical pages.

## CPU
- Protection :: Guest OS must run at a lower privilege level than Xen

- Exceptions :: Guest OS must register a descriptor table for exception handlers with Xen.
  - Aside from page faults, the handler remains the same.

- System Calls :: The guest OS may install a "fast" handler for system calls,
  allowing direct calls from an application to its guest OS and avoiding
  indirection through Xen on every call.

- Interrupts :: Hardware interrupts are replaced with a lightweight event system

- Time :: Each guest OS has a timer interface and is aware of both "real" and "virtual" time.


With these changes, Xen operates as a trap-and-emulate hypervisor, unlike the original
VMware.

# Lightweight Paravirtualization for ARM
KVM's paravirtualization for ARM v5 is essentially achieved via a script with regexes
to replace sensitive instructions with traps. After trapping, the sensitive instructions
are emulated in software.

KVM needs to retrieve the original sensitive instruction and its operands to be able
to emulate it. KVM for ARM does this by defining an encoding of all the sensitive
non-privileged instructions and their operands into trap instructions.

How KVM does it:

The `SWI` instruction always traps on ARM and is normally used for making system calls.
If `SWI` traps to the hypervisor and the system is already in privileged mode, KVM
uses the 24-bit immediate field (payload of the trap to encode the sensitive
instruction and its operands (KVM assumes the guest OS doesn't make syscalls to itself).

However 24-bits isn't enough to encode all sensitive instructions. So, KVM also uses
ARM's coprocessor access functions, which also trap.

TODO (pg 49): When does cpl actually change? I thought the hardware changes it during the trap
instruction. But if that's the case, how can the hypervisor check if the virtual CPU
is in privileged mode?


