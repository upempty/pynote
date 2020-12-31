## how CPU works
```
while(1):
    fetch instruction from memory pointed by PC;
    increament of PC;
    execute the instruction;
     if jmp instruction:
         write the jump's address to PC;
     else:
         execute nominal instruction;
```
