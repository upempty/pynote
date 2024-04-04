## how CPU works
```
PC (EIP/IP)
while(1):
    fetch instruction from memory pointed by PC;
    increament of PC;
    execute the instruction;
     if jmp instruction:
         write the jump's address to PC;
     elif: call instruction: possible nest call fun
         get_pc;
         --push next PC
         get_pc:xxxxx
         ret;
         --pop the next PC to be execute
     else:
         execute nominal instruction;
```
```
while True:
    # exec current instruction
    read EIP's code instruction to instruction buffer;
    increase the EIP;
    execute the introcution in instruction buffer;

    #exam interrupts from interrupt request register
    check for intrrupts()
    if there is:
      save current state;
      interrupt = get highest prio interrupt;
      handle interrupt(interrup)
      restore saved state;

    # move to next instruction
```
- https://wiki.osdev.org/CPU_Registers_x86-64
![image](https://github.com/upempty/pynote/assets/52414719/7e57ae59-6b50-4890-8138-cb84add9eb11)
![image](https://github.com/upempty/pynote/assets/52414719/d2579f58-ea9c-4a81-beb0-87ddb71e3830)

