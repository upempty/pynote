
-todo
  http://skelix.net/skelixos/index_en.html
```
gcc, qemu(vmare)
bootsttrap for 'hellworld'
protected mode
print func
interrupt/exception/syscall
syscall/executing program

multitasking
filesystem
memory management
```

```
Memory management
Process scheduling
Interprocess Communication (IPC)
Virtual files
Networking
```
- https://www.mimuw.edu.pl/~lichota/07-08/SO2/01_Tour/tour.html
```
Linux source code tour
Version: 2.6.17.13
Source tree
root of Linux source code tree
arch/ directories - contain architecture-specific source files
include/ directory - contains public includes
include/asm-xxx/ directories - contain architecture-specific include files and inline code
Documentation/ directory and scripts/ directory - auxiliary things
other directories contain code for various subsystems 
Functional areas
User process management (CPU scheduling, task accounting, forking, etc.) - kernel/ directory
IPC services for processes (pipes, shared memory, etc.) - ipc/ directory
Virtual memory management - mm/ directory
Low-level hardware support (buses, interrupts, CPUs, hotplug, driver model, power management) - drivers/base/ directory, kernel/irq/ directory, arch/i386/kernel/cpu/ directory, kernel/power/ directory 
Device drivers - drivers/ directory, sound/ directory
Networking (sockets handling, network stacks) - net/ directory
Disks, block devices and volume management - block/ directory, drivers/block/ directory, drivers/md/ directory, files in fs/ directory
Character devices - drivers/char/ directory
Filesystems -  files in fs/ directory (generic layer) and subdirectories with specific filesystems implementation
Security - security/ directory
Time management - kernel/ directory
Locks, compression, cryptography and other library-like services for kernel code -  lib/ directory, crypto/ directory
User process management
struct task_struct -  contains all information about process (or thread), e.g.:
pid
relationships (parent/child, group, ...)
thread information
security privileges (effective user id, SELinux context, etc.)
file descriptors
process virtual memory description (struct mm_struct)
scheduling information (state, slice used, sleep time, allowed CPUs, etc.)
process state information (signals, tracing, etc.)
CPU context (struct thread_struct)
...
struct files_struct, struct fdtable - file descriptors table structures
struct file - description of file usage by process
Virtual memory management
Process memory management:
struct mm_struct - process virtual memory description, shared between threads (as they share virtual memory)
struct vm_area_struct *mmap - list of VM areas
pgd_t *pgd - physical page table address
struct vm_area_struct - description of one virtual memory area (i.e. contigous space with the same handler, mmapped file, code area, stack, etc.)
struct vm_operations_struct - operations on VM area (handling page faults, memory protection, etc.)
Pages management:
struct page - describes one physical page of memory
struct address_space *mapping - description for pages mmapped from file
pgoff_t index - index in "mapping"
struct mmu_gather - TLB context
struct pte_t - page table entry (physical, CPU specific)
struct pmd_t - 2-level page directory
struct pgd_t - 3-level page directory
physical level page manipulation functions (i386)
Swap management:
struct swp_entry_t - description of page which is in swap (in struct page)
struct swap_info_struct - description of one swap area
Page cache - caching of file contents in memory, shared area of VM and VFS:
struct pgoff_t - index in page cache file mapping
struct address_space - describes mapping of a file to page cache
struct address_space_operations - operations which can be performed on file pages mapped to page cache
Low-level hardware support
PnP support
Driver model (and sysfs)
Power management
Buses
PCI
USB
PCMCIA
...
IRQ management
DMA management
CPU hotplug
Physical memory
...
Driver model documentation
To see driver model tree - "find /sys"

Driver model:
struct bus_type - bus representation
struct device - device representation
struct bus_type * bus - bus on which the device exists
struct device_driver *driver - which driver has allocated this device
struct dev_pm_info power - power state information
struct device_driver - driver
int     (*remove)       (struct device * dev) - remove device
void    (*shutdown)     (struct device * dev) - turn off device
int     (*suspend)      (struct device * dev, pm_message_t state) - put device into low-power mode
int     (*resume)       (struct device * dev) - wake up device from low-power mode
struct resource - resource representation
Power management:
Power management definitions
struct dev_pm_info - device power state
Specific buses structures:
struct pci_bus - PCI bus structure, contains operations specific for PCI bus
struct pci_dev - PCI device specific representation
struct  device dev - driver model device
struct pci_bus  *bus - PCI bus where this device belongs
struct usb_bus - similar for USB bus
...

IRQ management:
request_irq() - assigns action to IRQ and enables it
struct irqaction - defines action run when IRQ happens
struct irqaction *next - next in chain of IRQ (for shared IRQs)
handler - function to be called
Networking
sockets handling
network protocols
IP
IPX
NetBEUI
...
physical network stacks
Ethernet
PPP
...
packet scheduling
encapsulation
...

Sockets:
struct sock - representation of network socket
struct sk_buff *head - first socket buffer for this socket
struct inet_sock - representation of internet socket type
struct sock sk - generic socket part
struct inet_connection_sock - connection-oriented socket
struct inet_sock icsk_inet - internet socket part
struct tcp_sock - TCP socket
struct sk_buff - buffer for packet
struct sock *sk - socket
struct net_device *dev - network device
struct tcphdr *th - TCP header
struct iphdr *iph - IP header
Physical network devices:
struct net_device - physical network interface description and operations
Disks, block devices and volume management
Disk devices handling (I/O requests, scheduling)
Block devices
Physical disks and arrays
Virtual block devices (DM, MD, LVM, multipath, NBD)
Partition management
Logical volumes management
Page cache integration
Structures:
struct block_device - block device information
struct gendisk *bd_disk
struct gendisk - physical disk representation
struct hd_struct **part - partitions
struct block_device_operations *fops - disk operations
struct request_queue *queue - I/O requests queue
struct block_device_operations - operations performed on block device (open, ioctl, media change, get geometry)
struct hd_struct - partition information
struct request_queue - queue of I/O requests to disk device
elevator_t *elevator - I/O scheduling algorithm
requests queue operations (insert, merge, flush)
struct request - represents I/O request
struct bio *bio - head of bios queue for this request
unsigned short ioprio - I/O priority
elevator fields
struct bio - basic unit of I/O on physical level, can hold scatter/gather information, allows merging of requests on physical operation level or processing it in hardware, if it is capable
struct elevator_queue - description of I/O scheduler
elevator_ops - operations of I/O scheduler
Page cache integration:
struct buffer_head - representation of disk block in page cache
struct page *b_page - page where this block is mapped
sector_t b_blocknr - block number on disk
struct block_device *b_bdev - block device of disk
struct backing_dev_info - description and operations of deviced used to hold pages
Character devices
struct file_operations - operations on file which represents character device
Filesystems
Generic VFS layer
Inodes handling 
Directory handling
File handling
File caching and readahead (integration with page cache)
Specific filesystems implementations
Structures:
struct file - representation of file (in process)
const struct file_operations *f_op - operations on file
struct dentry *f_dentry - directory entry
struct vfsmount *f_vfsmnt -  mount point of filesystem where file resides
struct file_ra_state f_ra - file readahead state
struct address_space *f_mapping - description of caching file contents on backing device (see VM)
struct file_operations - generic interface of operations on file
struct dentry - representation of directory entry (i.e. name)
struct inode *d_inode - inode
struct dentry_operations *d_op - operations
struct dentry_operations - operations on file name
struct inode - inode representation in memory
struct inode_operations *i_op - inode operations
struct block_device *i_bdev - block device
struct inode_operations - operations on inode
create file, mkdir, lookup, rename, unlink, setattr
struct vfsmount - filesystem mount description
struct super_block *mnt_sb
struct super_block - description of filesystem
struct super_operations *s_op
struct super_operations - operations on filesystem
alloc_inode
read_inode
write_inode
statfs
write_super
struct file_system_type - registered filesystem type
Other important files
Documentation/ directory - documentation for kernel (interfaces documentation, implementation documentation, submissions procedures, etc.)
Coding style guide
Patch submission process
scripts/ directory - useful scripts for kernel development (patches, generating necessary files, config stuff)
Makefile files
Kbuild files - kernel build configuration files
Further reading
Linux device drivers book (online)
Linux MM docs
Daniel P. Bovet; Marco Cesati - Understanding the Linux Kernel, 3rd Edition
```
