- socket sending
```
send()
https://elixir.bootlin.com/linux/latest/source/tools/testing/selftests/net/tcp_mmap.c

__libc_send
--weak_alias (__libc_send, send) in <glibc c library:include/sys/socket.h>
--https://elixir.bootlin.com/glibc/latest/source/socket/sys/socket.h#L138
--https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/send.c#L33
SYSCALL_CANCEL (send,..)
INLINE_SYSCALL_CALL (__VA_ARGS__); 	
INTERNAL_SYSCALL_RAW(SYS_ify(name), nr, args)
--SYS_ify是个宏，用于将syscall name转换为syscall number
--#define SYS_ify(syscall_name)	(__NR_##syscall_name)
--__NR_send.....#define __NR_send 289 （arm (32-bit/EABI)）

!!!!!!!async thinking!!!!! We cannot let a process access peripherals.---Cannot access hardware directly

glibc::::
# define INTERNAL_SYSCALL_RAW(name, nr, args...)		\
  ({ long _sys_result;                                          \
     {                                                          \
       LOAD_ARGS_##nr (args)                                    \
       register long _x8 asm ("x8") = (name);                   \------save syscall number to x8 register.
       asm volatile ("svc       0       // syscall " # name     \------最后svc 0触发进入内核态, Supervisor Call causes an exception to be taken to EL1.
                     : "=r" (_x0) : "r"(_x8) ASM_ARGS_##nr : "memory"); \
       _sys_result = _x0;                                       \
     }                                                          \
     _sys_result; })
!!!!!!!async thinking!!!!!
Interrupt Number Code address of IDT
0x30 (syscall in JOS) t_syscall
系统调用切换到内核态后， 首先会进入异常向量表处理。
 kernel/arch/arm64/kernel/entry.S
https://elixir.bootlin.com/linux/latest/source/arch/arm64/kernel/entry.S
用户态的系统调用会在el0_sync处理

el0_svc_common
invoke_syscall(regs, scno, sc_nr, syscall_table);

		syscall_fn_t syscall_fn;
		syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
		ret = __invoke_syscall(regs, syscall_fn);

syscall_fn(regs);

sys_send = SYSCALL_DEFINE4(send,..)
__sys_sendto(fd, buff, len, flags, NULL, 0);
!!!!!!!!!!
https://elixir.bootlin.com/linux/latest/source/net/socket.c#L2199
sock = sockfd_lookup_light(fd, &err, &fput_needed);
if addr: err = move_addr_to_kernel(addr, addr_len, &address);
__sock_sendmsg(sock, &msg);
	int err = security_socket_sendmsg(sock, msg,
					  msg_data_left(msg));---socket_sendmsg if hook.FUNC is not null.
	return err ?: sock_sendmsg_nosec(sock, msg);

static inline int sock_sendmsg_nosec(struct socket *sock, struct msghdr *msg)
{
	int ret = INDIRECT_CALL_INET(READ_ONCE(sock->ops)->sendmsg,-----------------or 
                                      inet6_sendmsg,--------------------------------or
				     inet_sendmsg,----------------------------------or
                                     sock, msg,
				     msg_data_left(msg));

--inet_sendmsg---tcp_sendmsg, udp_sendmsg,
inet_sendmsg
tcp_sendmsg/udp_sendmsg
	lock_sock(sk);
	ret = tcp_sendmsg_locked(sk, msg, size);
	release_sock(sk);

tcp_sendmsg_locked
if ((flags & MSG_ZEROCOPY) && size)
 skb_peek_tail(&sk->sk_write_queue);--Peek an &sk_buff which is head's pre.
 msg_zerocopy_realloc(sk, size, skb_zcopy(skb));

/* Wait for a connection to finish. One exception is TCP Fast Open
	 * (passive side) where data is allowed to be sent before a connection
	 * is fully established.
*/
--what is msg structure?
skb = tcp_write_queue_tail(sk);---skb_peek_tail(&sk->sk_write_queue);

first_skb = tcp_rtx_and_write_queues_empty(sk);
skb = tcp_stream_alloc_skb(sk, sk->sk_allocation,
						   first_skb);

skb_copy_to_page_nocache(sk, &msg->msg_iter, skb,
						       pfrag->page,
						       pfrag->offset,
						       copy);-- *pfrag = sk_page_frag(sk);

      err = skb_do_copy_data_nocache(sk, skb, from, page_address(page) + off,
				       copy, skb->len);
	if (err)
		return err;

	skb_len_add(skb, copy);
	sk_wmem_queued_add(sk, copy);--	WRITE_ONCE(sk->sk_wmem_queued, sk->sk_wmem_queued + val);

/* Update the skb. */
			if (merge) {
				skb_frag_size_add(&skb_shinfo(skb)->frags[i - 1], copy);
			} else {
				skb_fill_page_desc(skb, i, pfrag->page,
						   pfrag->offset, copy);
				page_ref_inc(pfrag->page);
			}
WRITE_ONCE(tp->write_seq, tp->write_seq + copy);
tcp_skb_pcount_set(skb, 0);

/* push */
		if (forced_push(tp)) {
			tcp_mark_push(tp, skb);
			__tcp_push_pending_frames(sk, mss_now, TCP_NAGLE_PUSH);--(tcp_write_xmit(sk, cur_mss, nonagle, 0, sk_gfp_mask(sk, GFP_ATOMIC)))
		} else if (skb == tcp_send_head(sk))
			tcp_push_one(sk, mss_now);--tcp_write_xmit(sk, mss_now, TCP_NAGLE_PUSH, 1, sk->sk_allocation);

continue; when while (msg_data_left(msg))

tcp_write_xmit

FIFO
tcp_transmit_skb(sk, skb, 1, gfp) when while ((skb = tcp_send_head(sk)))
__tcp_transmit_skb(sk, skb, clone_it, gfp_mask,tcp_sk(sk)->rcv_nxt);
--static int __tcp_transmit_skb(struct sock *sk, struct sk_buff *skb,
/* This routine actually transmits TCP packets queued in by
 * tcp_do_sendmsg().  This is used by both the initial
 * transmission and possible later retransmissions.
 * All SKB's seen here are completely headerless.  It is our
 * job to build the TCP header, and pass the packet down to
 * IP so it can do the same plus pass the packet off to the
 * device.
 *
 * We are working here with either a clone of the original
 * SKB, or a fresh unique copy made by the retransmit engine.
 */

--skb_push(skb, tcp_header_size);
void *skb_push(struct sk_buff *skb, unsigned int len);
static inline void *__skb_push(struct sk_buff *skb, unsigned int len)
{
	skb->data -= len;
	skb->len  += len;
	return skb->data;
}
inet6_csk_xmit/ip_queue_xmit,
	err = INDIRECT_CALL_INET(icsk->icsk_af_ops->queue_xmit,
				 inet6_csk_xmit, ip_queue_xmit,
				 sk, skb, &inet->cork.fl);

/* Note: skb->sk can be different from sk, in case of tunnels */
__ip_queue_xmit(sk, skb, fl, READ_ONCE(inet_sk(sk)->tos)); when ip_queue_xmit,

ip_local_out(net, sk, skb);

__ip_local_out(net, sk, skb);
skb = l3mdev_ip_out(sk, skb);
 switch (proto) {
	case AF_INET:
		return vrf_ip_out(vrf_dev, sk, skb);--dev_queue_xmit_nit(skb, vrf_dev);

dst_output if (likely(err == 1)) of above __ip_local_out(net, sk, skb);

ip6_output/ip_output,
--INDIRECT_CALL_INET(skb_dst(skb)->output, ip6_output, ip_output, net, sk, skb);
ip_finish_output
__ip_finish_output
ip_fragment-----	if (skb->len > mtu || IPCB(skb)->frag_max_size) return ip_fragment(net, sk, skb, mtu, ip_finish_output2);
ip_do_fragment
for (;;) {
 err = output(net, sk, skb);
 skb = ip_fraglist_next(&iter);

ip_finish_output2 as output==ip_finish_output2 which is one param in ip do frag.
neigh_output(neigh, skb, is_v6gw);
neigh_hh_output
dev_queue_xmit(skb); -- after _skb_push(skb, hh_len);
__dev_queue_xmit
 __dev_xmit_skb(skb, q, dev, txq); when q = rcu_dereference_bh(txq->qdisc);-->
 --sch_direct_xmit(skb, q, dev, txq, NULL, true) -》（skb = dev_hard_start_xmit(skb, dev, txq, &ret);）or dev_qdisc_enqueue(skb, q, &to_free, txq) with qdisc_run(q);--->__netif_schedule __netif_reschedule(q) raise_softirq_irqoff(NET_TX_SOFTIRQ);
 dev_hard_start_xmit(skb, dev, txq, &rc);
 -- rc = xmit_one(skb, dev, txq, next != NULL); while (skb) // skb = next;
 xmit_one 
  netdev_start_xmit(skb, dev, txq, more);
  __netdev_start_xmit     ops->ndo_start_xmit(skb, dev); of net_device_ops {
! depends on drivers used for registering
----.ndo_start_xmit		= fm10k_xmit_frame,----struct net_device_ops fm10k_netdev_ops = {
----.ndo_start_xmit		= e1000_xmit_frame,

err = fm10k_xmit_frame_ring(skb, interface->tx_ring[r_idx]);
//
	/* record the location of the first descriptor for this packet */
	first = &tx_ring->tx_buffer[tx_ring->next_to_use];
	first->skb = skb;
 fm10k_tx_map(tx_ring, first);
 dma = dma_map_single(tx_ring->dev, data, size, DMA_TO_DEVICE);

		if (fm10k_tx_desc_push(tx_ring, tx_desc++, i++,
				       dma, size, flags)) {
			tx_desc = FM10K_TX_DESC(tx_ring, 0);
			i = 0;
		}

		size = skb_frag_size(frag);
		data_len -= size;

		dma = skb_frag_dma_map(tx_ring->dev, frag, 0, size,
				       DMA_TO_DEVICE);

		tx_buffer = &tx_ring->tx_buffer[i];




# define INTERNAL_SYSCALL_RAW(name, nr, args...)		\
  ({								\
      register int _a1 asm ("a1");				\
      int _nametmp = name;					\
      LOAD_ARGS_##nr (args)					\
      register int _name asm ("ip") = _nametmp;			\
      asm volatile ("bl      __libc_do_syscall"			\
                    : "=r" (_a1)				\
                    : "r" (_name) ASM_ARGS_##nr			\
                    : "memory", "lr");				\
      _a1; })
#else /* ARM */
# undef INTERNAL_SYSCALL_RAW
# define INTERNAL_SYSCALL_RAW(name, nr, args...)		\
  ({								\
       register int _a1 asm ("r0"), _nr asm ("r7");		\
       LOAD_ARGS_##nr (args)					\
       _nr = name;						\
       asm volatile ("swi	0x0	@ syscall " #name	\
		     : "=r" (_a1)				\
		     : "r" (_nr) ASM_ARGS_##nr			\
		     : "memory");				\
       _a1; })
#endif



__SYSCALL(__NR_send, sys_send) #####associate that number with sys_read(
!!!!!!!!!!
a syscall is triggered by the int80 instruction. 
Once it's called, the kernel checks the value stored in RAX - 
this is the syscall number, which defines what syscall gets run. \

--#define __SYSCALL(nr, call) [nr] = (call),
so
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall,
    [0] = sys_read,


call    *sys_call_table(, %rax, 8)

--#define __SYSCALL(nr, entry) .quad entry
	.data
	.align 3
	.globl sys_call_table
--sys_call_table:


sys_send = SYSCALL_DEFINE4(send,..)
__sys_sendto(fd, buff, len, flags, NULL, 0);




User application contains code that fills general purpose register with the values (system call number and arguments of this system call);
Processor switches from the user mode to kernel mode and starts execution of the system call entry - entry_SYSCALL_64;
entry_SYSCALL_64 switches to the kernel stack and saves some general purpose registers, old stack and code segment, flags and etc... on the stack;
entry_SYSCALL_64 checks the system call number in the rax register, searches a system call handler in the sys_call_table and calls it, if the number of a system call is correct;
If a system call is not correct, jump on exit from system call;
After a system call handler will finish its work, restore general purpose registers, old stack, flags and return address and exit from the entry_SYSCALL_64 with the sysretq instruction.



https://elixir.bootlin.com/linux/latest/source/net/socket.c#L2210
/*
 *	Send a datagram down a socket.
 */

SYSCALL_DEFINE4(send, int, fd, void __user *, buff, size_t, len,
		unsigned int, flags)
{
	return __sys_sendto(fd, buff, len, flags, NULL, 0);
}





void *start_server(void *arg)
{
	int server_fd = (int)(unsigned long)arg;
	struct sockaddr_in addr;
	socklen_t addrlen = sizeof(addr);
	char *buf;
	int fd;
	int r;

	buf = malloc(BUF_SIZE);

	for (;;) {
		fd = accept(server_fd, (struct sockaddr *)&addr, &addrlen);
		if (fd == -1) {
			perror("accept");
			break;
		}
		do {
			r = send(fd, buf, BUF_SIZE, 0);




struct proto tcp_prot = {
	.name			= "TCP",
	.owner			= THIS_MODULE,
	.close			= tcp_close,
	.pre_connect		= tcp_v4_pre_connect,
	.connect		= tcp_v4_connect,
	.disconnect		= tcp_disconnect,
	.accept			= inet_csk_accept,
	.ioctl			= tcp_ioctl,
	.init			= tcp_v4_init_sock,
	.destroy		= tcp_v4_destroy_sock,
	.shutdown		= tcp_shutdown,
	.setsockopt		= tcp_setsockopt,
	.getsockopt		= tcp_getsockopt,
	.bpf_bypass_getsockopt	= tcp_bpf_bypass_getsockopt,
	.keepalive		= tcp_set_keepalive,
	.recvmsg		= tcp_recvmsg,
	.sendmsg		= tcp_sendmsg,
	.splice_eof		= tcp_splice_eof,
	.backlog_rcv		= tcp_v4_do_rcv,
	.release_cb		= tcp_release_cb,
	.hash			= inet_hash,
	.unhash			= inet_unhash,
	.get_port		= inet_csk_get_port,
	.put_port		= inet_put_port,
#ifdef CONFIG_BPF_SYSCALL
	.psock_update_sk_prot	= tcp_bpf_update_proto,
#endif
	.enter_memory_pressure	= tcp_enter_memory_pressure,
	.leave_memory_pressure	= tcp_leave_memory_pressure,
	.stream_memory_free	= tcp_stream_memory_free,
	.sockets_allocated	= &tcp_sockets_allocated,
	.orphan_count		= &tcp_orphan_count,

	.memory_allocated	= &tcp_memory_allocated,
	.per_cpu_fw_alloc	= &tcp_memory_per_cpu_fw_alloc,

	.memory_pressure	= &tcp_memory_pressure,
	.sysctl_mem		= sysctl_tcp_mem,
	.sysctl_wmem_offset	= offsetof(struct net, ipv4.sysctl_tcp_wmem),
	.sysctl_rmem_offset	= offsetof(struct net, ipv4.sysctl_tcp_rmem),
	.max_header		= MAX_TCP_HEADER,
	.obj_size		= sizeof(struct tcp_sock),
	.slab_flags		= SLAB_TYPESAFE_BY_RCU,
	.twsk_prot		= &tcp_timewait_sock_ops,
	.rsk_prot		= &tcp_request_sock_ops,
	.h.hashinfo		= NULL,
	.no_autobind		= true,
	.diag_destroy		= tcp_abort,
};
EXPORT_SYMBOL(tcp_prot);


```
