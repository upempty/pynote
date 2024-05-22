
- pcap dump
```

https://github.com/DPDK/dpdk/blob/6f80df8cb0f889203d7cd27766abcc6ebc720e33/app/dumpcap/main.c#L806

https://doc.dpdk.org/guides-16.07/prog_guide/pdump_lib.html?highlight=rte_pdump_init

init server  in rte_pdump_init for sample.
https://github.com/DPDK/dpdk/blob/6f80df8cb0f889203d7cd27766abcc6ebc720e33/app/test-pmd/testpmd.c#L4567       

pdump_server in rte_pdump_init
The library API rte_pdump_init(), initializes the packet capture framework by creating the pthread and the server socket.
 The server socket in the pthread context will be listening to the client requests to enable or disable the packet capture.


===pdump_enable request msg to send===

struct pdump_request {
	uint16_t ver;
	uint16_t op;
	uint32_t flags;
	char device[RTE_DEV_NAME_MAX_LEN];
	uint16_t queue;
	struct rte_ring *ring;---------------
	struct rte_mempool *mp;--------------

	const struct rte_bpf_prm *prm;
	uint32_t snaplen;
};


https://github.com/DPDK/dpdk/blob/6f80df8cb0f889203d7cd27766abcc6ebc720e33/app/dumpcap/main.c#L806

	r = create_ring();
	mp = create_mempool();
	out = create_output();

	start_time = time(NULL);
	enable_pdump(r, mp);--------------------------------------------

static void enable_pdump(struct rte_ring *r, struct rte_mempool *mp)-------------------------------------
{
	struct interface *intf;
	unsigned int count = 0;
	uint32_t flags;
	int ret;

	flags = RTE_PDUMP_FLAG_RXTX;
	if (use_pcapng)
		flags |= RTE_PDUMP_FLAG_PCAPNG;

	TAILQ_FOREACH(intf, &interfaces, next) {
		ret = rte_pdump_enable_bpf(intf->port, RTE_PDUMP_ALL_QUEUES,----------------------------
					   flags, intf->opts.snap_len,
					   r, mp, intf->bpf_prm);


int
rte_pdump_enable_bpf(uint16_t port, uint16_t queue,
		     uint32_t flags, uint32_t snaplen,
		     struct rte_ring *ring,
		     struct rte_mempool *mp,
		     const struct rte_bpf_prm *prm)
{
	return pdump_enable(port, queue, flags, snaplen,----------------------------
			    ring, mp, prm);
}


static int
pdump_enable(uint16_t port, uint16_t queue,
	     uint32_t flags, uint32_t snaplen,
	     struct rte_ring *ring, struct rte_mempool *mp,
	     const struct rte_bpf_prm *prm)
{
	int ret;
	char name[RTE_DEV_NAME_MAX_LEN];

	ret = pdump_validate_port(port, name);
	if (ret < 0)
		return ret;
	ret = pdump_validate_ring_mp(ring, mp);
	if (ret < 0)
		return ret;
	ret = pdump_validate_flags(flags);
	if (ret < 0)
		return ret;

	if (snaplen == 0)
		snaplen = UINT32_MAX;

	return pdump_prepare_client_request(name, queue, flags, snaplen,
					    ENABLE, ring, mp, prm);
}



	if (rte_ring_is_prod_single(ring) || rte_ring_is_cons_single(ring)) {
		PDUMP_LOG_LINE(ERR,
			  "ring with SP or SC set is not valid for pdump,"
			  "must have MP and MC set");




static int
pdump_prepare_client_request(const char *device, uint16_t queue,
			     uint32_t flags, uint32_t snaplen,
			     uint16_t operation,
			     struct rte_ring *ring,
			     struct rte_mempool *mp,
			     const struct rte_bpf_prm *prm)

...
	req->ver = (flags & RTE_PDUMP_FLAG_PCAPNG) ? V2 : V1;
	req->flags = flags & RTE_PDUMP_FLAG_RXTX;---------------------------------------
	req->op = operation;
	req->queue = queue;
	rte_strscpy(req->device, device, sizeof(req->device));

	if ((operation & ENABLE) != 0) {
		req->ring = ring;----------------------------------------------------------
		req->mp = mp;--------------------------------------------------------------
		req->prm = prm;
		req->snaplen = snaplen;
	}

	rte_strscpy(mp_req.name, PDUMP_MP, RTE_MP_MAX_NAME_LEN);-----https://github.com/DPDK/dpdk/blob/6f80df8cb0f889203d7cd27766abcc6ebc720e33/lib/pdump/rte_pdump.c#L433

	mp_req.len_param = sizeof(*req);
	mp_req.num_fds = 0;
	if (rte_mp_request_sync(&mp_req, &mp_reply, &ts) == 0) {------------------
------------------------------------msg sending----mp_request_sync(path, req, reply, &end))---ret = send_msg(dst, req, MP_REQ);--snd = sendmsg(mp_fd, &msgh, 0);




static int
pdump_server(const struct rte_mp_msg *mp_msg, const void *peer)
{
	struct rte_mp_msg mp_resp;
	const struct pdump_request *cli_req;
	struct pdump_response *resp = (struct pdump_response *)&mp_resp.param;

	/* recv client requests */
	if (mp_msg->len_param != sizeof(*cli_req)) {
		PDUMP_LOG_LINE(ERR, "failed to recv from client");
		resp->err_value = -EINVAL;
	} else {
		cli_req = (const struct pdump_request *)mp_msg->param;
		resp->ver = cli_req->ver;
		resp->res_op = cli_req->op;
		resp->err_value = set_pdump_rxtx_cbs(cli_req);------------------------------


rte_pdump_init();----------------------------------------------------!!!!!!!!!!!!!!!!!!!!


int
rte_mp_action_register(const char *name, rte_mp_t action)
{
	struct action_entry *entry;
	const struct internal_config *internal_conf =
		eal_get_internal_configuration();

	if (validate_action_name(name) != 0)
		return -1;

	if (internal_conf->no_shconf) {
		EAL_LOG(DEBUG, "No shared files mode enabled, IPC is disabled");
		rte_errno = ENOTSUP;
		return -1;
	}

	entry = malloc(sizeof(struct action_entry));
	if (entry == NULL) {
		rte_errno = ENOMEM;
		return -1;
	}
	strlcpy(entry->action_name, name, sizeof(entry->action_name));
	entry->action = action;-------------------------------------------------------



int
rte_pdump_init(void)
{
	const struct rte_memzone *mz;
	int ret;

	mz = rte_memzone_reserve(MZ_RTE_PDUMP_STATS, sizeof(*pdump_stats),
				 rte_socket_id(), 0);
	if (mz == NULL) {
		PDUMP_LOG_LINE(ERR, "cannot allocate pdump statistics");
		rte_errno = ENOMEM;
		return -1;
	}
	pdump_stats = mz->addr;
	pdump_stats->mz = mz;

	ret = rte_mp_action_register(PDUMP_MP, pdump_server);------------------



static int
set_pdump_rxtx_cbs(const struct pdump_request *p)-------------------




static void
process_msg(struct mp_msg_internal *m, struct sockaddr_un *s)----------------------------in pdump server
{
	struct pending_request *pending_req;
	struct action_entry *entry;
	struct rte_mp_msg *msg = &m->msg;
	rte_mp_t action = NULL;
	const struct internal_config *internal_conf =
		eal_get_internal_configuration();

	EAL_LOG(DEBUG, "msg: %s", msg->name);

	if (m->type == MP_REP || m->type == MP_IGN) {
		struct pending_request *req = NULL;

		pthread_mutex_lock(&pending_requests.lock);
		pending_req = find_pending_request(s->sun_path, msg->name);
		if (pending_req) {
			memcpy(pending_req->reply, msg, sizeof(*msg));
			/* -1 indicates that we've been asked to ignore */
			pending_req->reply_received =
				m->type == MP_REP ? 1 : -1;

			if (pending_req->type == REQUEST_TYPE_SYNC)
				pthread_cond_signal(&pending_req->sync.cond);
			else if (pending_req->type == REQUEST_TYPE_ASYNC)
				req = async_reply_handle_thread_unsafe(
						pending_req);
		} else {
			EAL_LOG(ERR, "Drop mp reply: %s", msg->name);
			cleanup_msg_fds(msg);
		}
		pthread_mutex_unlock(&pending_requests.lock);

		if (req != NULL)
			trigger_async_action(req);
		return;
	}

	pthread_mutex_lock(&mp_mutex_action);
	entry = find_action_entry_by_name(msg->name);
	if (entry != NULL)
		action = entry->action;
	pthread_mutex_unlock(&mp_mutex_action);

	if (!action) {
		if (m->type == MP_REQ && !internal_conf->init_complete) {
			/* if this is a request, and init is not yet complete,
			 * and callback wasn't registered, we should tell the
			 * requester to ignore our existence because we're not
			 * yet ready to process this request.
			 */
			struct rte_mp_msg dummy;

			memset(&dummy, 0, sizeof(dummy));
			strlcpy(dummy.name, msg->name, sizeof(dummy.name));
			mp_send(&dummy, s->sun_path, MP_IGN);
		} else {
			EAL_LOG(ERR, "Cannot find action: %s",
				msg->name);
		}
		cleanup_msg_fds(msg);
	} else if (action(msg, s->sun_path) < 0) {----------------------------------------------------------process msg
		EAL_LOG(ERR, "Fail to handle message: %s", msg->name);
	}
}


static uint32_t
mp_handle(void *arg __rte_unused)--------------------------------------------------------
{
	struct mp_msg_internal msg;
	struct sockaddr_un sa;
	int fd;

	while ((fd = rte_atomic_load_explicit(&mp_fd, rte_memory_order_relaxed)) >= 0) {
		int ret;

		ret = read_msg(fd, &msg, &sa);
		if (ret <= 0)
			break;

		process_msg(&msg, &sa);------------------------------------------------------------
	}

	return 0;
}

int
rte_mp_channel_init(void)---------------------------------------------------------------------
{
	char path[PATH_MAX];
	int dir_fd;
	const struct internal_config *internal_conf =
		eal_get_internal_configuration();

	/* in no shared files mode, we do not have secondary processes support,
	 * so no need to initialize IPC.
	 */
	if (internal_conf->no_shconf) {
		EAL_LOG(DEBUG, "No shared files mode enabled, IPC will be disabled");
		rte_errno = ENOTSUP;
		return -1;
	}

	/* create filter path */
	create_socket_path("*", path, sizeof(path));
	strlcpy(mp_filter, basename(path), sizeof(mp_filter));

	/* path may have been modified, so recreate it */
	create_socket_path("*", path, sizeof(path));
	strlcpy(mp_dir_path, dirname(path), sizeof(mp_dir_path));

	/* lock the directory */
	dir_fd = open(mp_dir_path, O_RDONLY);
	if (dir_fd < 0) {
		EAL_LOG(ERR, "failed to open %s: %s",
			mp_dir_path, strerror(errno));
		return -1;
	}

	if (flock(dir_fd, LOCK_EX)) {
		EAL_LOG(ERR, "failed to lock %s: %s",
			mp_dir_path, strerror(errno));
		close(dir_fd);
		return -1;
	}

	if (open_socket_fd() < 0) {
		close(dir_fd);
		return -1;
	}

	if (rte_thread_create_internal_control(&mp_handle_tid, "mp-msg",
			mp_handle, NULL) < 0) {---------------------------------------------


/* Launch threads, called at application init(). */
int
rte_eal_init(int argc, char **argv)---------------------



	if (rte_eal_init(eal_argc, eal_argv) < 0)
		rte_exit(EXIT_FAILURE, "EAL init failed: is primary process running?\n");

parse_opts(argc, argv);
	dpdk_init();--------------------------------------------call rte_eal_init

	if (show_interfaces)
		dump_interfaces();



set_pdump_rxtx_cbs==

	/* register RX callback */
	if (flags & RTE_PDUMP_FLAG_RX) {
		end_q = (queue == RTE_PDUMP_ALL_QUEUES) ? nb_rx_q : queue + 1;
		ret = pdump_register_rx_callbacks(p->ver, end_q, port, queue,
						  ring, mp, filter,
						  operation, p->snaplen);
		if (ret < 0)
			return ret;
	}

	/* register TX callback */
	if (flags & RTE_PDUMP_FLAG_TX) {
		end_q = (queue == RTE_PDUMP_ALL_QUEUES) ? nb_tx_q : queue + 1;
		ret = pdump_register_tx_callbacks(p->ver, end_q, port, queue,
						  ring, mp, filter,
						  operation, p->snaplen);
		if (ret < 0)
			return ret;
	}




static int
pdump_register_rx_callbacks(enum pdump_version ver,
			    uint16_t end_q, uint16_t port, uint16_t queue,
			    struct rte_ring *ring, struct rte_mempool *mp,
			    struct rte_bpf *filter,
			    uint16_t operation, uint32_t snaplen)
{
	uint16_t qid;

	qid = (queue == RTE_PDUMP_ALL_QUEUES) ? 0 : queue;
	for (; qid < end_q; qid++) {
		struct pdump_rxtx_cbs *cbs = &rx_cbs[port][qid];----------------------------------

		if (operation == ENABLE) {
			if (cbs->cb) {
				PDUMP_LOG_LINE(ERR,
					"rx callback for port=%d queue=%d, already exists",
					port, qid);
				return -EEXIST;
			}
			cbs->ver = ver;
			cbs->ring = ring;----------------------------------------------------------
			cbs->mp = mp;--------------------------------------------------------------
			cbs->snaplen = snaplen;
			cbs->filter = filter;

			cbs->cb = rte_eth_add_first_rx_callback(port, qid,
								pdump_rx, cbs);-------------------------------
			if (cbs->cb == NULL) {
				PDUMP_LOG_LINE(ERR,
					"failed to add rx callback, errno=%d",
					rte_errno);
				return rte_errno;



static uint16_t
pdump_rx(uint16_t port, uint16_t queue,
	struct rte_mbuf **pkts, uint16_t nb_pkts,
	uint16_t max_pkts __rte_unused, void *user_params)
{
	const struct pdump_rxtx_cbs *cbs = user_params;
	struct rte_pdump_stats *stats = &pdump_stats->rx[port][queue];

	pdump_copy(port, queue, RTE_PCAPNG_DIRECTION_IN,
		   pkts, nb_pkts, cbs, stats);---------------------------copy dump data to cbs's ring and mem pool
	return nb_pkts;
}




	rte_atomic_store_explicit(
		&rte_eth_devices[port_id].post_rx_burst_cbs[queue_id],
		cb, rte_memory_order_release);



const struct rte_eth_rxtx_callback *
rte_eth_add_first_rx_callback(uint16_t port_id, uint16_t queue_id,
		rte_rx_callback_fn fn, void *user_param)
{
#ifndef RTE_ETHDEV_RXTX_CALLBACKS
	rte_errno = ENOTSUP;
	return NULL;
#endif
	/* check input parameters */
	if (!rte_eth_dev_is_valid_port(port_id) || fn == NULL ||
		queue_id >= rte_eth_devices[port_id].data->nb_rx_queues) {
		rte_errno = EINVAL;
		return NULL;
	}

	struct rte_eth_rxtx_callback *cb = rte_zmalloc(NULL, sizeof(*cb), 0);

	if (cb == NULL) {
		rte_errno = ENOMEM;
		return NULL;
	}

	cb->fn.rx = fn;--------------------------------------------------------------------------------------------------fn.rx callback----------------------------
	cb->param = user_param;------------------------------------------------------------------------------------------dest to rte_ring and rte_mem_pool

	rte_spinlock_lock(&eth_dev_rx_cb_lock);
	/* Add the callbacks at first position */
	cb->next = rte_eth_devices[port_id].post_rx_burst_cbs[queue_id];
	/* Stores to cb->fn, cb->param and cb->next should complete before
	 * cb is visible to data plane threads.
	 */
	rte_atomic_store_explicit(
		&rte_eth_devices[port_id].post_rx_burst_cbs[queue_id],-------------------------
		cb, rte_memory_order_release);




void
eth_dev_fp_ops_setup(struct rte_eth_fp_ops *fpo,
		const struct rte_eth_dev *dev)
{
	fpo->rx_pkt_burst = dev->rx_pkt_burst;
	fpo->tx_pkt_burst = dev->tx_pkt_burst;
	fpo->tx_pkt_prepare = dev->tx_pkt_prepare;
	fpo->rx_queue_count = dev->rx_queue_count;
	fpo->rx_descriptor_status = dev->rx_descriptor_status;
	fpo->tx_queue_count = dev->tx_queue_count;
	fpo->tx_descriptor_status = dev->tx_descriptor_status;
	fpo->recycle_tx_mbufs_reuse = dev->recycle_tx_mbufs_reuse;
	fpo->recycle_rx_descriptors_refill = dev->recycle_rx_descriptors_refill;

	fpo->rxq.data = dev->data->rx_queues;
	fpo->rxq.clbk = (void * __rte_atomic *)(uintptr_t)dev->post_rx_burst_cbs;---------------------------------

	fpo->txq.data = dev->data->tx_queues;
	fpo->txq.clbk = (void * __rte_atomic *)(uintptr_t)dev->pre_tx_burst_cbs;
}







static inline uint16_t
rte_eth_rx_burst(uint16_t port_id, uint16_t queue_id,
		 struct rte_mbuf **rx_pkts, const uint16_t nb_pkts)---------------------------------------
{
	uint16_t nb_rx;
	struct rte_eth_fp_ops *p;
	void *qd;

#ifdef RTE_ETHDEV_DEBUG_RX
	if (port_id >= RTE_MAX_ETHPORTS ||
			queue_id >= RTE_MAX_QUEUES_PER_PORT) {
		RTE_ETHDEV_LOG_LINE(ERR,
			"Invalid port_id=%u or queue_id=%u",
			port_id, queue_id);
		return 0;
	}
#endif

	/* fetch pointer to queue data */
	p = &rte_eth_fp_ops[port_id];
	qd = p->rxq.data[queue_id];

#ifdef RTE_ETHDEV_DEBUG_RX
	RTE_ETH_VALID_PORTID_OR_ERR_RET(port_id, 0);

	if (qd == NULL) {
		RTE_ETHDEV_LOG_LINE(ERR, "Invalid Rx queue_id=%u for port_id=%u",
			queue_id, port_id);
		return 0;
	}
#endif

	nb_rx = p->rx_pkt_burst(qd, rx_pkts, nb_pkts);

#ifdef RTE_ETHDEV_RXTX_CALLBACKS
	{
		void *cb;

		/* rte_memory_order_release memory order was used when the
		 * call back was inserted into the list.
		 * Since there is a clear dependency between loading
		 * cb and cb->fn/cb->next, rte_memory_order_acquire memory order is
		 * not required.
		 */
		cb = rte_atomic_load_explicit(&p->rxq.clbk[queue_id],----------------------------------clbk callback
				rte_memory_order_relaxed);
		if (unlikely(cb != NULL))
			nb_rx = rte_eth_call_rx_callbacks(port_id, queue_id,
					rx_pkts, nb_rx, nb_pkts, cb);------------------------------------------in cb handling to have real callback into predefined rte_ring and rte_mem_pool.
	}
#endif

	rte_ethdev_trace_rx_burst(port_id, queue_id, (void **)rx_pkts, nb_rx);
	return nb_rx;
}




uint16_t
rte_eth_call_rx_callbacks(uint16_t port_id, uint16_t queue_id,
	struct rte_mbuf **rx_pkts, uint16_t nb_rx, uint16_t nb_pkts,
	void *opaque)
{
	const struct rte_eth_rxtx_callback *cb = opaque;

	while (cb != NULL) {
		nb_rx = cb->fn.rx(port_id, queue_id, rx_pkts, nb_rx,---------------------------------------------fn.rx() dump copy to rte_ring[mbuf ptr] and rte_mem_pool[actual mbuf with data]
				nb_pkts, cb->param);
		cb = cb->next;
	}


```
- sendmsg for pointers for rte_ring and rte_mem_pool to other server to attach this client requested ring/mempool to the rx/tx burst's callback's extra ring/mbuf memory.
```


===pdump_enable request msg to send===

struct pdump_request {
	uint16_t ver;
	uint16_t op;
	uint32_t flags;
	char device[RTE_DEV_NAME_MAX_LEN];
	uint16_t queue;
	struct rte_ring *ring;---------------
	struct rte_mempool *mp;--------------

	const struct rte_bpf_prm *prm;
	uint32_t snaplen;
};


发送消息pdump_request的buffer 指针(ring, mp)怎么可以？如果传的地址在另一个task如pdump server里可用，表明地址是可以跨task的，要么是物理地址，要么是多线程方式（其共享堆内存),
多进程的方式，就要求虚拟地址在多进程模式也是相同的，这个dpdk里也对多进程模式做了特殊处理，如mmap(addr, page_sz, PROT_READ | PROT_WRITE,
   MAP_SHARED | MAP_POPULATE | MAP_FIXED, fd, 0);保证remap_needed_hugepages->remap_segment: MAP_FIXED映射到相同的虚拟地址(所有进程)，地址是rte eal启动参数--base-virtaddr 0x6a0000000000指定的。

ring = mz->mzddr (r = mz->addr;)

	if ((operation & ENABLE) != 0) {
		req->ring = ring;----------------------------------------------------------
		req->mp = mp;--------------------------------------------------------------
		req->prm = prm;
		req->snaplen = snaplen;
	}

	rte_strscpy(mp_req.name, PDUMP_MP, RTE_MP_MAX_NAME_LEN);-----https://github.com/DPDK/dpdk/blob/6f80df8cb0f889203d7cd27766abcc6ebc720e33/lib/pdump/rte_pdump.c#L433

	mp_req.len_param = sizeof(*req);
	mp_req.num_fds = 0;
	if (rte_mp_request_sync(&mp_req, &mp_reply, &ts) == 0) {------------------
------------------------------------msg sending----mp_request_sync(path, req, reply, &end))---ret = send_msg(dst, req, MP_REQ);--snd = sendmsg(mp_fd, &msgh, 0);


===detail:


static struct rte_ring *create_ring(void)
{
	struct rte_ring *ring;
	char ring_name[RTE_RING_NAMESIZE];
	size_t size, log2;

	/* Find next power of 2 >= size. */
	size = ring_size;
	log2 = sizeof(size) * 8 - __builtin_clzl(size - 1);
	size = 1u << log2;

	if (size != ring_size) {
		fprintf(stderr, "Ring size %u rounded up to %zu\n",
			ring_size, size);
		ring_size = size;
	}

	/* Want one ring per invocation of program */
	snprintf(ring_name, sizeof(ring_name),
		 "dumpcap-%d", getpid());

	ring = rte_ring_create(ring_name, ring_size,
			       rte_socket_id(), 0);-----------------------




/* create the ring */
struct rte_ring *
rte_ring_create(const char *name, unsigned int count, int socket_id,
		unsigned int flags)
{
	return rte_ring_create_elem(name, sizeof(void *), count, socket_id,
		flags);
}

rte_ring_create_elem:
	mz = rte_memzone_reserve_aligned(mz_name, ring_size, socket_id,
					 mz_flags, alignof(typeof(*r)));
	if (mz != NULL) {
		r = mz->addr;


/* allocate memory on heap */
		mz_addr = malloc_heap_alloc(NULL, requested_len, socket_id,
				flags, align, bound, contig);




static void *
heap_alloc(struct malloc_heap *heap, const char *type __rte_unused, size_t size,
		unsigned int flags, size_t align, size_t bound, bool contig)
{
	struct malloc_elem *elem;
	size_t user_size = size;

	size = RTE_CACHE_LINE_ROUNDUP(size);
	align = RTE_CACHE_LINE_ROUNDUP(align);

	/* roundup might cause an overflow */
	if (size == 0)
		return NULL;
	elem = find_suitable_element(heap, size, flags, align, bound, contig);
	if (elem != NULL) {
		elem = malloc_elem_alloc(elem, size, align, bound, contig);-------------------------------

		/* increase heap's count of allocated elements */
		heap->alloc_count++;

		asan_set_redzone(elem, user_size);
	}

	return elem == NULL ? NULL : (void *)(&elem[1]);
}


```
