# x86-64 MMU Virtualization with Extended Page Tables

Hypervisors must virtualize physical memory so that each VM has the illusion
of managing its own contiguous physical memory.

So, we have the guest OS managing:
* virtual address -> guest-physical memory

And the hypervisor managing:
* guest-physical -> host physical memory

In the absence of architecture support, hypervisors rely on shadow paging where the
hypervisor manages a "composite set of page tables" mapping:
* virtual memory -> host physical memory

## Extended Paging (nested page tables)
Extended page tables provide architectural support for MMU virtualization
and eliminates the need for software-based shadow paging.

Combines the classic hardware-defined page table maintained by the guest-os with
a second page table structure, maintained by the hypervisor, which specifies
guest-physical to host-physical mappings.

### Example

* guest-physical tree is n-level
* guest-physical to host-physical is m-level

On a TLB miss, the hardware walks the guest page table structure
consisting of guest-physical pages with each guest-physical
reference required to be individually mapped to its own host-physical
address.

To resolve each reference (to a page table) in the first tree, the processor
must perform *m* references in the second tree, and then lookup the mapping in
the first tree.

There are *n* such steps, each requiring *m* + 1 references.
These n * (m + 1) references lead ot the desired guest-physical address.
Another *m* lookups are necessary to convert this guest-physical address into
a host-physical address.

Thus the number of lookups required is n * m + n + m

## Virtualizing Memory in KVM

KVM can run with extended paging on or off. In practice, extended paging is always
used except perhaps in the case of nested virtualization.

* QEMU, as a userspace process, allocates the guest-physical memory in a contiguous
  portion of its own virtual address space.
  
  i)   This leaves the details of memory management and allocation to the host OS
  ii)  Provides convenient access from userspace to the guest-physical address space;
       this is used to emulate devices that perform DMA to and from guest-physical memory
  iii) When the QEMU process is scheduled - including when the guest VCPU is running - the
       root-mode %cr3 register defines an address space that includes QEMU and the contiguous
       guest-physical memory.

* The VM manages its own page tables. With nested paging, the non-root cr3 register points
  to the guest page tables (which define the address space in guest-physical memory).

* NOTE: KVM does not need to track changes to the guest page tables nor context switches.
  Assignment to cr3 in non-root mode do not need to cause a `#vmexit` when nested paging
  is enabled.

* NOTE: on x86 the format of the regular page tables (pointed by cr3) is different
  from the format of the nested tables. The KVM module is responsible for managing
  nested page tables (which are rooted at `eptp` register on Intel).

### How-To & Considerations

It is easier for the hypervisor to access guest-physical address space than to access
guest-virtual address space.

Specifically, the VT-x arch doesn't include support for root mode software to access the
virtual address space of non-root mode environments.

It's easy to access a guest-physical address from user mode (QEMU): just add a constant
offset to the address to index into the user mode linear address space. The KVM module
can actually use the same technique since the user mode process is mapping the address space.

However, it's hard to access the guest's virtual address space since those mappings are
only present in the MMU of the processor while the virtual machine is executing.

Since the KVM decoder needs to read %eip to get the current instruction, and deref its
memory operands, KVM must access the guest page tables in software.

NOTE: KVM & Linux kernel need to keep the nested page tables consistent with the guest-physical
address space of the host OS.

Example: If the guest-physical mapping is changed by adding a new page on `#vmexit` from
the VM, the same mapping must be reflected in the address space of the QEMU process.

Example: If the host OS swaps out a page from the QEMU process, the guest-physical
extended mapping must also be removed.

### Performance

Nested paging / extended page tables provides significant benefits over shadow paging
by eliminating the need for memory tracing, which accounts for as much as 90% of all
`#vmexits` in typical workloads.

It's not free however - two-dimensional page walk with 4-level paging results in 24
memory references to service a single TLB miss.


# Follow-up Notes
The core reason why VMWare's DBT implementation is faster than faulting on each update
is that the translated VM code can inform the VMM of multiple updates to the page table
which can later be handled in a batch with only one trap.

We can avoid faults for certain types of updates, but not others:
RW page -> RO page :: can't avoid trap - need to update immediately

RO -> RW :: can avoid; trap will happen when OS / user tries to write the page later

TODO: w/ EPT, how does the hypervisor know when the guest has released a page?
