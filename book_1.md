# Definitions

* *Virtualization* is the application of the layering principle through enforced modularity
whereby the exposed virtual resource is identical to the underlying physical resource
being virtualized.

* A *virtual machine* is an abstraction of a complete compute environment through
the combined virtualization of the processor, memory, and I/O components of a
computer.

* The *hypervisor* is a specialized piece of system software that manages and runs
virtual machines.

* The *virtual machine monitor (VMM)* refers to the portion of the hypervisor that focuses
on the CPU and memory virtualization.

## Virtualization
The idea is grounded in two fundamental principles:

1. Layering - the presentation of a single abstraction, realized by adding a level of
indirection when (i) the indirection relies on a single lower layer and (ii) uses a well-
defined namespace to expose the abstraction.

2. Enforced modularity - which guarantees that the clients of the layer cannot bypass it
to access the physical resource directly of have visibility into the underlying physical
namespace.


These principles can be applied across many areas in CS - not just to virtualize CPUs.
Indeed, RAID is one such example of the application of these techniques.

### A few examples:
Virtualizing in Computer Architecture

* The MMU is a great example - it adds a level of indirection which hides the physical
  addresses from applications. Both virtual and physical memory expose the same abstraction
  of byte-addressable memory - so the same ISA can operate identically with virtual
  and physical memory.
  
Virtualization within Operating Systems

* Operating systems essentially virtualize the underlying hardware for user applications.
* It controls the MMU to expose isolated address spaces
* It schedules threads on physical cores
* It mounts multiple file systems into a distinct namespace

# Virtualization in IO Subsystems
Ubiquitous in disks and disk controllers.

Achieved by combining three simple techniques:

1. Multiplexing
```
          -> X
X -> Virt -> X
          -> X
```
2. Aggregation
```
X -> 
X -> Virt -> X
X -> 
```

3. Emulation
```
X -> Virt -> Y
```


# Virtual Machines
A VM is a complete computer environment with its own isolated processing capabilities,
memory, and communication channels.

This definitions applies to a range of distinct, incompatible abstractions:

* language-based virtual machines - such as the Java Virtual Machine

* lightweight virtual machines - rely on hardware and software isolation mechanisms to
ensure applications are securely isolated from others and the OS.
  - i.e. Docker & FreeBSD Jail

* system-level virtual machines - the isolated environment resembles the hardware of a computer
so that hte VM can run a standard operating system & its applications in FULL isolation
from other VMs and the rest of the environment.


* hypervisor - relies on direct execution on the CPU for max efficiency

* machine simulator - implemented as a normal user-level application aiming to provide an
accurate simulation of the virtualized architecture. Often runs at a fraction of the native
speed.

# Hypervisors
Hypervisor is a system that runs VMs with the goal of minimizing execution overheads.

Popek & Goldberg formalized the idea:

1. The VMM provides an environment for programs that is essentially identical with the
original machine.
2. Programs running in this environment show at worst only minor decreases in speed
3. The VMM is in complete control of system resources


This is consistent with the broader definition of virtualization: the hypervisor applies
the layering principle with 3 criteria:

1. Equivalence - the exposed resource is equivalent with the underlying computer.
2. Safety - virtual machines are isolated from each other and the hypervisor
3. Performance - at most, a minor decrease in speed


# Type-1 vs. Type-2 Hypervisors

* Type-1 hypervisors directly control all resources of the physical computer
  - the VMM runs on a bare machine
 
* Type-2 hypervisors operates "as part of" or "on top of" an existing host OS.
  - the VMM runs on an extended host, under the host OS

In the type-1 environment, the VMM must perform the system's scheduling and (real)
resource allocation. Thus the type-1 VMM may include code not specifically necessary
for virtualization (i.e. scheduler).

Both types are in common use:

* Type-1 Examples
  - VMware ESX Server
  - Xen
  - Microsoft Hyper-V

* Type-2 Examples
  - VMware Fusion
  - VMware Workstation
  - KVM
  - Parallels
  - Oracle VirtualBox
  
# A sketch hypervisor: multiplexing and emulation

A note on distinction - a machine simulator emulates the virtual machine's ISA, a hypervisor
multiplexes it.

In its basic form, a hypervisor *multiplexes* the physical processing element (PE) across
VMs, and emulates everything else (i.e. IO bus, IO devices).

Multiplexing and emulation is sufficient, as IO ops in today's systems are implemented via
reasonably high-level operations e.g. - device drivers issue commands to send a list of
network packets or a 32 KB disk request.


A hypervisor emulates the hardware/software interface of at least one representative device
per category, i.e. one disk device, one network device, one screen device, etc. and
uses the available physical devices to issue the actual IO.

IO emulation is the preferred approach since it's more portable - the VM sees the same
virtual hardware, even when running on a platform with different hardware devices.

Modern hardware provides architecture support for IO virtualization to enable
multiplexing of classes of IO devices (i.e. SSD vs HDD [I am assuming...]).

