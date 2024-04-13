- slab
used in kernel
- malloc (glibc)
uses ptmalloc or other ways in userspace. maintain free list, similar as buddy system, but not buddy system.
- paging:
  is mainly for simplify the mm, so that os can allocate or release page based, but not any size of memory block.  
  memory protected: access right (read/write/exec) for each page?
  vm page converted to physical memory via MMU, TLB as cache to be used.
