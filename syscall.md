
```
send()->glibc(send)-> SEND_NR, int 0x80(system call) intrrupt
or
keyboard enter one key(a) interrupt->
-> IDT descriptor entry(0x80th) on LIDT
--> selector(GDT/LDT's x-th entry->GDT==kernel code base address, and limit.
--> offset, then offset + base address, then it is Destination code segment = syscall
ENTRY (syscall).
-save registers...
-.................
```
