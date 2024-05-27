
- userspace dpdk pmd driver

https://github.com/DPDK/dpdk/blob/main/drivers/net/e1000/base/e1000_regs.h#L73  
https://github.com/DPDK/dpdk/blob/main/drivers/net/e1000/igb_rxtx.c#L822  
https://github.com/DPDK/dpdk/blob/main/drivers/net/e1000/igb_ethdev.c#L1639  


```
userspace dpdk pmd driver: 
================================
dpdk/drivers/net/e1000/igb_rxtx.c
https://github.com/DPDK/dpdk/blob/77e63757d2a0106a0cc519f5a82e2d749d16475a/drivers/net/e1000/igb_rxtx.c#L2408
--DPDK is a set of libraries and optimized network interface card (NIC) drivers for fast packet processing in a user space.
--Drivers are special libraries which provide poll-mode driver implementations for devices:
  either hardware devices or pseudo/virtual devices. 
  They are contained in the drivers subdirectory, classified by type, 
  and each compiles to a library with the format librte_X_Y.a where X is the device class name and Y is the driver name.

==how userspace driver/api uses the igb_uio/vfio-pci driver,
it is to controll the nic/pci hw register which rely on the base addr
which is hw or hw_addr which was mmap from /sys/../reource0 created by igb_uio/vfio-pci driver!!!

#define E1000_PCI_REG_ADDR(hw, reg) \
	((volatile uint32_t *)((char *)(hw)->hw_addr + (reg)))

hw->hw_addr= (void *)pci_dev->mem_resource[0].addr; -----<---eth_igb_dev_init--<--
--<---eth_igb_pci_probe--<--
static struct rte_pci_driver rte_igb_pmd = {
	.id_table = pci_id_igb_map,
	.drv_flags = RTE_PCI_DRV_NEED_MAPPING | RTE_PCI_DRV_INTR_LSC,
	.probe = eth_igb_pci_probe,


1. init phase: tx/rx desc register (in hw space) written from userspace allocated memory.
   set in eth_igb_tx_init/eth_igb_rx_init, eth_igb_tx_queue_setup/eth_igb_rx_queue_setup.
   set callback: eth_igb_xmit_pkts/eth_igb_recv_pkts; which is also set in igb_ethdev.c for dev reset/probe case:
   
2. update Transmit Descriptror Tail(TDT)/RDT in tx xmit/recv:
	/*
	 * Set the Transmit Descriptor Tail (TDT).
	 */
    eth_igb_xmit_pkts: E1000_PCI_REG_WRITE_RELAXED(txq->tdt_reg_addr, tx_id);

    eth_igb_recv_pkts: E1000_PCI_REG_WRITE(rxq->rdt_reg_addr, rx_id);


==dpdk/drivers/net/e1000
/igb_rxtx.c
https://github.com/DPDK/dpdk/blob/77e63757d2a0106a0cc519f5a82e2d749d16475a/drivers/net/e1000/igb_rxtx.c#L2408


eth_igb_tx_init:
/* Setup the Base and Length of the Tx Descriptor Rings. */
		txq = dev->data->tx_queues[i];
		bus_addr = txq->tx_ring_phys_addr;

		E1000_WRITE_REG(hw, E1000_TDLEN(txq->reg_idx),
				txq->nb_tx_desc *
				sizeof(union e1000_adv_tx_desc));
		E1000_WRITE_REG(hw, E1000_TDBAH(txq->reg_idx),
				(uint32_t)(bus_addr >> 32));
		E1000_WRITE_REG(hw, E1000_TDBAL(txq->reg_idx), (uint32_t)bus_addr);
...
eth_igb_tx_queue_setup:

eth_igb_rx_init:
int
eth_igb_rx_init(struct rte_eth_dev *dev)
{

		bus_addr = rxq->rx_ring_phys_addr;
		E1000_WRITE_REG(hw, E1000_RDLEN(rxq->reg_idx),
				rxq->nb_rx_desc *
				sizeof(union e1000_adv_rx_desc));
		E1000_WRITE_REG(hw, E1000_RDBAH(rxq->reg_idx),
				(uint32_t)(bus_addr >> 32));
		E1000_WRITE_REG(hw, E1000_RDBAL(rxq->reg_idx), (uint32_t)bus_addr);
...
eth_igb_rx_queue_setup

================================

```


```

e1000's init: in dpdk/drivers/net/e1000
/igb_ethdev.c: https://github.com/DPDK/dpdk/blob/77e63757d2a0106a0cc519f5a82e2d749d16475a/drivers/net/e1000/igb_ethdev.c#L752


hw->hw_addr= (void *)pci_dev->mem_resource[0].addr;

pmd drivers: RTE_INIT(rte_virtio_net_pci_pmd_init) in dpdk/drivers/net/virtio
/virtio_pci_ethdev.c

https://github.com/DPDK/dpdk/blob/main/drivers/net/virtio/virtio_pci_ethdev.c#L230


/* map a particular resource from a file */
void *
pci_map_resource(void *requested_addr, int fd, off_t offset, size_t size,---------------new mmap-------------compared with legacy
		 int additional_flags)
{
	void *mapaddr;

	/* Map the PCI memory resource of device */
	mapaddr = rte_mem_map(requested_addr, size,
		RTE_PROT_READ | RTE_PROT_WRITE,
		RTE_MAP_SHARED | additional_flags, fd, offset);
	if (mapaddr == NULL) {
		RTE_LOG(ERR, EAL,
			"%s(): cannot map resource(%d, %p, 0x%zx, 0x%llx): %s (%p)\n",
			__func__, fd, requested_addr, size,
			(unsigned long long)offset,
			rte_strerror(rte_errno), mapaddr);
	} else
		RTE_LOG(DEBUG, EAL, "  PCI memory mapped at %p\n", mapaddr);

	return mapaddr;
}

-<-pci_uio_map_resource(struct rte_pci_device *dev)--<-rte_pci_map_device-<-rte_pci_map_device

dpdk/drivers/net/virtio
/virtio_pci_ethdev.c---https://github.com/DPDK/dpdk/blob/main/drivers/net/virtio/virtio_pci_ethdev.c#L59


static struct rte_pci_driver rte_virtio_net_pci_pmd = {
	.driver = {
		.name = "net_virtio",
	},
	.id_table = pci_id_virtio_map,
	.drv_flags = 0,
	.probe = eth_virtio_pci_probe,
	.remove = eth_virtio_pci_remove,
};



mmap() used in pci_uio_ioport_map in 
https://github.com/DPDK/dpdk/blob/main/drivers/bus/pci/linux/pci.c#L744


dpdk/drivers/net/virtio
/virtio_pci.c:
int vtpci_legacy_ioport_map(struct virtio_hw *hw)---------legacy------------------
{
	return rte_pci_ioport_map(VTPCI_DEV(hw), 0, VTPCI_IO(hw));
}



dpdk/drivers/net/virtio/virtio_pci_ethdev.c


for /sys/..../resource0:

 static struct rte_pci_driver rte_virtio_net_pci_pmd = {--------------------------------PMD------------------------
222  	.driver = {
223  		.name = "net_virtio",
224  	},
225  	.id_table = pci_id_virtio_map,
226  	.drv_flags = 0,
227  	.probe = eth_virtio_pci_probe,
228  	.remove = eth_virtio_pci_remove,
229  };



https://github.com/DPDK/dpdk/blob/main/drivers/bus/pci/linux/pci_uio.c



/* map the PCI resource of a PCI device in virtual memory */
int
pci_uio_map_resource(struct rte_pci_device *dev)

		ret = pci_uio_map_resource_by_index(dev, i,
				uio_res, map_idx);




/* map the PCI resource of a PCI device in virtual memory */
int
pci_uio_map_resource(struct rte_pci_device *dev)
{
	int i, map_idx = 0, ret;
	uint64_t phaddr;
	struct mapped_pci_resource *uio_res = NULL;
	struct mapped_pci_res_list *uio_res_list =
		RTE_TAILQ_CAST(rte_uio_tailq.head, mapped_pci_res_list);

	dev->intr_handle.fd = -1;
	dev->intr_handle.uio_cfg_fd = -1;
	dev->intr_handle.type = RTE_INTR_HANDLE_UNKNOWN;

	/* secondary processes - use already recorded details */
	if (rte_eal_process_type() != RTE_PROC_PRIMARY)
		return pci_uio_map_secondary(dev);

	/* allocate uio resource */
	ret = pci_uio_alloc_resource(dev, &uio_res);
	if (ret)
		return ret;

	/* Map all BARs */
	for (i = 0; i != PCI_MAX_RESOURCE; i++) {
		/* skip empty BAR */
		phaddr = dev->mem_resource[i].phys_addr;
		if (phaddr == 0)
			continue;

		ret = pci_uio_map_resource_by_index(dev, i,
				uio_res, map_idx);
		if (ret)
			goto error;

		map_idx++;
	}

	uio_res->nb_maps = map_idx;

	TAILQ_INSERT_TAIL(uio_res_list, uio_res, next);----------------------------------------------!!!!!!!!!!!!!!!

int
pci_uio_map_resource_by_index(struct rte_pci_device *dev, int res_idx,
		struct mapped_pci_resource *uio_res, int map_idx)-------------
{
	int fd;
	char *devname;
	void *mapaddr;
	uint64_t offset;
	uint64_t pagesz;
	struct pci_map *maps;

	maps = uio_res->maps;---------------------------
	devname = uio_res->path;


	/* if matching map is found, then use it */
	offset = res_idx * pagesz;
	mapaddr = pci_map_resource(NULL, fd, (off_t)offset,
			(size_t)dev->mem_resource[res_idx].len, 0);
	close(fd);
	if (mapaddr == MAP_FAILED)
		goto error;

	maps[map_idx].phaddr = dev->mem_resource[res_idx].phys_addr;
	maps[map_idx].size = dev->mem_resource[res_idx].len;
	maps[map_idx].addr = mapaddr;--------------------------------
	maps[map_idx].offset = offset;
	strcpy(maps[map_idx].path, devname);
	dev->mem_resource[res_idx].addr = mapaddr;-------------------------------------!!!!!!!!!!!!!!!!!!!!!! used by following






/* map a particular resource from a file */
void *
pci_map_resource(void *requested_addr, int fd, off_t offset, size_t size,
		 int additional_flags)
{
	void *mapaddr;

	/* Map the PCI memory resource of device */
	mapaddr = mmap(requested_addr, size, PROT_READ | PROT_WRITE,
			MAP_SHARED | additional_flags, fd, offset);



===used by following here coded:

1 usercase: https://github.com/ceph/dpdk/blob/eac901ce29be559b1bb5c5da33fe2bf5c0b4bfd6/drivers/net/e1000/em_ethdev.c#L316

static int
eth_em_dev_init(struct rte_eth_dev *eth_dev)
{
	struct rte_pci_device *pci_dev = E1000_DEV_TO_PCI(eth_dev);
	struct rte_intr_handle *intr_handle = &pci_dev->intr_handle;
	struct e1000_adapter *adapter =
		E1000_DEV_PRIVATE(eth_dev->data->dev_private);
	struct e1000_hw *hw =
		E1000_DEV_PRIVATE_TO_HW(eth_dev->data->dev_private);
	struct e1000_vfta * shadow_vfta =
		E1000_DEV_PRIVATE_TO_VFTA(eth_dev->data->dev_private);

	eth_dev->dev_ops = &eth_em_ops;
	eth_dev->rx_pkt_burst = (eth_rx_burst_t)&eth_em_recv_pkts;
	eth_dev->tx_pkt_burst = (eth_tx_burst_t)&eth_em_xmit_pkts;

	/* for secondary processes, we don't initialise any further as primary
	 * has already done this work. Only check we don't need a different
	 * RX function */
	if (rte_eal_process_type() != RTE_PROC_PRIMARY){
		if (eth_dev->data->scattered_rx)
			eth_dev->rx_pkt_burst =
				(eth_rx_burst_t)&eth_em_recv_scattered_pkts;
		return 0;
	}

	rte_eth_copy_pci_info(eth_dev, pci_dev);

	hw->hw_addr = (void *)pci_dev->mem_resource[0].addr;-----------------------
	hw->device_id = pci_dev->id.device_id;
	adapter->stopped = 0;


....


static int
eth_igb_dev_init(struct rte_eth_dev *eth_dev)
{
	int error = 0;
	struct rte_pci_device *pci_dev = E1000_DEV_TO_PCI(eth_dev);
	struct e1000_hw *hw =
		E1000_DEV_PRIVATE_TO_HW(eth_dev->data->dev_private);
	struct e1000_vfta * shadow_vfta =
		E1000_DEV_PRIVATE_TO_VFTA(eth_dev->data->dev_private);
	struct e1000_filter_info *filter_info =
		E1000_DEV_PRIVATE_TO_FILTER_INFO(eth_dev->data->dev_private);
	struct e1000_adapter *adapter =
		E1000_DEV_PRIVATE(eth_dev->data->dev_private);

	uint32_t ctrl_ext;

	eth_dev->dev_ops = &eth_igb_ops;
	eth_dev->rx_pkt_burst = &eth_igb_recv_pkts;
	eth_dev->tx_pkt_burst = &eth_igb_xmit_pkts;

	/* for secondary processes, we don't initialise any further as primary
	 * has already done this work. Only check we don't need a different
	 * RX function */
	if (rte_eal_process_type() != RTE_PROC_PRIMARY){
		if (eth_dev->data->scattered_rx)
			eth_dev->rx_pkt_burst = &eth_igb_recv_scattered_pkts;
		return 0;
	}

	rte_eth_copy_pci_info(eth_dev, pci_dev);

	hw->hw_addr= (void *)pci_dev->mem_resource[0].addr;--------------following code really!!!!!!!!!!!!


======================
----------------------continue for using hw and hw->hw_addr
----------------------which is based addres of pci configuraiton space, where it stored the tx/rx ring/descriptors' info!!!!

https://github.com/ceph/dpdk/blob/eac901ce29be559b1bb5c5da33fe2bf5c0b4bfd6/drivers/net/e1000/base/e1000_osdep.h#L112

#define E1000_PCI_REG_ADDR(hw, reg) \
	((volatile uint32_t *)((char *)(hw)->hw_addr + (reg)))

#define E1000_PCI_REG_ARRAY_ADDR(hw, reg, index) \
	E1000_PCI_REG_ADDR((hw), (reg) + ((index) << 2))


https://github.com/ceph/dpdk/blob/master/drivers/net/e1000/igb_rxtx.c#L1350
int
eth_igb_tx_queue_setup(struct rte_eth_dev *dev,
			 uint16_t queue_idx,
			 uint16_t nb_desc,
			 unsigned int socket_id,
			 const struct rte_eth_txconf *tx_conf)
{
	const struct rte_memzone *tz;
	struct igb_tx_queue *txq;
	struct e1000_hw     *hw;
	uint32_t size;

	hw = E1000_DEV_PRIVATE_TO_HW(dev->data->dev_private);

	/*
	 * Validate number of transmit descriptors.
	 * It must not exceed hardware maximum, and must be multiple
	 * of E1000_ALIGN.
	 */
	if (nb_desc % IGB_TXD_ALIGN != 0 ||
			(nb_desc > E1000_MAX_RING_DESC) ||
			(nb_desc < E1000_MIN_RING_DESC)) {
		return -EINVAL;
	}

	/*
	 * The tx_free_thresh and tx_rs_thresh values are not used in the 1G
	 * driver.
	 */
	if (tx_conf->tx_free_thresh != 0)
		PMD_INIT_LOG(INFO, "The tx_free_thresh parameter is not "
			     "used for the 1G driver.");
	if (tx_conf->tx_rs_thresh != 0)
		PMD_INIT_LOG(INFO, "The tx_rs_thresh parameter is not "
			     "used for the 1G driver.");
	if (tx_conf->tx_thresh.wthresh == 0 && hw->mac.type != e1000_82576)
		PMD_INIT_LOG(INFO, "To improve 1G driver performance, "
			     "consider setting the TX WTHRESH value to 4, 8, "
			     "or 16.");

	/* Free memory prior to re-allocation if needed */
	if (dev->data->tx_queues[queue_idx] != NULL) {
		igb_tx_queue_release(dev->data->tx_queues[queue_idx]);
		dev->data->tx_queues[queue_idx] = NULL;
	}

	/* First allocate the tx queue data structure */
	txq = rte_zmalloc("ethdev TX queue", sizeof(struct igb_tx_queue),
							RTE_CACHE_LINE_SIZE);
	if (txq == NULL)
		return -ENOMEM;

	/*
	 * Allocate TX ring hardware descriptors. A memzone large enough to
	 * handle the maximum ring size is allocated in order to allow for
	 * resizing in later calls to the queue setup function.
	 */
	size = sizeof(union e1000_adv_tx_desc) * E1000_MAX_RING_DESC;
	tz = rte_eth_dma_zone_reserve(dev, "tx_ring", queue_idx, size,
				      E1000_ALIGN, socket_id);
	if (tz == NULL) {
		igb_tx_queue_release(txq);
		return -ENOMEM;
	}

	txq->nb_tx_desc = nb_desc;
	txq->pthresh = tx_conf->tx_thresh.pthresh;
	txq->hthresh = tx_conf->tx_thresh.hthresh;
	txq->wthresh = tx_conf->tx_thresh.wthresh;
	if (txq->wthresh > 0 && hw->mac.type == e1000_82576)
		txq->wthresh = 1;
	txq->queue_id = queue_idx;
	txq->reg_idx = (uint16_t)((RTE_ETH_DEV_SRIOV(dev).active == 0) ?
		queue_idx : RTE_ETH_DEV_SRIOV(dev).def_pool_q_idx + queue_idx);
	txq->port_id = dev->data->port_id;

	txq->tdt_reg_addr = E1000_PCI_REG_ADDR(hw, E1000_TDT(txq->reg_idx));--------------------!!!!!!!!!!!!!!!!!!!!!!!!to check this memory used by userspace related!!!!!!!
https://github.com/ceph/dpdk/blob/master/drivers/net/e1000/igb_rxtx.c#L1350

	txq->tx_ring_phys_addr = rte_mem_phy2mch(tz->memseg_id, tz->phys_addr);


	txq->tx_ring = (union e1000_adv_tx_desc *) tz->addr;
	/* Allocate software ring */
	txq->sw_ring = rte_zmalloc("txq->sw_ring",
				   sizeof(struct igb_tx_entry) * nb_desc,
				   RTE_CACHE_LINE_SIZE);
	if (txq->sw_ring == NULL) {
		igb_tx_queue_release(txq);
		return -ENOMEM;
	}
	PMD_INIT_LOG(DEBUG, "sw_ring=%p hw_ring=%p dma_addr=0x%"PRIx64,
		     txq->sw_ring, txq->tx_ring, txq->tx_ring_phys_addr);

	igb_reset_tx_queue(txq, dev);
	dev->tx_pkt_burst = eth_igb_xmit_pkts;
	dev->data->tx_queues[queue_idx] = txq;

	return 0;
}



static inline uint16_t
tx_xmit_pkts(void *tx_queue, struct rte_mbuf **tx_pkts,
	     uint16_t nb_pkts)



	/* update tail pointer */
	rte_wmb();
	IXGBE_PCI_REG_WRITE(txq->tdt_reg_addr, txq->tx_tail);-------------------update the register's value!!!

also: 
https://github.com/ceph/dpdk/blob/master/drivers/net/ixgbe/ixgbe_rxtx.c#L2330

```
