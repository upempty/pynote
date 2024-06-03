
sending data via UDS is to push to receving queue directly  
- unix domain socket
- https://elixir.bootlin.com/linux/v4.12.5/source/net/unix/af_unix.c#L904

```
==
After connect from client, and accept from server, the socket pair with client sock and "new created sock" for server relation x->perr created.
thus the data sent will be put into peer's receiveing queue.
e.g. s1-----s2 socket
s1 sends to s2's receive queue
s2 sends to s1's receive queue


static int unix_stream_connect(struct socket *sock, struct sockaddr *uaddr,
			       int addr_len, int flags)

/* create new sock for complete connection */
	newsk = unix_create1(sock_net(sk), NULL, 0);
	if (newsk == NULL)
		goto out;

	/* Allocate skb for sending to listening sock */
	skb = sock_wmalloc(newsk, 1, 0, GFP_KERNEL);


	/*  Find listening sock. */
	other = unix_find_other(net, sunaddr, addr_len, sk->sk_type, hash, &err);===u====-----err = kern_path(sunname->sun_path, LOOKUP_FOLLOW, &path);inode = d_backing_inode(path.dentry);u = unix_find_socket_byinode(inode);

     //associated client/server each other from clinet side.
     unix_peer(newsk)	= sk;======================================================peer sk set
     unix_peer(sk)	= newsk;===================================================peer sk set

     __skb_queue_tail(&other->sk_receive_queue, skb);-----------------------------newsk (owner sock's skb) in client used. send to server's accept receive queue


static int unix_accept(struct socket *sock, struct socket *newsock, int flags,
		       bool kern)

    skb = skb_recv_datagram(sk, 0, flags&O_NONBLOCK, &err);-----------------------passed sk

	tsk = skb->sk;--------------------------------------------------------------newsk received, so server's newsk is associated with client sk!!!!
	skb_free_datagram(sk, skb);
	wake_up_interruptible(&unix_sk(sk)->peer_wait);

	/* attach accepted sock to socket */
	unix_state_lock(tsk);
	newsock->state = SS_CONNECTED;
	unix_sock_inherit_flags(sock, newsock);
	sock_graft(tsk, newsock);-----------------------------------------------------------

-----
static inline void sock_graft(struct sock *sk, struct socket *parent)--------------------
{
	write_lock_bh(&sk->sk_callback_lock);
	sk->sk_wq = parent->wq;
	parent->sk = sk;--------------------------------------------------------
	sk_set_socket(sk, parent);
	sk->sk_uid = SOCK_INODE(parent)->i_uid;
	security_sock_graft(sk, parent);
	write_unlock_bh(&sk->sk_callback_lock);
}



static int unix_stream_sendmsg(struct socket *sock, struct msghdr *msg,
			       size_t len)
{

other = unix_peer(sk);------------------------------------------------------------------send to peer/pair's receive queue!!!!!!!!!!!
skb_queue_tail(&other->sk_receive_queue, skb);------------------------------------------



```
