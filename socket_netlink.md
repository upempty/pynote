

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
