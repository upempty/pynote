
buddy and slab
- buddy
![image](https://github.com/upempty/pynote/assets/52414719/e5b29e18-6fcd-44b6-91f8-ebbc8ca00e8d)

- slab

  ![image](https://github.com/upempty/pynote/assets/52414719/e19651bc-0c63-4da5-8a24-e694ac94be3b)  


  ![image](https://github.com/upempty/pynote/assets/52414719/0d5149e2-bbd3-43a6-8fbb-c3c7d46e6e1e)



![image](https://github.com/upempty/pynote/assets/52414719/64f636ed-3115-41a0-ba0d-dec9fa4dc301)


standand memory allocation:
- malloc (glibc) /(calloc with clear 0), realloc(increase or decrease the mem size, realloc ptr address may change if no enough memory for increasing, so that move to new location of memory)
  - uses ptmalloc or other ways in userspace. maintain free list, similar as buddy system, but not buddy system.
  - maintain private heap and maintain freelist, if free list has no enough memory to use, then may to brk(sbrk) to increase heap memory size.
 
heap memory allocation:
- brk (sbrk to increase or decrease heap size)

memory mapping
- mmap/munmap
shared memory
- shmget/shmat
- paging: 
  - is mainly for simplify the mm, so that os can allocate or release page based, but not any size of memory block.  
  - memory protected: access right (read/write/exec) for each page?  
  - vm page converted to physical memory via MMU, TLB as cache to be used.  
  - when process needs memory, os will dispatch whole page, but not part of page to it. So one page will be whole allocated to one process, but not splited to multiple process.



kernel memory
- kmalloc (kzalloc with extra clear to 0)
  -  used in kenel, return vm, but physical address is continus, so usually to allocate small memory block, as it's easy to find the continus part of small.
  -  not need extra page table item, can directly mapping to kernel space, high efficiency. can e.g use virt_to_phys(v).
  -  as continus, the alloctated mem can be used for DMA usage. as some HW requires phys address continus.
- vmalloc/vfree
  - used in kernel, return vm, and virtual address continus, but physical not must be. it is suitable to allocate big memory.
  - need extra page table item, to mapping to kernel space, low efficiency. can not used for DMA probably.
 
user space memory allocation
  - posix_memalign()/aligned_alloc()
- slab
  - used in kernel
