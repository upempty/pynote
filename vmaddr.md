
- logical address(seg cs/ds seg always 0 nowadays with some other properties)->linear address->physical address
```
nowadays process uses virtual address via linear address directly. Segmentation not as popular today, but paging used.
X86 page translation: virtual page + offset  <-TLB-> physical page + offset.
```
![image](https://github.com/upempty/pynote/assets/52414719/6c8fcf2f-35b5-458e-9366-b4f330ac7b5b)


![image](https://github.com/upempty/pynote/assets/52414719/3c1baaae-7f36-488e-b2f6-cefc9a5cee6b)

![image](https://github.com/upempty/pynote/assets/52414719/6ee9e6db-f79d-4edf-9ae2-4ba6eb1d9677)


- CR3 why physical: qucik access for page enties, physical page frame number stored in page table. And seperate CR3 for CPU cores.

- Context switch to another process
![image](https://github.com/upempty/pynote/assets/52414719/77ac58c2-dad1-4cc8-bdcf-fd69dba35578)

- TLB (translation lookaside buffer)
```
TLB is a cache.
TLB only maintains the mapping for the pages what are accessible to the current process,
then process switch will flush the TLB, But clearing TLB is slow, overhead when switching the process.
Many moden processors have optimizations that reduce the overhead.
Intel: TLBs have a process contxt ID(PCID), so each TLB entry is tagged with PCID of the process that the page mapping is for.
Arm: domain field used for that.
TLB will be updated when first page fault, page replacemement(when full), context switch, page table update etc. And if page hit, it will query from TLB directly.
```
