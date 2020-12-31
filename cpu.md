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
