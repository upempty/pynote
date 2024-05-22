

- multiple queues for tx/rx support per port(or per VF) in dpdk driver
```

rte_eth_dev_configure(portid, _nb_rx_queues, _nb_tx_queues...) used for 
create muliple queues per portid/vf: Tx queues and Rx queues.
usually one Tx queue and one Rx queue per port.


rte_eth_rx_burst(,NIC_RX_QUEUE,..) per each queue receiving.



RTE_LIBRTE_I40E_QUEUE_NUM_PER_VF ----max support in hw register!!!

queue-num-per-vf



Scheduling of multiple RX/TX queues on a single port
https://mails.dpdk.org/archives/users/2023-May/007148.html

> As for RX, the answer might be clear.
> The NIC can only receive a packet once at a time, since the cable only
> outputs one signal (0 or 1) at a time (correct me if I'm wrong).
> Therefore the NIC can receive a packet, check it's information, and finally
> put in into the right queue via some policies, e.g. RSS, all sequentially.


Transmit scheduling is up to the hardware (not DPDK).
Generally I assume it is round-robin
but there maybe cases like priority queues (like DCB) or large packets
with segment offload.


rte_eth_dev_configure(portid, _nb_rx_queues, _nb_tx_queues
===https://elixir.free-electrons.com/dpdk/v17.11-rc2/source/lib/librte_ether/rte_ethdev.c#L774
/*
	 * Setup new number of RX/TX queues and reconfigure device.
	 */
	diag = rte_eth_dev_rx_queue_config(dev, nb_rx_q);
       		dev->data->rx_queues = rte_zmalloc("ethdev->rx_queues",-------------------------
				sizeof(dev->data->rx_queues[0]) * nb_queues,
				RTE_CACHE_LINE_SIZE);
		if (dev->data->rx_queues == NULL) {


                dev->data->nb_rx_queues = nb_queues;------------------



static int
i40evf_configure_vsi_queues(struct rte_eth_dev *dev)
{
	struct i40e_vf *vf = I40EVF_DEV_PRIVATE_TO_VF(dev->data->dev_private);
	struct i40e_rx_queue **rxq =
		(struct i40e_rx_queue **)dev->data->rx_queues;

	struct i40e_rx_queue **rxq =
		(struct i40e_rx_queue **)dev->data->rx_queues;--------------------------



		i40evf_fill_virtchnl_vsi_rxq_info(&vc_qpi->rxq,------------------------------------virtual channels
			vc_vqci->vsi_id, i, dev->data->nb_rx_queues,
					vf->max_pkt_len, rxq[i]);
	}

	for (i = 0, vc_qpi = vc_vqci->qpair; i < nb_qp; i++, vc_qpi++) {
		i40evf_fill_virtchnl_vsi_txq_info(&vc_qpi->txq,
			vc_vqci->vsi_id, i, dev->data->nb_tx_queues, txq[i]);------per queue per vc_qpi used.
		i40evf_fill_virtchnl_vsi_rxq_info(&vc_qpi->rxq,
			vc_vqci->vsi_id, i, dev->data->nb_rx_queues,
					vf->max_pkt_len, rxq[i]);------------------




static void
i40evf_fill_virtchnl_vsi_rxq_info(struct virtchnl_rxq_info *rxq_info,------------------------per vc-qpi
				  uint16_t vsi_id,
				  uint16_t queue_id,
				  uint16_t nb_rxq,
				  uint32_t max_pkt_size,
				  struct i40e_rx_queue *rxq)
{
	rxq_info->vsi_id = vsi_id;--------------------------------
	rxq_info->queue_id = queue_id;----------------------------------
	rxq_info->max_pkt_size = max_pkt_size;
	if (queue_id < nb_rxq) {
		rxq_info->ring_len = rxq->nb_rx_desc;
		rxq_info->dma_ring_addr = rxq->rx_ring_phys_addr;--------------------
		rxq_info->databuffer_size =
			(rte_pktmbuf_data_room_size(rxq->mp) -
				RTE_PKTMBUF_HEADROOM);
	}
}






int
i40e_dev_rx_queue_setup(struct rte_eth_dev *dev,
			uint16_t queue_idx,
			uint16_t nb_desc,
			unsigned int socket_id,
			const struct rte_eth_rxconf *rx_conf,
			struct rte_mempool *mp)
{
	struct i40e_hw *hw = I40E_DEV_PRIVATE_TO_HW(dev->data->dev_private);
	struct i40e_adapter *ad =
		I40E_DEV_PRIVATE_TO_ADAPTER(dev->data->dev_private);
	struct i40e_vsi *vsi;
	struct i40e_pf *pf = NULL;
	struct i40e_vf *vf = NULL;
	struct i40e_rx_queue *rxq;
	const struct rte_memzone *rz;
	uint32_t ring_size;
	uint16_t len, i;
	uint16_t reg_idx, base, bsf, tc_mapping;
	int q_offset, use_def_burst_func = 1;

	if (hw->mac.type == I40E_MAC_VF || hw->mac.type == I40E_MAC_X722_VF) {
		vf = I40EVF_DEV_PRIVATE_TO_VF(dev->data->dev_private);
		vsi = &vf->vsi;
		if (!vsi)
			return -EINVAL;
		reg_idx = queue_idx;
	} else {
		pf = I40E_DEV_PRIVATE_TO_PF(dev->data->dev_private);
		vsi = i40e_pf_get_vsi_by_qindex(pf, queue_idx);
		if (!vsi)
			return -EINVAL;
		q_offset = i40e_get_queue_offset_by_qindex(pf, queue_idx);
		if (q_offset < 0)
			return -EINVAL;
		reg_idx = vsi->base_queue + q_offset;
	}

	if (nb_desc % I40E_ALIGN_RING_DESC != 0 ||
	    (nb_desc > I40E_MAX_RING_DESC) ||
	    (nb_desc < I40E_MIN_RING_DESC)) {
		PMD_DRV_LOG(ERR, "Number (%u) of receive descriptors is "
			    "invalid", nb_desc);
		return -EINVAL;
	}

	/* Free memory if needed */
	if (dev->data->rx_queues[queue_idx]) {
		i40e_dev_rx_queue_release(dev->data->rx_queues[queue_idx]);
		dev->data->rx_queues[queue_idx] = NULL;
	}

	/* Allocate the rx queue data structure */
	rxq = rte_zmalloc_socket("i40e rx queue",
				 sizeof(struct i40e_rx_queue),
				 RTE_CACHE_LINE_SIZE,
				 socket_id);
	if (!rxq) {
		PMD_DRV_LOG(ERR, "Failed to allocate memory for "
			    "rx queue data structure");
		return -ENOMEM;
	}
	rxq->mp = mp;
	rxq->nb_rx_desc = nb_desc;
	rxq->rx_free_thresh = rx_conf->rx_free_thresh;
	rxq->queue_id = queue_idx;
	rxq->reg_idx = reg_idx;
	rxq->port_id = dev->data->port_id;
	rxq->crc_len = (uint8_t) ((dev->data->dev_conf.rxmode.hw_strip_crc) ?
							0 : ETHER_CRC_LEN);
	rxq->drop_en = rx_conf->rx_drop_en;
	rxq->vsi = vsi;
	rxq->rx_deferred_start = rx_conf->rx_deferred_start;

	/* Allocate the maximun number of RX ring hardware descriptor. */
	len = I40E_MAX_RING_DESC;

	/**
	 * Allocating a little more memory because vectorized/bulk_alloc Rx
	 * functions doesn't check boundaries each time.
	 */
	len += RTE_PMD_I40E_RX_MAX_BURST;

	ring_size = RTE_ALIGN(len * sizeof(union i40e_rx_desc),
			      I40E_DMA_MEM_ALIGN);

	rz = rte_eth_dma_zone_reserve(dev, "rx_ring", queue_idx,
			      ring_size, I40E_RING_BASE_ALIGN, socket_id);---------------------------------------
	if (!rz) {
		i40e_dev_rx_queue_release(rxq);
		PMD_DRV_LOG(ERR, "Failed to reserve DMA memory for RX");
		return -ENOMEM;
	}

	/* Zero all the descriptors in the ring. */
	memset(rz->addr, 0, ring_size);

	rxq->rx_ring_phys_addr = rz->phys_addr;---------------------------------------------------






	/* check and configure queue intr-vector mapping */
	if ((rte_intr_cap_multiple(intr_handle) ||
	     !RTE_ETH_DEV_SRIOV(dev).active) &&
	    dev->data->dev_conf.intr_conf.rxq != 0) {
		intr_vector = dev->data->nb_rx_queues;------------------------------------------------
		if (rte_intr_efd_enable(intr_handle, intr_vector))--------------------------

or


static void
eth_igb_configure_msix_intr(struct rte_eth_dev *dev)
{



		tmpval = (dev->data->nb_rx_queues | E1000_IVAR_VALID) << 8;---------------nb_rx_queues 
		E1000_WRITE_REG(hw, E1000_IVAR_MISC, tmpval);

https://elixir.free-electrons.com/dpdk/v17.11-rc2/source/drivers/net/i40e/i40e_pf.c#L688

	/* Then, it's Linux VF driver */------------
	qbit_max = 1 << pf->vf_nb_qp_max;
	for (i = 0; i < irqmap->num_vectors; i++) {
		map = &irqmap->vecmap[i];

		vector_id = map->vector_id;
		/* validate msg params */
		if (vector_id >= hw->func_caps.num_msix_vectors_vf) {
			ret = I40E_ERR_PARAM;
			goto send_msg;
		}

		if ((map->rxq_map < qbit_max) && (map->txq_map < qbit_max)) {
			i40e_pf_config_irq_link_list(vf, map);
		} else {
			/* configured queue size excceed limit */
			ret = I40E_ERR_PARAM;
			goto send_msg;
		}



queue-num-per-vf=4
./<build_target>/app/dpdk-testpmd -c f -n 4 -a 18:00.0,queue-num-per-vf=4 \
--file-prefix=test1 --socket-mem 1024,1024 -- -i


Both kernel driver, I40E and DPDK PMD driver, igb_uio/vfio-pci support VF request queue number at runtime, 
that means the users could configure the VF queue number at runtime.

Snice DPDK 19.02, VF is able to request max to 16 queues and the PF EAL parameter ‘queue-num-per-vf’ is redefined
 as the number of reserved queue per VF. For example, if the PCI address of an i40e PF is aaaa:bb.cc, with the EAL parameter
 -a aaaa:bb.cc,queue-num-per-vf=8, the number of reserved queue per VF created from this PF is 8.


Now RTE_LIBRTE_I40E_QUEUE_NUM_PER_VF is used to determine the max queue number per VF. 
It’s not friendly to the users because it means the users must decide the max queue 
number when compiling. There’s no chance to change it when deploying their APP. 
It’s good to make the queue number to be configurable so the users can change it 
when launching the APP. This requirement is meaningless to ixgbe since the queue is fixed on ixgbe. 
The number of queues per i40e VF can be determinated during run time. For example, 
if the PCI address of an i40e PF is aaaa:bb.cc, 
with the EAL parameter -w aaaa:bb.cc,queue-num-per-vf=8, the number of queues per VF created from this PF is 8.



uint16_t
eth_igb_recv_pkts(void *rx_queue, struct rte_mbuf **rx_pkts,
	       uint16_t nb_pkts)
{
	struct igb_rx_queue *rxq;
	volatile union e1000_adv_rx_desc *rx_ring;
	volatile union e1000_adv_rx_desc *rxdp;
	struct igb_rx_entry *sw_ring;
	struct igb_rx_entry *rxe;
	struct rte_mbuf *rxm;
	struct rte_mbuf *nmb;
	union e1000_adv_rx_desc rxd;
	uint64_t dma_addr;
	uint32_t staterr;
	uint32_t hlen_type_rss;
	uint16_t pkt_len;
	uint16_t rx_id;
	uint16_t nb_rx;
	uint16_t nb_hold;
	uint64_t pkt_flags;

	nb_rx = 0;
	nb_hold = 0;
	rxq = rx_queue;
	rx_id = rxq->rx_tail;
	rx_ring = rxq->rx_ring;
	sw_ring = rxq->sw_ring;
	while (nb_rx < nb_pkts) {
		/*
		 * The order of operations here is important as the DD status
		 * bit must not be read after any other descriptor fields.
		 * rx_ring and rxdp are pointing to volatile data so the order
		 * of accesses cannot be reordered by the compiler. If they were
		 * not volatile, they could be reordered which could lead to
		 * using invalid descriptor fields when read from rxd.
		 */
		rxdp = &rx_ring[rx_id];
		staterr = rxdp->wb.upper.status_error;
		if (! (staterr & rte_cpu_to_le_32(E1000_RXD_STAT_DD)))
			break;
		rxd = *rxdp;

		/*
		 * End of packet.
		 *
		 * If the E1000_RXD_STAT_EOP flag is not set, the RX packet is
		 * likely to be invalid and to be dropped by the various
		 * validation checks performed by the network stack.
		 *
		 * Allocate a new mbuf to replenish the RX ring descriptor.
		 * If the allocation fails:
		 *    - arrange for that RX descriptor to be the first one
		 *      being parsed the next time the receive function is
		 *      invoked [on the same queue].
		 *
		 *    - Stop parsing the RX ring and return immediately.
		 *
		 * This policy do not drop the packet received in the RX
		 * descriptor for which the allocation of a new mbuf failed.
		 * Thus, it allows that packet to be later retrieved if
		 * mbuf have been freed in the mean time.
		 * As a side effect, holding RX descriptors instead of
		 * systematically giving them back to the NIC may lead to
		 * RX ring exhaustion situations.
		 * However, the NIC can gracefully prevent such situations
		 * to happen by sending specific "back-pressure" flow control
		 * frames to its peer(s).
		 */
		PMD_RX_LOG(DEBUG, "port_id=%u queue_id=%u rx_id=%u "
			   "staterr=0x%x pkt_len=%u",
			   (unsigned) rxq->port_id, (unsigned) rxq->queue_id,
			   (unsigned) rx_id, (unsigned) staterr,
			   (unsigned) rte_le_to_cpu_16(rxd.wb.upper.length));

		nmb = rte_mbuf_raw_alloc(rxq->mb_pool);
		if (nmb == NULL) {
			PMD_RX_LOG(DEBUG, "RX mbuf alloc failed port_id=%u "
				   "queue_id=%u", (unsigned) rxq->port_id,
				   (unsigned) rxq->queue_id);
			rte_eth_devices[rxq->port_id].data->rx_mbuf_alloc_failed++;
			break;
		}

		nb_hold++;
		rxe = &sw_ring[rx_id];
		rx_id++;
		if (rx_id == rxq->nb_rx_desc)
			rx_id = 0;

		/* Prefetch next mbuf while processing current one. */
		rte_igb_prefetch(sw_ring[rx_id].mbuf);

		/*
		 * When next RX descriptor is on a cache-line boundary,
		 * prefetch the next 4 RX descriptors and the next 8 pointers
		 * to mbufs.
		 */
		if ((rx_id & 0x3) == 0) {
			rte_igb_prefetch(&rx_ring[rx_id]);
			rte_igb_prefetch(&sw_ring[rx_id]);
		}

		rxm = rxe->mbuf;
		rxe->mbuf = nmb;
		dma_addr =
			rte_cpu_to_le_64(rte_mbuf_data_dma_addr_default(nmb));
		rxdp->read.hdr_addr = 0;
		rxdp->read.pkt_addr = dma_addr;

		/*
		 * Initialize the returned mbuf.
		 * 1) setup generic mbuf fields:
		 *    - number of segments,
		 *    - next segment,
		 *    - packet length,
		 *    - RX port identifier.
		 * 2) integrate hardware offload data, if any:
		 *    - RSS flag & hash,
		 *    - IP checksum flag,
		 *    - VLAN TCI, if any,
		 *    - error flags.
		 */
		pkt_len = (uint16_t) (rte_le_to_cpu_16(rxd.wb.upper.length) -
				      rxq->crc_len);
		rxm->data_off = RTE_PKTMBUF_HEADROOM;
		rte_packet_prefetch((char *)rxm->buf_addr + rxm->data_off);
		rxm->nb_segs = 1;
		rxm->next = NULL;
		rxm->pkt_len = pkt_len;
		rxm->data_len = pkt_len;
		rxm->port = rxq->port_id;

		rxm->hash.rss = rxd.wb.lower.hi_dword.rss;
		hlen_type_rss = rte_le_to_cpu_32(rxd.wb.lower.lo_dword.data);

		/*
		 * The vlan_tci field is only valid when PKT_RX_VLAN is
		 * set in the pkt_flags field and must be in CPU byte order.
		 */
		if ((staterr & rte_cpu_to_le_32(E1000_RXDEXT_STATERR_LB)) &&
				(rxq->flags & IGB_RXQ_FLAG_LB_BSWAP_VLAN)) {
			rxm->vlan_tci = rte_be_to_cpu_16(rxd.wb.upper.vlan);
		} else {
			rxm->vlan_tci = rte_le_to_cpu_16(rxd.wb.upper.vlan);
		}
		pkt_flags = rx_desc_hlen_type_rss_to_pkt_flags(rxq, hlen_type_rss);
		pkt_flags = pkt_flags | rx_desc_status_to_pkt_flags(staterr);
		pkt_flags = pkt_flags | rx_desc_error_to_pkt_flags(staterr);
		rxm->ol_flags = pkt_flags;
		rxm->packet_type = igb_rxd_pkt_info_to_pkt_type(rxd.wb.lower.
						lo_dword.hs_rss.pkt_info);

		/*
		 * Store the mbuf address into the next entry of the array
		 * of returned packets.
		 */
		rx_pkts[nb_rx++] = rxm;
	}
	rxq->rx_tail = rx_id;

	/*
	 * If the number of free RX descriptors is greater than the RX free
	 * threshold of the queue, advance the Receive Descriptor Tail (RDT)
	 * register.
	 * Update the RDT with the value of the last processed RX descriptor
	 * minus 1, to guarantee that the RDT register is never equal to the
	 * RDH register, which creates a "full" ring situtation from the
	 * hardware point of view...
	 */
	nb_hold = (uint16_t) (nb_hold + rxq->nb_rx_hold);
	if (nb_hold > rxq->rx_free_thresh) {
		PMD_RX_LOG(DEBUG, "port_id=%u queue_id=%u rx_tail=%u "
			   "nb_hold=%u nb_rx=%u",
			   (unsigned) rxq->port_id, (unsigned) rxq->queue_id,
			   (unsigned) rx_id, (unsigned) nb_hold,
			   (unsigned) nb_rx);
		rx_id = (uint16_t) ((rx_id == 0) ?
				     (rxq->nb_rx_desc - 1) : (rx_id - 1));
		E1000_PCI_REG_WRITE(rxq->rdt_reg_addr, rx_id);
		nb_hold = 0;
	}
	rxq->nb_rx_hold = nb_hold;
	return nb_rx;
}




int
eth_igb_tx_queue_setup(struct rte_eth_dev *dev,
			 uint16_t queue_idx,



```
  
