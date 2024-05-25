
- pci driver, uio driver(need to explictly modprobe), igb_uio driver(need to explictly insmod), user space api based on pci resource sys dev file (e.g. /sys/.../resource0) to access the tx/rx desc's addr's buffer for tx/rx data
  - uio driver in linux native
  - igb_uio kernel mode driver/module: https://github.com/gaojinghua/igb_uio/tree/master
    - a) user mode driver: https://github.com/echoechoin/e1000-igb_uio/tree/maste
    - b) dpdk driver
    - either a) or b) can be used for user space tx/rx data bypass kernel stack!!!  

https://codilime.com/blog/how-can-dpdk-access-devices-from-user-space/

how dpdk associated with igb_uio or vfio-pci: rte_eal_init->rte_bus_scan ....->pci_uio_map_resource https://zhuanlan.zhihu.com/p/587858559


![image](https://github.com/upempty/pynote/assets/52414719/bbde5a6e-921a-44dd-91b2-72a9b3d32685)


![image](https://github.com/upempty/pynote/assets/52414719/e84be08f-bb66-4a69-b0b5-8cf437153caa)

![image](https://github.com/upempty/pynote/assets/52414719/789b0cd9-f499-4abd-9eda-2ef2287725b1)


```
bootloader will set the pci configuration space with predefined values of hw.
so it knows the address of pci address tables(bars).

from the bars' resource, to get the resource0, get the whole start addr which can be called address of hw based: which will start all hw info example rx desc(start addr), desc number,
which can be ptr to mem of allocated page. inside desc, it will have addr which ptr to the data buffer.



igbuio_pci_probe(struct pci_dev *dev, const struct pci_device_id *id)-------------------
{
...

	/* register uio driver */
	err = uio_register_device(&dev->dev, &udev->info);


static struct pci_driver igbuio_pci_driver = {------------------------
	.name = "igb_uio",
	.id_table = NULL,
	.probe = igbuio_pci_probe,-------------------------!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!drv->probe(dev) called inside of module install which has register driver....
	.remove = igbuio_pci_remove,
};

static int __init
igbuio_pci_init_module(void)
{
	int ret;

	ret = igbuio_config_intr_mode(intr_mode);
	if (ret < 0)
		return ret;

	ret = register_chrdev(MAJOR_NO/* MAJOR */,
			      DEV_NAME /*NAME*/,
			      &igb_net_fops);
	if (ret < 0) {
		printk(KERN_ERR "register_chrdev failed\n");
		return ret;
	}

	return pci_register_driver(&igbuio_pci_driver)---------------------->driver_register(&drv->driver);-->driver_probe_device(drv, dev);
--ret = __driver_probe_device(drv, dev);-->
-->ret = really_probe(dev, drv)-->---ret = call_driver_probe(dev, drv);

	if (dev->bus->probe)
		ret = dev->bus->probe(dev);
	else if (drv->probe)
		ret = drv->probe(dev);----------------------------------------------------igbuio_pci_probe--------------------will exposed the pci conf spaces' base address(resource0/1/2...or bars..'s addr) to user.
=====================================================================================so that user can use this exposed(ether driver api or .../sys/..../resource mmap's memory) descriptors inside to handler tx/rx data.






https://wiki.osdev.org/PCI_Express
https://github.com/bisdn/dpdk-dev/blob/master/lib/librte_eal/linuxapp/igb_uio/igb_uio.c
 pci_create_resource_files


https://elixir.bootlin.com/linux/latest/source/drivers/pci/pci-sysfs.c#L1511------------------what is content comes from???
!!!!!!!!!!!pci memory to usersapce?? https://github.com/billfarrow/pcimem


bash# lspci -v
0001:00:07.0 Class 0680: Unknown device bec0:0001 (rev 01)
    Flags: bus master, 66MHz, medium devsel, latency 128, IRQ 31
    Memory at 8d010000 (32-bit, non-prefetchable) [size=4K]
    Memory at 8d000000 (32-bit, non-prefetchable) [size=64K]
    Memory at 8c000000 (32-bit, non-prefetchable) [size=16M]

This PCI card makes 3 seperate regions of memory available to the host
computer. The sysfs resource0 file corresponds to the first memory region. The
PCI card lets the host computer know about these memory regions using the BAR
registers in the PCI config.

== mmap() ==

These sysfs resource can be used with mmap() to map the PCI memory into a
userspace applications memory space.  The application then has a pointer to the
start of the PCI memory region and can read and write values directly. (There
is a bit more going on here with respect to memory pointers, but that is all
taken care of by the kernel).

fd = open("/sys/devices/pci0001\:00/0001\:00\:07.0/resource0", O_RDWR | O_SYNC);
ptr = mmap(0, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
printf("PCI BAR0 0x0000 = 0x%4x\n",  *((unsigned short *) ptr);




int __must_check pci_create_sysfs_dev_files(struct pci_dev *pdev)
{
	if (!sysfs_initialized)
		return -EACCES;

	return pci_create_resource_files(pdev);-------------------
}
https://elixir.bootlin.com/linux/latest/source/fs/kernfs/file.c#L1020


struct kernfs_node *__kernfs_create_file(struct kernfs_node *parent,
					 const char *name,
					 umode_t mode, kuid_t uid, kgid_t gid,
					 loff_t size,
					 const struct kernfs_ops *ops,
					 void *priv, const void *ns,
					 struct lock_class_key *key)
{
	struct kernfs_node *kn;
	unsigned flags;
	int rc;

	flags = KERNFS_FILE;

	kn = kernfs_new_node(parent, name, (mode & S_IALLUGO) | S_IFREG,
			     uid, gid, flags);
	if (!kn)
		return ERR_PTR(-ENOMEM);

	kn->attr.ops = ops;
	kn->attr.size = size;
	kn->ns = ns;
	kn->priv = priv;



how pci/uio device files created?
binded will create? 
>>./utils/dpdk-devbind.py --bind=igb_uio <网卡2的pci_id>
https://github.com/echoechoin/e1000-igb_uio/blob/master/utils/dpdk-devbind.py

# list of supported DPDK drivers
dpdk_drivers = ["igb_uio", "vfio-pci", "uio_pci_generic"]
    if driver in dpdk_drivers:
        filename = "/sys/bus/pci/devices/%s/driver_override" % dev_id   == inside the file it writes igb_uio
        f = open(filename, "w")
        f.write("%s" % driver)


//////sysfs, usually mounted at /sys, provides access to PCI resources on platforms that support it//////

!!!!!!!!!!!!!!!!!!/sys/bus/pci/devices/%s/resource0 pci_id----pci_create_sysfs_dev_files!!!!!!!!!
>> sysfs filesystem's ops to create file and directory inode info with content, so it's specific header which is for system device based. (sysfs_super_info,, etc) .name		= "sysfs"
>> sysfs_fs_type  sysfs_inode_init ,   inode->i_fop = &sysfs_file_operations;
谁写的呢？ pci_write_config


static void __pci_set_master(struct pci_dev *dev, bool enable)
{
	u16 old_cmd, cmd;

	pci_read_config_word(dev, PCI_COMMAND, &old_cmd);--------------
	if (enable)
		cmd = old_cmd | PCI_COMMAND_MASTER;
	else
		cmd = old_cmd & ~PCI_COMMAND_MASTER;
	if (cmd != old_cmd) {
		pci_dbg(dev, "%s bus mastering\n",
			enable ? "enabling" : "disabling");
		pci_write_config_word(dev, PCI_COMMAND, cmd);
	}
	dev->is_busmaster = enable;
}

addr = pci_resource_start(dev, pci_bar);--get how where to get those info?
==#define pci_resource_start(dev, bar)	((dev)->resource[(bar)].start)

pci_dev resource: region allocated/reserved.
struct resource {
	resource_size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
	unsigned long desc;
	struct resource *parent, *sibling, *child;
};



/**
 * __pci_request_region - Reserved PCI I/O and memory resource
 * @pdev: PCI device whose resources are to be reserved
 * @bar: BAR to be reserved
 * @res_name: Name to be associated with resource.
 * @exclusive: whether the region access is exclusive or not
 *
 * Mark the PCI region associated with PCI device @pdev BAR @bar as
 * being reserved by owner @res_name.  Do not access any
 * address inside the PCI regions unless this call returns
 * successfully.
 *
 * If @exclusive is set, then the region is marked so that userspace
 * is explicitly not allowed to map the resource via /dev/mem or
 * sysfs MMIO access.
 *
 * Returns 0 on success, or %EBUSY on error.  A warning
 * message is also printed on failure.
 */
static int __pci_request_region(struct pci_dev *pdev, int bar,
				const char *res_name, int exclusive)
{
	struct pci_devres *dr;

	if (pci_resource_len(pdev, bar) == 0)
		return 0;

	if (pci_resource_flags(pdev, bar) & IORESOURCE_IO) {
		if (!request_region(pci_resource_start(pdev, bar),
			    pci_resource_len(pdev, bar), res_name))
			goto err_out;
	} else if (pci_resource_flags(pdev, bar) & IORESOURCE_MEM) {
		if (!__request_mem_region(pci_resource_start(pdev, bar),
					pci_resource_len(pdev, bar), res_name,
					exclusive))
			goto err_out;
	}

	dr = find_pci_dr(pdev);
	if (dr)
		dr->region_mask |= 1 << bar;

	return 0;



====return smp_load_acquire(&iomem_inode)->i_mapping;


static int pci_create_attr(struct pci_dev *pdev, int num, int write_combine)
{
	/* allocate attribute structure, piggyback attribute name */
	int name_len = write_combine ? 13 : 10;
	struct bin_attribute *res_attr;
	char *res_attr_name;
	int retval;

	res_attr = kzalloc(sizeof(*res_attr) + name_len, GFP_ATOMIC);
	if (!res_attr)
		return -ENOMEM;

	res_attr_name = (char *)(res_attr + 1);

	sysfs_bin_attr_init(res_attr);
	if (write_combine) {
		sprintf(res_attr_name, "resource%d_wc", num);
		res_attr->mmap = pci_mmap_resource_wc;
	} else {
		sprintf(res_attr_name, "resource%d", num);-------------------------------
		if (pci_resource_flags(pdev, num) & IORESOURCE_IO) {
			res_attr->read = pci_read_resource_io;
			res_attr->write = pci_write_resource_io;
			if (arch_can_pci_mmap_io())
				res_attr->mmap = pci_mmap_resource_uc;
		} else {
			res_attr->mmap = pci_mmap_resource_uc;-----------------------------return smp_load_acquire(&iomem_inode)->i_mapping;!!!!!!!!!!!!! how user space mmap from the specific fd, will use this mapping to memory.
		}
	}
	if (res_attr->mmap) {
		res_attr->f_mapping = iomem_get_mapping;
		/*
		 * generic_file_llseek() consults f_mapping->host to determine
		 * the file size. As iomem_inode knows nothing about the
		 * attribute, it's not going to work, so override it as well.
		 */
		res_attr->llseek = pci_llseek_resource;
	}
	res_attr->attr.name = res_attr_name;
	res_attr->attr.mode = 0600;
	res_attr->size = pci_resource_len(pdev, num);
	res_attr->private = (void *)(unsigned long)num;
	retval = sysfs_create_bin_file(&pdev->dev.kobj, res_attr);---------------------------------------


 pci_create_resource_files(struct pci_dev *pdev)




static int __init pci_sysfs_init(void)
{
	struct pci_dev *pdev = NULL;
	struct pci_bus *pbus = NULL;
	int retval;

	sysfs_initialized = 1;
	for_each_pci_dev(pdev) {
		retval = pci_create_sysfs_dev_files(pdev);
		if (retval) {
			pci_dev_put(pdev);
			return retval;
		}
	}





Summary

Each BAR is a 32-bit memory location that describes describes a memory region (base address + width) that your CPU can use to talk to a PCIe device.

When you read or write to offsets within the memory regions specified by a BAR, TLP packets are sent back and forth between the CPU/memory and the PCIe device, which tells the PCIe device to do something or send something back.

Such reads and writes are the main way in which drivers interact with PCIe devices.

What reads and writes to specific addresses mean is defined by each specific PCIe device and completely device dependant, but typically:

reads return status information such as:
what the device is currently doing, or how much work it has done so far
how the device has been configured
writes:
configure how the device should operate

tell the device to start doing some work, e.g. write to disk, render a frame on the GPU, or send a packet over the network.

A very common pattern in which such operations happen is:

CPU specifies where the input data is in RAM, and were the output should go to RAM
device reads input data from main memory via DMA. Again, more TLP packes.
device does some work
device writes output data back to main memory via DMA
device sends an interrupt to tell the CPU it finished its work
Here's how the configuration spaces and BAR regions might look like on physical memory of a hypothetical device, with some of the BARs pointing to corresponding memory regions:

+--------------+
| Func 64:00.0 |
| Conf. Space  |
+--------------+
| BAR 0        |>--------------+
+--------------+               |
| BAR 1        |>----------+   |
+--------------+           |   |
| BAR 2        |           |   |
+--------------+           |   |
| BAR 3        |           |   |
+--------------+           |   |
| BAR 4        |           |   |
+--------------+           |   |
| BAR 5        |           |   |
+--------------+           |   |
                           |   |
+--------------+           |   |
| Func 64:00.1 |           |   |
| Conf. Space  |           |   |
+--------------+           |   |
| BAR 0        |>------+   |   |
+--------------+       |   |   |
| BAR 1        |       |   |   |
+--------------+       |   |   |
| BAR 2        |>--+   |   |   |
+--------------+   |   |   |   |
| BAR 3        |   |   |   |   |
+--------------+   |   |   |   |
| BAR 4        |   |   |   |   |
+--------------+   |   |   |   |
| BAR 5        |   |   |   |   |
+--------------+   |   |   |   |
                   |   |   |   |
                   |   |   |   |
                   |   |   |   |
+--------------+<--|---|---+   |
| Region of    |   |   |       |
| 64:00.0 BAR0 |   |   |       |
| (1 MiB)      |   |   |       |
+--------------+   |   |       |
                   |   |       |
+--------------+<--|---+       |
| Region of    |   |           |
| 64:00.1 BAR2 |   |           |
| (2 MiB)      |   |           |
+--------------+   |           |
                   |           |
+--------------+<--|-----------+
| Region of    |   |
| 64:00.0 BAR1 |   |
| (512 KiB)    |   |
+--------------+   |
                   |
+--------------+<--+
| Region of    |
| 64:00.1 BAR0 |
| (512 KiB)    |
+--------------+
Where BARs are located

Let's locate ourselves globally within the PCIe environment.

For each PCIe device, this is how things are logically organized hierarchically:

+--------+                  +------------+       +------+
| device |>---------------->| function 0 |>----->| BAR0 |
|        |                  |            |       +------+
|        |>------------+    |            |
|        |             |    |            |       +------+
   ...        ...      |    |            |>----->| BAR1 |
|        |             |    |            |       +------+
|        |>--------+   |    |            |
+--------+         |   |         ...        ...    ...
                   |   |    |            |
                   |   |    |            |       +------+
                   |   |    |            |>----->| BAR5 |
                   |   |    +------------+       +------+
                   |   |
                   |   |
                   |   |    +------------+       +------+
                   |   +--->| function 1 |>----->| BAR0 |
                   |        |            |       +------+
                   |        |            |
                   |        |            |       +------+
                   |        |            |>----->| BAR1 |
                   |        |            |       +------+
                   |        |            |
                   |             ...        ...    ...
                   |        |            |
                   |        |            |       +------+
                   |        |            |>----->| BAR5 |
                   |        +------------+       +------+
                   |
                   |
                   |             ...
                   |
                   |
                   |        +------------+       +------+
                   +------->| function 7 |>----->| BAR0 |
                            |            |       +------+
                            |            |
                            |            |       +------+
                            |            |>----->| BAR1 |
                            |            |       +------+
                            |            |
                                 ...        ...    ...
                            |            |
                            |            |       +------+
                            |            |>----->| BAR5 |
                            +------------+       +------+

Each PCIe device can define from 1 to 8 "functions". Each function can be identified by the 2-byte "bus-device-function" (BDF) triplet:

bus: 8 bits (max 256), which bus the PCIe device is connected to
device: 5 bits (max 32 per bus), which device of the bus
function: 3 bits (max 8 per device), which function of the device
These three values are often represented in the format

<bus>:<device>.<function>
e.g. doing lspci on my Lenovo ThinkPad P14s laptop gives among other lines:

00:00.0 Host bridge: Advanced Micro Devices, Inc. [AMD] Device 14e8
00:00.2 IOMMU: Advanced Micro Devices, Inc. [AMD] Device 14e9
00:01.0 Host bridge: Advanced Micro Devices, Inc. [AMD] Device 14ea
00:02.0 Host bridge: Advanced Micro Devices, Inc. [AMD] Device 14ea
00:02.1 PCI bridge: Advanced Micro Devices, Inc. [AMD] Device 14ee
00:02.2 PCI bridge: Advanced Micro Devices, Inc. [AMD] Device 14ee
00:02.4 PCI bridge: Advanced Micro Devices, Inc. [AMD] Device 14ee
64:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix1 (rev dd)
64:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Rembrandt Radeon High Definition Audio Controller
64:00.2 Encryption controller: Advanced Micro Devices, Inc. [AMD] Family 19h (Model 74h) CCP/PSP 3.0 Device
64:00.3 USB controller: Advanced Micro Devices, Inc. [AMD] Device 15b9
64:00.4 USB controller: Advanced Micro Devices, Inc. [AMD] Device 15ba
64:00.5 Multimedia controller: Advanced Micro Devices, Inc. [AMD] ACP/ACP3X/ACP6x Audio Coprocessor (rev 63)
65:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Device 14ec
65:00.1 Signal processing controller: Advanced Micro Devices, Inc. [AMD] Device 1502
so e.g. here we see that

bus 00 contains several devices such as:
00:00, which contains two functions:
00:00.0
00:00.2
00:01 which has just one mandatory function 00:01.0
00:02 which has four functions:
00:02.0
00:02.1
00:02.2
00:02.4
and so on for bus 64.

Each function has one "PCIe configuration space", which a standardized small chunk of memory used to get information or do certain operations common to all devices.

For PCI Type 0 (Non-Bridge) devices, the configuration space looks like this:

0                                  15 16                                 31
+------------------+-----------------+------------------+------------------+
| Vendor ID                          | Device ID                           | 00
+------------------+-----------------+------------------+------------------+
| Command                            | Status                              | 04
+------------------+-----------------+------------------+------------------+
| Revision ID                        | Class Code                          | 08
+------------------+-----------------+------------------+------------------+
| Cache Line Size  | Latency Timer   | Header Type      | BIST             | 0C
+------------------+-----------------+------------------+------------------+
| BAR0                                                                     | 10
+------------------+-----------------+------------------+------------------+
| BAR1                                                                     | 14
+------------------+-----------------+------------------+------------------+
| BAR2                                                                     | 18
+------------------+-----------------+------------------+------------------+
| BAR3                                                                     | 1C
+------------------+-----------------+------------------+------------------+
| BAR4                                                                     | 20
+------------------+-----------------+------------------+------------------+
| BAR5                                                                     | 24
+------------------+-----------------+------------------+------------------+
| Cardbus CIS Pointer                                                      | 28
+------------------+-----------------+------------------+------------------+
| Subsystem Vendor ID                | Subsystem ID                        | 2C
+------------------+-----------------+------------------+------------------+
| Expansion ROM Base Address                                               | 30
+------------------+-----------------+------------------+------------------+
| Cap. Pointer     |                                                       | 34
+------------------+-----------------+------------------+------------------+
|                                                                          | 38
+------------------+-----------------+------------------+------------------+
| Interrupt Line   | Interrupt Pin   | Min Gnt.         | Max Lat.         | 3C
+------------------+-----------------+------------------+------------------+
Adapted from: https://en.wikipedia.org/wiki/File:Pci-config-space.svg




/* fs/sysfs/file.c */
const struct file_operations sysfs_file_operations = { //文本文件
	.read		= sysfs_read_file,
	.write		= sysfs_write_file,----------------------
	.llseek		= generic_file_llseek,
	.open		= sysfs_open_file,
	.release	= sysfs_release,
	.poll		= sysfs_poll,
};
 
/* fs/sysfs/bin.c */
const struct file_operations bin_fops = { //二进制文件
	.read		= read,
	.write		= write,----------------------------------------------
	.mmap		= mmap,---------------
	.llseek		= generic_file_llseek,
	.open		= open,
	.release	= release,
};

switch (sysfs_type(sd)) { //struct sysfs_dirent类型
	case SYSFS_DIR: //目录
		inode->i_op = &sysfs_dir_inode_operations;
		inode->i_fop = &sysfs_dir_operations;
		break;
	case SYSFS_KOBJ_ATTR: //文本文件
		inode->i_size = PAGE_SIZE; //文件大小固定为一页内存
		inode->i_fop = &sysfs_file_operations;
		break;
	case SYSFS_KOBJ_BIN_ATTR: //二进制文件
		bin_attr = sd->s_bin_attr.bin_attr;
		inode->i_size = bin_attr->size;
		inode->i_fop = &bin_fops;----------------------------------------------
		break;
	case SYSFS_KOBJ_LINK: //符号链接文件
		inode->i_op = &sysfs_symlink_inode_operations;
		break;


b->legacy_io->write = pci_write_legacy_io;
	/* See pci_create_attr() for motivation */
	b->legacy_io->llseek = pci_llseek_resource;
	b->legacy_io->mmap = pci_mmap_legacy_io;


#define BIN_ATTR(_name, _mode, _read, _write, _size)			\
struct bin_attribute bin_attr_##_name = __BIN_ATTR(_name, _mode, _read,	\
					_write, _size)
/* macros to create static binary attributes easier */
#define __BIN_ATTR(_name, _mode, _read, _write, _size) {		\
	.attr = { .name = __stringify(_name), .mode = _mode },		\
	.read	= _read,						\
	.write	= _write,						\
	.size	= _size,						\
}



static BIN_ATTR(config, 0644, pci_read_config, pci_write_config, 0);-------------------

pci_write_config->
pci_user_write_config_word

You can find it in drivers/pci/access.c:

    /* Returns 0 on success, negative values indicate error. */
#define PCI_USER_WRITE_CONFIG(size, type)                               \
int pci_user_write_config_##size                                        \
        (struct pci_dev *dev, int pos, type val)                        \
{                                                                       \
        int ret = PCIBIOS_SUCCESSFUL;                                   \
        if (PCI_##size##_BAD)                                           \
                return -EINVAL;                                         \
        raw_spin_lock_irq(&pci_lock);                           \
        if (unlikely(dev->block_cfg_access))                            \
                pci_wait_cfg(dev);                                      \
        ret = dev->bus->ops->write(dev->bus, dev->devfn,                \
                                        pos, sizeof(type), val);        \-----------------------------------
        raw_spin_unlock_irq(&pci_lock);                         \
        return pcibios_err_to_errno(ret);                               \
}                                                                       \
EXPORT_SYMBOL_GPL(pci_user_write_config_##size);


==pci driver: pci_create_sysfs_dev_files->pci_create_resource_files->pci_create_attr-->retval (inside also res_attr->write = pci_write_resource_io;) = sysfs_create_bin_file(&pdev->dev.kobj, res_attr);

pci_create_resource_files(pdev);-<-- pci_create_sysfs_dev_files(struct pci_dev *pdev)--<-- pci_create_sysfs_dev_files(struct pci_dev *pdev)--<--pci_iov_add_virtfn(struct pci_dev *dev, int id)
---------------------------------------------<---sriov_add_vfs(struct pci_dev *dev, u16 num_vfs)---<---pci_enable_sriov(struct pci_dev *dev, int nr_virtfn)--<--
--ret = pci_enable_sriov(pdev, nr_virtfn); in https://github.com/torvalds/linux/blob/master/drivers/vfio/pci/vfio_pci_core.c#L2393
----int vfio_pci_core_sriov_configure(struct vfio_pci_core_device *vdev, int nr_virtfn)---vfio_pci_sriov_configure
static struct pci_driver vfio_pci_driver = {
	.name			= "vfio-pci",
	.id_table		= vfio_pci_table,
	.probe			= vfio_pci_probe,
	.remove			= vfio_pci_remove,
	.sriov_configure	= vfio_pci_sriov_configure,


in module_init(igbuio_pci_init_module):
----pci_register_driver(&igbuio_pci_driver);

static struct pci_driver igbuio_pci_driver = {
	.name = "igb_uio",
	.id_table = NULL,
	.probe = igbuio_pci_probe,
	.remove = igbuio_pci_remove,
};


#define pci_resource_start(dev, bar)	(pci_resource_n(dev, bar)->start)


==pci-sysfs.c(consumer of pci_mmap_page_range)
creates the /sys/bus/pci/devices/B:D:F/resource[_wc]X file based
on BAR flag(Prefetchable or non-Prefetchable) and expose to userspace.

!!!!!!!!!!!!!!!!!/dev/uio0--------------uio driver when register uio device: ret = dev_set_name(&idev->dev, "uio%d", idev->minor);-----uio_fd (read data) possible used in case of e1000_intr_listen thread created/used.
!!!!!!!!!!!!!!!!!/sys/class/uio/uio0/device/config---config_fd




////
下载编译igb_uio驱动，然后加载驱动：

git clone git://dpdk.org/dpdk-kmods
cd dpdk-kmods/linux/igb_uio && make
modprobe uio----https://github.com/torvalds/linux/blob/c760b3725e52403dc1b28644fb09c47a83cacea6/drivers/uio/uio.c#L1075
insmod ./igb_uio.ko----https://github.com/bisdn/dpdk-dev/blob/master/lib/librte_eal/linuxapp/igb_uio/igb_uio.c
将网卡2绑定到igb_uio驱动上：

./utils/dpdk-devbind.py --bind=igb_uio <网卡2的pci_id>
////





==uio code in user mode:
bypass kernel to send/recv packet directly via the pci dev's descriptors of tx and rx.

https://github.com/echoechoin/e1000-igb_uio/blob/master/src/e1000.c#L217


https://github.com/echoechoin/e1000-igb_uio/blob/master/src/e1000.c#L217

/**
 * @brief 通过pci_id获取e1000设备
 * 
 * @param pci_id: 0000:02:02.0 
 */
struct e1000_device *e1000_device_get(const char *pci_id)
{
    char path[1024] = {0};
    int resource_fd = -1;
    int uio_fd = -1;
    int config_fd = -1;
    struct e1000_device *dev = NULL;
    if (!is_intel_82545EM(pci_id)) // 只支持intel 82545EM 也就是VMware的默认网卡
        goto error;

    dev = (struct e1000_device *)malloc(sizeof(struct e1000_device));
    if (!dev)
        goto error;

    snprintf(dev->name, PCI_PRI_STR_SIZE, "%s", pci_id);
    snprintf(path, sizeof(path), "/sys/bus/pci/devices/%s/resource0", pci_id);
    resource_fd = open(path, O_RDWR);
    if (resource_fd < 0)
        goto error;

    dev->hw_addr = mmap(NULL, 0x1ffff, PROT_READ | PROT_WRITE, MAP_SHARED, resource_fd, 0);
    if (dev->hw_addr == MAP_FAILED)
        goto error;

    snprintf(path, sizeof(path), "/dev/uio0");
    uio_fd = open(path, O_RDWR);
    if (uio_fd < 0)
        goto error;

    snprintf(path, sizeof(path), "/sys/class/uio/uio0/device/config");
    config_fd = open(path, O_RDWR);
    if (config_fd < 0)
        goto error;

    dev->uio_fd = uio_fd;
    dev->config_fd = config_fd;
    return dev;



===============

void __init pcibios_irq_init(void)
{
	DBG(KERN_DEBUG "PCI: IRQ init\n");

x86: void pcibios_scan_root(int busnum)

struct pci_bus *pci_scan_root_bus(struct device *parent, int bus,
		struct pci_ops *ops, void *sysdata, struct list_head *resources)

struct pci_bus *pci_scan_root_bus_msi(struct device *parent, int bus,
		struct pci_ops *ops, void *sysdata,
		struct list_head *resources, struct msi_controller *msi)

unsigned int pci_scan_child_bus(struct pci_bus *bus)

int pci_scan_slot(struct pci_bus *bus, int devfn)

struct pci_dev *pci_scan_single_device(struct pci_bus *bus, int devfn)


static struct pci_dev *pci_scan_device(struct pci_bus *bus, int devfn)

struct pci_dev *pci_alloc_dev(struct pci_bus *bus)


tatic struct device_attribute sriov_numvfs_attr =
		__ATTR(sriov_numvfs, (S_IRUGO|S_IWUSR|S_IWGRP),
		       sriov_numvfs_show, sriov_numvfs_store);

static ssize_t sriov_numvfs_store(struct device *dev,
				  struct device_attribute *attr,
				  const char *buf, size_t count)
{
	struct pci_dev *pdev = to_pci_dev(dev);
	int ret;
	u16 num_vfs;

	ret = kstrtou16(buf, 0, &num_vfs);
	if (ret < 0)
		return ret;

	if (num_vfs > pci_sriov_get_totalvfs(pdev))
		return -ERANGE;

	device_lock(&pdev->dev);

	if (num_vfs == pdev->sriov->num_VFs)
		goto exit;

	/* is PF driver loaded w/callback */
	if (!pdev->driver || !pdev->driver->sriov_configure) {
		pci_info(pdev, "Driver does not support SRIOV configuration via sysfs\n");
		ret = -ENOENT;
		goto exit;
	}

	if (num_vfs == 0) {
		/* disable VFs */
		ret = pdev->driver->sriov_configure(pdev, 0);


```
