
```
=sw
send()->glibc(send)-> SEND_NR, int 0x80(system call)/swi/swc 0 intrrupt.
=hardware
CPU response: save pc, registers, find 0x80 IDT entry:get previlleage ,
has authorize to trigger this interrupt, switch to kernel mode, provide kernel stack via TSS.
ISR address assign to PC of CPU.
--> IDT descriptor entry(0x80th) on LIDT
--> selector(GDT/LDT's x-th entry->GDT==kernel code base address, and limit.
--> offset, then offset + base address, then it is Destination code segment = syscall
=sw
then jump to OS code of ISR syscall entry.

ENTRY (syscall).
-save registers if possible.
-.................

syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
		ret = __invoke_syscall(regs, syscall_fn);

sys_recv = SYSCALL_DEFINE4(recv,..)
__sys_recv---

-iret
.................
```
