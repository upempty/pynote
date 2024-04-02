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
how TSS be located by process.

During task switching, the processor first stores the context of current task in the current TSS, then loads the task register of the new task,
then loads the context of the new task from the new TSS with the new task's descriptor in GDT, finally executes the new task.

```

```
Let's take a look at the code, this is the TSS structure we are going to use,

06/include/task.h

struct TSS_STRUCT {

        unsigned int back_link;

        unsigned int esp0, ss0;

        unsigned int esp1, ss1;

        unsigned int esp2, ss2;

        unsigned int cr3;

        unsigned int eip;

        unsigned int eflags;

        unsigned int eax,ecx,edx,ebx;

        unsigned int esp, ebp;

        unsigned int esi, edi;

        unsigned int es, cs, ss, ds, fs, gs;

        unsigned int ldt;

        unsigned int trace_bitmap;

};

It is exactly the same as the TSS structure I mentioned above, but because this structure is going to be used by the processor, so it has to be 104 bytes long and no paddings in it, if you use IA-64 or other arches or other compilers, check documents youself.

For a working OS, those information are not enough, so anothing structure TASK_STRUCT is used to wrap TSS structure.

#define TS_RUNNING      0

#define TS_RUNABLE      1

#define TS_STOPPED      2

 

struct TASK_STRUCT {

        struct TSS_STRUCT tss;

        unsigned long long tss_entry;

        unsigned long long ldt[2];

        unsigned long long ldt_entry;

        int state;

        int priority;

        struct TASK_STRUCT *next;

};

 

#define DEFAULT_LDT_CODE        0x00cffa000000ffffULL

#define DEFAULT_LDT_DATA        0x00cff2000000ffffULL

 

#define INITIAL_PRIO            200

This is all we need at this moment, those TS_* stuff defines all states that a process can be at. The tss_entry in TASK_STRUCT defines the descriptor of the tss field in this structure, I am going to explain it in a short while. Two *ldt* fields will be explained later on. state field store the current state of this process, which is one of those TS_*. priority indicates the sequence of the execution of processes in system, and the new task will be given a initial priority INITIAL_PRIO. All tasks in Skelix are managed as a link, the next field defines the next task in the link.

Now let's look at an example, this is the TASK_STRUCT structure for task 0, that is the first task in system, when kernel finishes all initialization, it becomes task 0.

static unsigned long TASK0_STACK[256] = {0xf};

This is the stack used as the CPL0 stack for task 0. Whitout that 0xF, I encountered a problem during my compiling, if it is just initialized like static unsigned long TASK0_STACK[256]; then this memory area just vanishes (Shen Feng explains, without 0xF, TASK0_STACK[256] would be an zero-initialized static data, and it would be in .bss section, and the object file only record its length and start address etc.. That's why a nonzero value has to be used to keep it in .data section.)

struct TASK_STRUCT TASK0 = {

        /* tss */

        {       /* back_link */

                0,

                /* esp0                                    ss0 */

                (unsigned)&TASK0_STACK+sizeof TASK0_STACK, DATA_SEL, 

Make esp0 points to the "bottom" of the stack. DATA_SEL and CODE_SEL appears in next few lines are defined at 06/include/kernel.h, they are selectors of data and code segments in GDT.

                /* esp1 ss1 esp2 ss2 */

                0, 0, 0, 0, 

                /* cr3 */

                0, 

                /* eip eflags */

                0, 0, 

                /* eax ecx edx ebx */

                0, 0, 0, 0,

                /* esp ebp */

                0, 0,

                /* esi edi */

                0, 0, 

                /* es          cs             ds */

                USER_DATA_SEL, USER_CODE_SEL, USER_DATA_SEL, 

                /* ss          fs             gs */

                USER_DATA_SEL, USER_DATA_SEL, USER_DATA_SEL, 

                /* ldt */

                0x20,

                /* trace_bitmap */

                0x00000000},

                /* tss_entry */

                0,

                /* idt[2] */

                {DEFAULT_LDT_CODE, DEFAULT_LDT_DATA},

                /* idt_entry */

                0,

                /* state */

                TS_RUNNING,

                /* priority */

                INITIAL_PRIO,

                /* next */

                0,

};

Now we have the TSS, we are going to create a TSS descriptor, remember there are two reserved place in GDT.

06/bootsect.s

gdt:   

        .quad    0x0000000000000000 # null descriptor

        .quad    0x00cf9a000000ffff # cs

        .quad    0x00cf92000000ffff # ds

        .quad    0x0000000000000000 # reserved for tss

        .quad    0x0000000000000000 # reserved for ldt

The forth entry(0x3) are reserved for TSS of current running task, so a macro CURR_TASK_TSS = 3 has been defined as an index refers to this position in GDT. We are going to let the current task uses this place, once it gives up the control, it saves its descriptor in its tss_entry of TASK_STRUCT. When a new task takes over the control, it loads its own TSS descriptor from its TASK_STRUCT to this place. In this way, we can allow unlimited tasks works in system. Because there is a length limit about GDT, it can only have 8096 descriptors, actually, Linux has the limit of tasks for quite long time, I don't know why it has this limit, because eliminating this limit seems to be quite simple.

unsigned long long 

set_tss(unsigned long long tss) {

        unsigned long long tss_entry = 0x0080890000000067ULL;

        tss_entry |= ((tss)<<16) & 0xffffff0000ULL;

        tss_entry |= ((tss)<<32) & 0xff00000000000000ULL;

        return gdt[CURR_TASK_TSS] = tss_entry;

}

set_tss generates the TSS descriptor and put it into GDT, we can see it set the DPL of descriptor to 0, so only kernel can use this descriptor.

unsigned long long

get_tss(void) {

        return gdt[CURR_TASK_TSS];

}
```
