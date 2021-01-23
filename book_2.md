# The Popek/Goldberg Theorem

Describes an ISA model with the following properties:

* The processor has two execution levels: supervisor mode and user mode

* Virtual memory is implemented via segmentation (instead of paging) using a single segment
  with base B and limit L.
  - The segment defines the range [B, B+L) of the linear address space for valid virtual
    addresses [0, L)

* Physical memory is contiguous, starting at address 0, and the amount of physical memory is
  known at processor reset time (SZ).

* The processor's system state, called the processor status word (PSW) consists of the tuple
  (M, B, L, PC):
  
  - the execution level M = {s, u};
  - the segment register (B, L); and
  - the program counter (PC), a virtual address

* The trap architecture has provisions to first save the content of the PSW to a well-known
  location in memory, and then load into the PSW the values from another well-known location in
  memory.
  - The trap architecture mechanism is used to enter into the operating system following a
  system call or an exception in executing an instruction.

* The ISA includes at least one instruction or sequence that can load into the hardware PSW the
  tuple (M, B, L, PC) from a location in virtual memory. This is necessary to resume execution
  in user mode after a system call or trap.


## The question(s):

Given a computer that meets this basic architectural model, under which precise conditions
can a VMM be constructed, so that the VMM:

* can execute one or more virtual machines;

* is in complete control of the machine at all times;

* supports arbitrary, unmodified, and potentially malicious OSes designed for the same arch

* be efficient to show at worst a small decrease in speed


Any VMM meeting these conditions should display the following properties:

- Equivalence: The virtual machine is essentially identical to the underlying processor

- Safety: the VMM must be in complete control of the hardware at all times without making any
  assumptions about the software running in the VM. A VM is isolated from the underlying
  hardware as if it was running on a distinct computer.

- Performance: The speed of a program in the virtualized environment must be at worst a minor
  decrease over the execution time when run directly on the underlying hardware.


# Theorem 1:
For any conventional third-gen computer, a VMM may be constructed if the set of sensitive
instructions for that computer is a subset of privileged instructions.

An instruction is *control-sensitive* if it can update the system state

An instruction is *behavior-sensitive* if its semantics depend on the actual values set
in the system state

Otherwise the instruction is *innocuous*.

This is key. Since all sensitive constructions are privileged, the OS guest can run in user
mode, allowing the VMM to catch and emulate any privileged instructions (see the text for
more details).

# Theorem 3:
(theorem 2 not discussed by the text)

An instruction is *user-sensitive* if it is behavior-sensitive or control-sensitive in supervisor
mode, but not in user mode.

Theorem:

A hybrid VMM may be constructed for any conventional third-generation machine in which the set
of user-sensitive instructions are a subset of the privileged instructions.

* The hybrid VMM acts like a normal VMM when the virtual machine is running applications, i.e.,
  is in guest-user mode.

* Instead of direct execution, the hybrid VMM interprets in software 100% of the instructions
  in guest-supervisor mode. In other words, it interprets the execution of all paths through the
  guest-operating system. Despite the high overheads of interpretation, this approach might
  be practical as long as the portion of time spend in the OS is small.

An example is the `JRST 1` instruction on the PDP-10. In user mode, `JRST 1` returns from a
subroutine. In supervisor mode `JRST 1` returns to user mode.

The issue is that no trap happens when `JRST 1` is executed while the guest OS is running
in user mode. This prevents the VMM from recognizing when when the guest-OS transitions
back to the application. The result is that the VMM thinks the guest is still in guest-
supervisor mode even though the guest OS thinks it is in user mode.
