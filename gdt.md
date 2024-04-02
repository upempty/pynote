- segmentation
```
Because Linux is designed for a wide variety of platforms, some of which offer only limited support for segmentation, Linux supports minimal segmentation. Specifically Linux uses only 6 segments:
Kernel code.
Kerned data.
User code.
User data.
A task-state segment, TSS
A default LDT segment
All processes share the same user code and data segments, because all process share the same logical address space and all segment descriptors are stored in the Global Descriptor Table. ( The LDT is generally not used. )
Each process has its own TSS, whose descriptor is stored in the GDT. The TSS stores the hardware state of a process during context switches.
The default LDT is shared by all processes and generally not used, but if a process needs to create its own LDT, it may do so, and use that instead of the default.
The Pentium architecture provides 2 bits ( 4 values ) for protection in a segment selector, but Linux only uses two values: user mode and kernel mode.
Because Linux is designed to run on 64-bit as well as 32-bit architectures, it employs a three-level paging strategy as shown in Figure 8.24, where the number of bits in each portion of the address varies by architecture.
In the case of the Pentium architecture, the size of the middle directory portion is set to 0 bits, effectively bypassing the middle directory.
```
- TSS
```
how TSS be located by process

```
