
- socket and VFS
```
- socket id is unique within the process level, but children process also can share to use the same socket fd.
===vfs file, different file operation for different "file type"
===vfs abstract for file types: regular file, directory, socket, pipe, device file,
=== symbolic link, evnetfd, semaphore, msg queue, shared memory, mount point etc.
file->const struct file_operations	*f_op---which is associated with socket_file_ops when socket() created.
so if need to use VFS api to receive/read the msg, 
then it needs to use read_iter, which is defined for socket file operations, even not defining the read(), but read_iter.
And to get socket from file->private_data.

struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);--------------either read or read_iter is assigned as it's enough one of them is not null assigned.
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);--------------------either read or read_iter is assigned as it's enough one of them is not null assigned.
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);


struct socket *sock_from_file(struct file *file)
{
	if (file->f_op == &socket_file_ops)
		return file->private_data;	/* set in sock_alloc_file */!!!!!!!!!!!!!!!!

	return NULL;
}



struct file *sock_alloc_file(struct socket *sock, int flags, const char *dname)
{
	struct file *file;

	if (!dname)
		dname = sock->sk ? sock->sk->sk_prot_creator->name : "";

	file = alloc_file_pseudo(SOCK_INODE(sock), sock_mnt, dname,
				O_RDWR | (flags & O_NONBLOCK),
				&socket_file_ops);
	if (IS_ERR(file)) {
		sock_release(sock);
		return file;
	}

	file->f_mode |= FMODE_NOWAIT;
	sock->file = file;
	file->private_data = sock;---------------------
	stream_open(SOCK_INODE(sock), file);
	return file;



static int sock_map_fd(struct socket *sock, int flags)
{
	struct file *newfile;
	int fd = get_unused_fd_flags(flags);
	if (unlikely(fd < 0)) {
		sock_release(sock);
		return fd;
	}

	newfile = sock_alloc_file(sock, flags, NULL);


int __sys_socket(int family, int type, int protocol)
{
	struct socket *sock;
	int flags;

	sock = __sys_socket_create(family, type,
				   update_socket_protocol(family, type, protocol));
	if (IS_ERR(sock))
		return PTR_ERR(sock);

	flags = type & ~SOCK_TYPE_MASK;
	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

	return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
}

	case SYS_SOCKET:
		err = __sys_socket(a0, a1, a[2]);

...
struct file {
	union {
		/* fput() uses task work when closing and freeing file (default). */
		struct callback_head 	f_task_work;
		/* fput() must use workqueue (most kernel threads). */
		struct llist_node	f_llist;
		unsigned int 		f_iocb_flags;
	};

	/*
	 * Protects f_ep, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	fmode_t			f_mode;
	atomic_long_t		f_count;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	unsigned int		f_flags;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;-----------------------------

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct hlist_head	*f_ep;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */


```

- socket: levels to use protocol operations or file operations (f_op: regular or socket or eventfd or else)

```
vfs to associate fd with underlayer socket, so can use common abstract API whatever regular file or net socket etc.
   file->f_op-> socket_file_ops
        ->private_data ->socket structure detail info (state, protocol, buffers)

socket_file_ops
- function read and write which is actually abstract f_op (file operation):
- function send and recv it is via proto_ops.
f_op:-----------------------------
either read or read_iter is assigned as it's enough one of them is not null assigned.
-- read()==SYSCALL_DEFINE3(read...)->ksys_read->vfs_read->
vfs_read
== ret = file->f_op->read(file, buf, count, pos);
const struct file_operations	*f_op;
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);

===vfs_read
	if (file->f_op->read)-----------------------------------if there is some file operation assigned this,
		ret = file->f_op->read(file, buf, count, pos);
	else if (file->f_op->read_iter)-------------------------if there is some file operation assigned this, e.g. socket file ops indeed as below!!!!
		ret = new_sync_read(file, buf, count, pos);----ret = call_read_iter(filp, &kiocb, &iter);--->return file->f_op->read_iter(kio, iter);
	else
		ret = -EINVAL;

===vfs_write
	if (file->f_op->write)
		ret = file->f_op->write(file, buf, count, pos);
	else if (file->f_op->write_iter)
		ret = new_sync_write(file, buf, count, pos);
	else
		ret = -EINVAL;


socket file:

https://elixir.bootlin.com/linux/latest/source/net/socket.c#L154
/*
 *	Socket files have a set of 'special' operations as well as the generic file ones. These don't appear
 *	in the operation structures but are done directly via the socketcall() multiplexor.
 */

static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,----------------------------------------------------can be called inside of vfs_read.
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.uring_cmd =    io_uring_cmd_sock,
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.splice_write = splice_to_socket,
	.splice_read =	sock_splice_read,
	.splice_eof =	sock_splice_eof,
	.show_fdinfo =	sock_show_fdinfo,
};

int register_filesystem(struct file_system_type * fs)--------------------
{
	int res = 0;
	struct file_system_type ** p;

	if (fs->parameters &&
	    !fs_validate_description(fs->name, fs->parameters))
		return -EINVAL;

	BUG_ON(strchr(fs->name, '.'));
	if (fs->next)
		return -EBUSY;
	write_lock(&file_systems_lock);
	p = find_filesystem(fs->name, strlen(fs->name));----e.g. usbfs, ext2_fs, etc based on the name to searching. 
	if (*p)
		res = -EBUSY;
	else
		*p = fs;-------add this file type, e.g. usbfs, ext2_fs, etc. If not found, adding it.
	write_unlock(&file_systems_lock);
	return res;
}


static inline void register_as_ext2(void)
{
	int err = register_filesystem(&ext2_fs_type);
	if (err)
		printk(KERN_WARNING
		       "EXT4-fs: Unable to register as ext2 (%d)\n", err);
}



err = register_filesystem(&ext4_fs_type);
ext4_init_fs(void)

static struct file_system_type ext2_fs_type = {
	.owner			= THIS_MODULE,
	.name			= "ext2",
	.init_fs_context	= ext4_init_fs_context,
	.parameters		= ext4_param_specs,
	.kill_sb		= ext4_kill_sb,
	.fs_flags		= FS_REQUIRES_DEV,
};

static struct file_system_type ext4_fs_type = {
	.owner			= THIS_MODULE,
	.name			= "ext4",
	.init_fs_context	= ext4_init_fs_context,
	.parameters		= ext4_param_specs,
	.kill_sb		= ext4_kill_sb,
	.fs_flags		= FS_REQUIRES_DEV | FS_ALLOW_IDMAP,
};


proto_ops:------------------------e.g. .poll  = tcp_poll, if not used/overwrite, then to use below sock_poll.
file_operations socket_file_ops--.poll  = sock_poll,

but in proto_ops.



struct proto_ops {
	int		family;
	struct module	*owner;
	int		(*release)   (struct socket *sock);
	int		(*bind)	     (struct socket *sock,
				      struct sockaddr *myaddr,
				      int sockaddr_len);
	int		(*connect)   (struct socket *sock,
...........................
	int		(*sendmsg)   (struct socket *sock, struct msghdr *m,
				      size_t total_len);
..........................


inet_init:
for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);


static struct inet_protosw inetsw_array[] =
{
	{
		.type =       SOCK_STREAM,
		.protocol =   IPPROTO_TCP,
		.prot =       &tcp_prot,
		.ops =        &inet_stream_ops,
		.flags =      INET_PROTOSW_PERMANENT |
			      INET_PROTOSW_ICSK,
	},



case SYS_SENDTO:
		err = __sys_sendto(a0, (void __user *)a1, a[2], a[3],
				   (struct sockaddr __user *)a[4], a[5]);
-》
static inline int sock_sendmsg_nosec(struct socket *sock, struct msghdr *msg)
{
	int ret = INDIRECT_CALL_INET(READ_ONCE(sock->ops)->sendmsg, inet6_sendmsg,
				     inet_sendmsg, sock, msg,
				     msg_data_left(msg));


=》
int inet_sendmsg(struct socket *sock, struct msghdr *msg, size_t size)
{
	struct sock *sk = sock->sk;

	if (unlikely(inet_send_prepare(sk)))
		return -EAGAIN;

	return INDIRECT_CALL_2(sk->sk_prot->sendmsg, tcp_sendmsg, udp_sendmsg,
			       sk, msg, size);
}
=》tcp_sendmsg_locked(sk, msg, size);
=》
onst struct proto_ops inet_stream_ops = {
	.family		   = PF_INET,
	.owner		   = THIS_MODULE,
	.release	   = inet_release,
	.bind		   = inet_bind,
	.connect	   = inet_stream_connect,
	.socketpair	   = sock_no_socketpair,
	.accept		   = inet_accept,
	.getname	   = inet_getname,
	.poll		   = tcp_poll,
	.ioctl		   = inet_ioctl,
	.gettstamp	   = sock_gettstamp,
	.listen		   = inet_listen,
	.shutdown	   = inet_shutdown,
	.setsockopt	   = sock_common_setsockopt,
	.getsockopt	   = sock_common_getsockopt,
	.sendmsg	   = inet_sendmsg,---------------------------------------------
	.recvmsg	   = inet_recvmsg,
#ifdef CONFIG_MMU
	.mmap		   = tcp_mmap,
#endif
	.splice_eof	   = inet_splice_eof,
	.splice_read	   = tcp_splice_read,
	.read_sock	   = tcp_read_sock,
	.read_skb	   = tcp_read_skb,
	.sendmsg_locked    = tcp_sendmsg_locked,
	.peek_len	   = tcp_peek_len,




https://elixir.bootlin.com/linux/latest/source/fs/proc/inode.c#L586


https://elixir.bootlin.com/linux/latest/source/net/socket.c#L154
/*
 *	Socket files have a set of 'special' operations as well as the generic file ones. These don't appear
 *	in the operation structures but are done directly via the socketcall() multiplexor.
 */

static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.uring_cmd =    io_uring_cmd_sock,
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.splice_write = splice_to_socket,
	.splice_read =	sock_splice_read,
	.splice_eof =	sock_splice_eof,
	.show_fdinfo =	sock_show_fdinfo,
};


static const struct file_operations eventfd_fops = {
#ifdef CONFIG_PROC_FS
	.show_fdinfo	= eventfd_show_fdinfo,
#endif
	.release	= eventfd_release,
	.poll		= eventfd_poll,
	.read_iter	= eventfd_read,
	.write		= eventfd_write,
	.llseek		= noop_llseek,
};

const struct file_operations ext2_dir_operations = {
	.llseek		= generic_file_llseek,
	.read		= generic_read_dir,
	.iterate_shared	= ext2_readdir,
	.unlocked_ioctl = ext2_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= ext2_compat_ioctl,
#endif
	.fsync		= ext2_fsync,
};

https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L1983
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *,
			unsigned int flags);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	unsigned long mmap_supported_flags;
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	void (*splice_eof)(struct file *file);
	int (*setlease)(struct file *, int, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);


https://elixir.bootlin.com/linux/latest/source/fs/proc/inode.c#L586
	if (S_ISREG(inode->i_mode)) {
		inode->i_op = de->proc_iops;
		if (de->proc_ops->proc_read_iter)
			inode->i_fop = &proc_iter_file_ops;
		else
			inode->i_fop = &proc_reg_file_ops;


==vfs


static const struct file_operations proc_reg_file_ops = {
	.llseek		= proc_reg_llseek,
	.read		= proc_reg_read,
	.write		= proc_reg_write,
	.poll		= proc_reg_poll,
	.unlocked_ioctl	= proc_reg_unlocked_ioctl,
	.mmap		= proc_reg_mmap,
	.get_unmapped_area = proc_reg_get_unmapped_area,
	.open		= proc_reg_open,
	.release	= proc_reg_release,
};

static ssize_t proc_reg_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
	struct proc_dir_entry *pde = PDE(file_inode(iocb->ki_filp));
	ssize_t ret;

	if (pde_is_permanent(pde))
		return pde->proc_ops->proc_read_iter(iocb, iter);

	if (!use_pde(pde))
		return -EIO;
	ret = pde->proc_ops->proc_read_iter(iocb, iter);
	unuse_pde(pde);
	return ret;



struct proc_ops {
	unsigned int proc_flags;
	int	(*proc_open)(struct inode *, struct file *);
	ssize_t	(*proc_read)(struct file *, char __user *, size_t, loff_t *);
	ssize_t (*proc_read_iter)(struct kiocb *, struct iov_iter *);



```
- socket + inode association
```
struct socket_alloc {
	struct socket socket;
	struct inode vfs_inode;
};


static struct inode *sock_alloc_inode(struct super_block *sb)
{
	struct socket_alloc *ei;

	ei = alloc_inode_sb(sb, sock_inode_cachep, GFP_KERNEL);
	if (!ei)
		return NULL;
	init_waitqueue_head(&ei->socket.wq.wait);
	ei->socket.wq.fasync_list = NULL;
	ei->socket.wq.flags = 0;

	ei->socket.state = SS_UNCONNECTED;
	ei->socket.flags = 0;
	ei->socket.ops = NULL;
	ei->socket.sk = NULL;
	ei->socket.file = NULL;

	return &ei->vfs_inode;
}


-- register_filesystem to file_systems (e.g. ext2, ext4, socket etc).
socket: https://elixir.bootlin.com/linux/latest/source/net/socket.c#L3283
sock_init(void) -> err = register_filesystem(&sock_fs_type);

static struct file_system_type sock_fs_type = {
	.name =		"sockfs",
	.init_fs_context = sockfs_init_fs_context,
	.kill_sb =	kill_anon_super,
};

static int sockfs_init_fs_context(struct fs_context *fc)
{
	struct pseudo_fs_context *ctx = init_pseudo(fc, SOCKFS_MAGIC);
	if (!ctx)
		return -ENOMEM;
	ctx->ops = &sockfs_ops;
	ctx->dops = &sockfs_dentry_operations;


static const struct super_operations sockfs_ops = {
	.alloc_inode	= sock_alloc_inode,-------------------------------------------------------

	.free_inode	= sock_free_inode,
	.statfs		= simple_statfs,
};

```

- device file for usb

```
==find file path(filetype---usbfs)'s inode, to get file_operations corresponding in device file level 
e.g. /dev/usb/lp0




736  int __init usbfs_init(void)
737  {
738  	int retval;
739  
740  	retval = register_filesystem(&usb_fs_type);----------------------------


551  static struct file_system_type usb_fs_type = {
552  	.owner =	THIS_MODULE,
553  	.name =		"usbfs",------------------------------------------
554  	.get_sb =	usb_get_sb,
555  	.kill_sb =	kill_litter_super,
556  };



411  static struct file_operations default_file_operations = {
412  	.read =		default_read_file,
413  	.write =	default_write_file,
414  	.open =		default_open,
415  	.llseek =	default_file_lseek,
416  };


244  static struct inode *usbfs_get_inode (struct super_block *sb, int mode, dev_t dev)
245  {
246  	struct inode *inode = new_inode(sb);
247  
248  	if (inode) {
249  		inode->i_mode = mode;
250  		inode->i_uid = current->fsuid;
251  		inode->i_gid = current->fsgid;
252  		inode->i_blksize = PAGE_CACHE_SIZE;
253  		inode->i_blocks = 0;
254  		inode->i_atime = inode->i_mtime = inode->i_ctime = CURRENT_TIME;
255  		switch (mode & S_IFMT) {
256  		default:
257  			init_special_inode(inode, mode, dev);
258  			break;
259  		case S_IFREG:
260  			inode->i_fop = &default_file_operations;-------------------------



2438  struct file_system_type {
2439  	const char *name;-------------------------------------
2440  	int fs_flags;
2441  #define FS_REQUIRES_DEV		1
2442  #define FS_BINARY_MOUNTDATA	2
2443  #define FS_HAS_SUBTYPE		4
2444  #define FS_USERNS_MOUNT		8	/* Can be mounted by userns root */
2445  #define FS_DISALLOW_NOTIFY_PERM	16	/* Disable fanotify permission events */
2446  #define FS_ALLOW_IDMAP         32      /* FS has been updated to handle vfs idmappings. */
2447  #define FS_THP_SUPPORT		8192	/* Remove once all fs converted */
2448  #define FS_RENAME_DOES_D_MOVE	32768	/* FS will handle d_move() during rename() internally. */
2449  	int (*init_fs_context)(struct fs_context *);
2450  	const struct fs_parameter_spec *parameters;
2451  	struct dentry *(*mount) (struct file_system_type *, int,
2452  		       const char *, void *);
2453  	void (*kill_sb) (struct super_block *);
2454  	struct module *owner;
2455  	struct file_system_type * next;
2456  	struct hlist_head fs_supers;
2457  
2458  	struct lock_class_key s_lock_key;
2459  	struct lock_class_key s_umount_key;
2460  	struct lock_class_key s_vfs_rename_key;
2461  	struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];
2462  
2463  	struct lock_class_key i_lock_key;
2464  	struct lock_class_key i_mutex_key;
2465  	struct lock_class_key invalidate_lock_key;
2466  	struct lock_class_key i_mutex_dir_key;
2467  };


274  /* SMP-safe */
275  static int usbfs_mknod (struct inode *dir, struct dentry *dentry, int mode,
276  			dev_t dev)
277  {
278  	struct inode *inode = usbfs_get_inode(dir->i_sb, mode, dev);--------------------------
279  	int error = -EPERM;


460  static int fs_create_by_name (const char *name, mode_t mode,
461  			      struct dentry *parent, struct dentry **dentry)
462  {
463  	int error = 0;
464  
465  	/* If the parent is not specified, we create it in the root.
466  	 * We need the root dentry to do this, which is in the super
467  	 * block. A pointer to that is in the struct vfsmount that we
468  	 * have around.
469  	 */
470  	if (!parent ) {
471  		if (usbfs_mount && usbfs_mount->mnt_sb) {
472  			parent = usbfs_mount->mnt_sb->s_root;
473  		}
474  	}
475  
476  	if (!parent) {
477  		dbg("Ah! can not find a parent!");
478  		return -EFAULT;
479  	}
480  
481  	*dentry = NULL;
482  	mutex_lock(&parent->d_inode->i_mutex);
483  	*dentry = lookup_one_len(name, parent, strlen(name));
484  	if (!IS_ERR(dentry)) {
485  		if ((mode & S_IFMT) == S_IFDIR)
486  			error = usbfs_mkdir (parent->d_inode, *dentry, mode);----------------------
487  		else
488  			error = usbfs_create (parent->d_inode, *dentry, mode);---------------------


496  static struct dentry *fs_create_file (const char *name, mode_t mode,
497  				      struct dentry *parent, void *data,
498  				      struct file_operations *fops,
499  				      uid_t uid, gid_t gid)
500  {
501  	struct dentry *dentry;
502  	int error;
503  
504  	dbg("creating file '%s'",name);
505  ------
506  	error = fs_create_by_name (name, mode, parent, &dentry);-------------------------------




617  static void usbfs_add_bus(struct usb_bus *bus)
618  {
619  	struct dentry *parent;
620  	char name[8];
621  	int retval;
622  
623  	/* create the special files if this is the first bus added */
624  	if (num_buses == 0) {
625  		retval = create_special_files();---------------------------------------------
626  		if (retval)
627  			return;
628  	}
629  	++num_buses;
630  
631  	sprintf (name, "%03d", bus->busnum);
632  
633  	parent = usbfs_mount->mnt_sb->s_root;
634  	bus->usbfs_dentry = fs_create_file (name, busmode | S_IFDIR, parent,
635  					    bus, NULL, busuid, busgid);----------------------------------
636  	if (bus->usbfs_dentry == NULL) {



707  static int usbfs_notify(struct notifier_block *self, unsigned long action, void *dev)
708  {
709  	switch (action) {
710  	case USB_DEVICE_ADD:
711  		usbfs_add_device(dev);-----------------------------------------
712  		break;
713  	case USB_DEVICE_REMOVE:
714  		usbfs_remove_device(dev);
715  		break;
716  	case USB_BUS_ADD:
717  		usbfs_add_bus(dev);



728  static struct notifier_block usbfs_nb = {
729  	.notifier_call = 	usbfs_notify,------------------ when to call notifier_call()
730  };



736  int __init usbfs_init(void)
737  {
738  	int retval;
739  
740  	retval = register_filesystem(&usb_fs_type);-------------------------
741  	if (retval)
742  		return retval;
743  
744  	usb_register_notify(&usbfs_nb);------------------------------------
745  
746  	/* create mount point for usbfs */
747  	usbdir = proc_mkdir("usb", proc_bus);--------------------------------------
748  
749  	return 0;
750  }


51  void usb_notify_remove_device(struct usb_device *udev)
52  {
53  	blocking_notifier_call_chain(&usb_notifier_list,----------------------------------notifier_call() be called
54  			USB_DEVICE_REMOVE, udev);
55  }


314  struct usb_device_driver usb_generic_driver = {
315  	.name =	"usb",
316  	.match = usb_generic_driver_match,
317  	.probe = usb_generic_driver_probe,
318  	.disconnect = usb_generic_driver_disconnect,



629  		device_lock(dev);
630  		usb_unbind_interface(dev);-------------------disconnect

hub.c	6228 usb_unbind_and_rebind_marked_interfaces(udev); in usb_reset_device()---------------------


1871  	INIT_WORK(&hub->events, hub_event);-----

 .probe = hub_probe,---
```
