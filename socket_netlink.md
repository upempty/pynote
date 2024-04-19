

    Netlink通过自定义一种新的协议并加入协议族即可通过socket API使用Netlink协议完成数据交换，而ioctl和proc文件系统均需要通过程序加入相应的设备或文件。  
    >> 可以自定义protocol 在用户太client和内核模块server，这样就可以通过socket交互了：fd = socket(PF_NETLINK, SOCK_RAW, NETLINK_NewID....)   
    Netlink使用socket缓存队列，是一种异步通信机制，而ioctl是同步通信机制，如果传输的数据量较大，会影响系统性能。  
    Netlink支持多播，属于一个Netlink组的模块和进程都能获得该多播消息。  
    Netlink允许内核发起会话，而ioctl和系统调用只能由用户空间进程发起。  

    


- user space sending msg
```
int ret = sendmsg(sock_fd, &msg, 0);
```

- user space receiving msg
```
recvmsg(sock_fd, &msg, 0);  //msg is also receiver for read
```

- Kernel sending msg
```
pid = nlh->nlmsg_pid; // Sending process port ID, will send new message back to the 'user space sender'
res = nlmsg_unicast(nl_sk, skb_out, pid); //nlmsg_unicast - unicast a netlink message
```

- Kernel receiving msg
```
static int __init hello_init(void)
{
	struct netlink_kernel_cfg cfg = {
		.input = hello_nl_recv_msg, /////////////////////
	};
```

- user sending->kernel recv----inside callback, then kernel send------>user receiving
```
static const struct proto_ops netlink_ops = {

.sendmsg =	netlink_sendmsg,
---
netlink_sendmsg()
 -> err = netlink_unicast(sk, skb, dst_portid, msg->msg_flags&MSG_DONTWAIT);
  -> netlink_unicast_kernel(sk, skb, ssk);
    -> nlk->netlink_rcv(skb);
       !!!!!!!!!!!nlk_sk(sk)->netlink_rcv = cfg->input;
----------------------for example, inside it will call nlmsg_unicast(nl_sk, skb_out, pid) for sedning to pid's recv queue user from kernel.
------------------------by get pid/port id's sock, then send to this sock's buffer:
--------------------------sk = netlink_getsockbyportid(ssk, portid);
--------------------------netlink_sendskb(sk, skb);

registered for kernel rcv func
---
genl_pernet_init(struct net *net) or or kernel module registered with new protocol:
---------------
genl_pernet_init(struct net *net)
{
	struct netlink_kernel_cfg cfg = {
		.input		= genl_rcv,-----------none kernel sending.

struct netlink_kernel_cfg cfg = {
		.input = hello_nl_recv_msg, // inside including kernel msg/response sending to pid
	};
...
  netlink_kernel_create
   -> __netlink_kernel_create
	 if (cfg && cfg->input)
		 nlk_sk(sk)->netlink_rcv = cfg->input;

```

https://github.com/helight/kernel_modules/tree/master/netlink_test

![image](https://github.com/upempty/pynote/assets/52414719/3f69fc9e-e32b-4385-aaaf-05f8d713e32b)

![image](https://github.com/upempty/pynote/assets/52414719/56a8eb79-d254-43b4-be7f-f16fac28878f)
```
static inline struct nlmsghdr *nlmsg_hdr(const struct sk_buff *skb)
{
	return (struct nlmsghdr *)skb->data;
}

```


```


--struct netlink_sock *nlk = nlk_sk(sk);

struct sockaddr_nl {
	__kernel_sa_family_t	nl_family;	/* AF_NETLINK	*/
	unsigned short	nl_pad;		/* zero		*/
	__u32		nl_pid;		/* port ID	*/
       	__u32		nl_groups;	/* multicast groups mask */
};

struct nlmsghdr {
	__u32		nlmsg_len;	/* Length of message including header */
	__u16		nlmsg_type;	/* Message content */
	__u16		nlmsg_flags;	/* Additional flags */
	__u32		nlmsg_seq;	/* Sequence number */
	__u32		nlmsg_pid;	/* Sending process port ID */
};



static const struct proto_ops netlink_ops = {
	.family =	PF_NETLINK,
	.owner =	THIS_MODULE,
	.release =	netlink_release,
	.bind =		netlink_bind,
	.connect =	netlink_connect,
	.socketpair =	sock_no_socketpair,
	.accept =	sock_no_accept,
	.getname =	netlink_getname,
	.poll =		datagram_poll,
	.ioctl =	netlink_ioctl,
	.listen =	sock_no_listen,
	.shutdown =	sock_no_shutdown,
	.setsockopt =	netlink_setsockopt,
	.getsockopt =	netlink_getsockopt,
	.sendmsg =	netlink_sendmsg,
	.recvmsg =	netlink_recvmsg,
	.mmap =		sock_no_mmap,
};


/**
 * 	datagram_poll - generic datagram poll
 *	@file: file struct
 *	@sock: socket
 *	@wait: poll table
 *
 *	Datagram poll: Again totally generic. This also handles
 *	sequenced packet sockets providing the socket receive queue
 *	is only ever holding data ready to receive.
 *
 *	Note: when you *don't* use this routine for this protocol,
 *	and you use a different write policy from sock_writeable()
 *	then please supply your own write_space callback.
 */
__poll_t datagram_poll(struct file *file, struct socket *sock,
			   poll_table *wait)
{
	struct sock *sk = sock->sk;
	__poll_t mask;
	u8 shutdown;

	sock_poll_wait(file, sock, wait);
	mask = 0;

	/* exceptional events? */
	if (READ_ONCE(sk->sk_err) ||
	    !skb_queue_empty_lockless(&sk->sk_error_queue))
		mask |= EPOLLERR |
			(sock_flag(sk, SOCK_SELECT_ERR_QUEUE) ? EPOLLPRI : 0);

	shutdown = READ_ONCE(sk->sk_shutdown);
	if (shutdown & RCV_SHUTDOWN)
		mask |= EPOLLRDHUP | EPOLLIN | EPOLLRDNORM;
	if (shutdown == SHUTDOWN_MASK)
		mask |= EPOLLHUP;

	/* readable? */
	if (!skb_queue_empty_lockless(&sk->sk_receive_queue))
		mask |= EPOLLIN | EPOLLRDNORM;

	/* Connection-based need to check for termination and startup */
	if (connection_based(sk)) {
		int state = READ_ONCE(sk->sk_state);

		if (state == TCP_CLOSE)
			mask |= EPOLLHUP;
		/* connection hasn't started yet? */
		if (state == TCP_SYN_SENT)
			return mask;
	}

	/* writable? */
	if (sock_writeable(sk))
		mask |= EPOLLOUT | EPOLLWRNORM | EPOLLWRBAND;
	else
		sk_set_bit(SOCKWQ_ASYNC_NOSPACE, sk);

	return mask;
}
EXPORT_SYMBOL(datagram_poll);





static int netlink_connect(struct socket *sock, struct sockaddr *addr,
			   int alen, int flags)
{
	int err = 0;
	struct sock *sk = sock->sk;
	struct netlink_sock *nlk = nlk_sk(sk);
	struct sockaddr_nl *nladdr = (struct sockaddr_nl *)addr;

	if (alen < sizeof(addr->sa_family))
		return -EINVAL;

	if (addr->sa_family == AF_UNSPEC) {
		/* paired with READ_ONCE() in netlink_getsockbyportid() */
		WRITE_ONCE(sk->sk_state, NETLINK_UNCONNECTED);
		/* dst_portid and dst_group can be read locklessly */
		WRITE_ONCE(nlk->dst_portid, 0);
		WRITE_ONCE(nlk->dst_group, 0);
		return 0;
	}
	if (addr->sa_family != AF_NETLINK)
		return -EINVAL;

	if (alen < sizeof(struct sockaddr_nl))
		return -EINVAL;

	if ((nladdr->nl_groups || nladdr->nl_pid) &&
	    !netlink_allowed(sock, NL_CFG_F_NONROOT_SEND))
		return -EPERM;

	/* No need for barriers here as we return to user-space without
	 * using any of the bound attributes.
	 * Paired with WRITE_ONCE() in netlink_insert().
	 */
	if (!READ_ONCE(nlk->bound))
		err = netlink_autobind(sock);

	if (err == 0) {
		/* paired with READ_ONCE() in netlink_getsockbyportid() */
		WRITE_ONCE(sk->sk_state, NETLINK_CONNECTED);
		/* dst_portid and dst_group can be read locklessly */
		WRITE_ONCE(nlk->dst_portid, nladdr->nl_pid);
		WRITE_ONCE(nlk->dst_group, ffs(nladdr->nl_groups));
	}

	return err;
}



```

- broadcast
```
用户态：
src_addr.nl_family = AF_NETLINK;
src_addr.nl_pad = 0;
src_addr.nl_pid = getpid();-----------------------------------------------
src_addr.nl_groups = GROUP_IB; // Multicast--------------------------------

err = bind(sock_fd, (struct sockaddr*)&src_addr, sizeof(src_addr));


内核态:

static void netlink_ib_open()
{
struct sk_buff *skb = NULL;
struct nlmsghdr *nlh = NULL;
int err;

nl_sk = netlink_kernel_create(NETLINK_IB, GROUP_IB, nl_ib_data_ready,
THIS_MODULE);
if (nl_sk == NULL) {
printk(KERN_ALERT "Error during netlink_kernel_create");
}

skb = alloc_skb(NLMSG_SPACE(MAX_PAYLOAD),GFP_KERNEL);
nlh = (struct nlmsghdr *)skb->data;
nlh = skb_put(skb, NLMSG_ALIGN(1024));
nlh->nlmsg_pid = 0; // Send from kernel
nlh->nlmsg_flags = 0;

strcpy(NLMSG_DATA(nlh), "Greeting from kernel!");
NETLINK_CB(skb).pid = 0;
NETLINK_CB(skb).dst_pid = 0; //Multicast-------------------------------
NETLINK_CB(skb).dst_group = GROUP_IB;// 0 if unicast-------------------

printk( KERN_ALERT "Sending data...\n");
//err = netlink_unicast(nl_sk, skb, 6992, GFP_KERNEL);
err = netlink_broadcast(nl_sk, skb, 0, GROUP_IB, GFP_KERNEL);

```
