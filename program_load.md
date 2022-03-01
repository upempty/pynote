
To determine the segment start addr: [linker](https://github.com/aosp-mirror/platform_bionic/blob/master/linker/linker_phdr.cpp)
```

Ref: https://github.com/aosp-mirror/platform_bionic/blob/master/linker/linker_phdr.cpp

 For example, consider the following list:
    [ offset:0,      filesz:0x4000, memsz:0x4000, vaddr:0x30000 ],
    [ offset:0x4000, filesz:0x2000, memsz:0x8000, vaddr:0x40000 ],
  This corresponds to two segments that cover these virtual address ranges:
       0x30000...0x34000
       0x40000...0x48000
  If the loader decides to load the first segment at address 0xa0000000
  then the segments' load address ranges will be:
       0xa0030000...0xa0034000
       0xa0040000...0xa0048000
  In other words, all segments must be loaded at an address that has the same
  constant offset from their p_vaddr value. This offset is computed as the
  difference between the first segment's load address, and its p_vaddr value.
  However, in practice, segments do _not_ start at page boundaries. Since we
  can only memory-map at page boundaries, this means that the bias is
  computed as:
       load_bias = phdr0_load_address - PAGE_START(phdr0->p_vaddr)
       
load_bias = phdr0_load_address - PAGE_START(phdr0->p_vaddr)



 // To get load_bias:
 bool ElfReader::ReserveAddressSpace(address_space_params* address_space)
 
 // To get seg start addr(virtual):
 The load_bias must be added to any p_vaddr value read from the ELF file to
  determine the corresponding memory address.
  
    And that the phdr0_load_address must start at a page boundary, with
  the segment's real content starting at:
       phdr0_load_address + PAGE_OFFSET(phdr0->p_vaddr)
       (ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;)
       
  ```
  
  
