- stack
```
Stack example on the X86 architecture
On the X86 architecture the stack grows downwards. The stack frames have a certain structure regarding the calling convention. The CDECL calling convention is the most widely used. It is most likely used by your compiler. Two registers are used:

ESP: Extended Stack Pointer. 32 bit value containing the top-of-stack address (more accurately the ``bottom-of-stack`` on X86!)
EBP: Extended Base Pointer. 32 bit value defining the current stack frame, when using CDECL calling convention. It points at the current local data. It can also access the routine parameters.
Take care when implementing your kernel. If you use segmentation, the DS segment should be configured to have its base at the same address as SS does. Otherwise you may run into problems when passing pointers to local variables into functions, because normal GPRs can't access the stack the way you might think.

Here is an example stack. The elements are 4 byte words in protected mode:

Memory address:   Stack elements:

               +----------------------------+
 0x105000      | Parameter 1 for routine 1  | \
               +----------------------------+  |
 0x104FFC      | First callers return addr. |   >  Stack frame 1
               +----------------------------+  |
 0x104FF8      | First callers EBP          | /
               +----------------------------+
 0x104FF4   +->| Parameter 2 for routine 2  | \  <-- Routine 1's EBP
            |  +----------------------------+  |
 0x104FF0   |  | Parameter 1 for routine 2  |  |
            |  +----------------------------+  |
 0x104FEC   |  | Return address, routine 1  |  |
            |  +----------------------------+  |
 0x104FE8   +--| EBP value for routine 1    |   >  Stack frame 2
               +----------------------------+  |
 0x104FE4   +->| Local data                 |  | <-- Routine 2's EBP
            |  +----------------------------+  |
 0x104FE0   |  | Local data                 |  |
            |  +----------------------------+  |
 0x104FDC   |  | Local data                 | /
            |  +----------------------------+
 0x104FD8   |  | Parameter 1 for routine 3  | \
            |  +----------------------------+  |
 0x104FD4   |  | Return address, routine 2  |  |
            |  +----------------------------+   >  Stack frame 3
 0x104FD0   +--| EBP value for routine 2    |  |
               +----------------------------+  |
 0x104FCC   +->| Local data                 | /  <-- Routine 3's EBP
            |  +----------------------------+
 0x104FC8   |  | Return address, routine 3  | \
            |  +----------------------------+  |
 0x104FC4   +--| EBP value for routine 3    |  |
               +----------------------------+   >  Stack frame 4
 0x104FC0      | Local data                 |  | <-- Current EBP
               +----------------------------+  |
 0x104FBC      | Local data                 | /
               +----------------------------+
 0x104FB8      |                            |    <-- Current ESP
                \/\/\/\/\/\/\/\/\/\/\/\/\/\/
The CDECL calling convention is described here:

Caller's responsibilities
Push parameters in reverse order (last parameter pushed first)
Perform the call
Pop the parameters, use them, or simply increment ESP to remove them (stack clearing)
The return value is stored in EAX
Callee's responsibilities (callee is the routine being called)
Store caller's EBP on the stack
Save current ESP in EBP
Code, storing local data on the stack
For a fast exit load the old ESP from EBP, else pop local data elements
Pop the old EBP and return – store return value in EAX
It looks like this in assembly (NASM):

SECTION .text

caller:
    
    ; ...
    
    ; Caller responsibilities:
    PUSH  3         ; push the parameters in reverse order
    PUSH  2
    CALL  callee    ; perform the call
    ADD   ESP, 8    ; stack cleaning (remove the 2 words)
    
    ; ... Use the return value in EAX ...
    
    
callee:
    
    ; Callee responsibilities:
    PUSH  EBP       ; store caller's EBP
    MOV   EBP, ESP  ; save current stack pointer in EBP
    
    ; ... Code, store return value in EAX ...
    
    ; Callee responsibilities:
    MOV   ESP, EBP  ; remove an unknown number of local data elements
    POP   EBP       ; restore caller's EBP
    RET             ; return
The GCC compiler does all this automatically, but if you have to call C/C++ methods from assembly or reverse, you have to know the convention. Now look at one stack frame (the callee's):

+-------------------------+
| Parameter 3             |
+-------------------------+
| Parameter 2             |
+-------------------------+
| Parameter 1             |
+-------------------------+
| Caller's return address |
+-------------------------+
| Caller's EBP value      |
+-------------------------+
| Local variable          | <-- Current EBP
+-------------------------+
| Local variable          |
+-------------------------+
| Local variable          |
+-------------------------+
| Temporary data          |
+-------------------------+
| Temporary data          |
+-------------------------+
|                         | <-- Current ESP
+-------------------------+
```
- https://wiki.osdev.org/Stack
