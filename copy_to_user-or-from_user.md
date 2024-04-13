
从用户空间读取数据，或向用户空间返回数据的系统调用，内部都会有显式或隐式的调用copy_to_user()/copy_from_user()的功能。
e.g.
when send(), the buffer in userspace will be passed into kernel space (even virtual address is the same, but the previlige is not the same, so it shall use copy_to_user from buffer, and also for address if used).
```

int __sys_sendto(int fd, void __user *buff, size_t len, unsigned int flags,
		 struct sockaddr __user *addr,  int addr_len)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err;
	struct msghdr msg;
	struct iovec iov;
	int fput_needed;

	err = import_single_range(WRITE, buff, len, &iov, &msg.msg_iter);
	if (unlikely(err))
		return err;
	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (!sock)
		goto out;

	msg.msg_name = NULL;
	msg.msg_control = NULL;
	msg.msg_controllen = 0;
	msg.msg_namelen = 0;
	msg.msg_ubuf = NULL;
	if (addr) {
		err = move_addr_to_kernel(addr, addr_len, &address);
		if (err < 0)
			goto out_put;
		msg.msg_name = (struct sockaddr *)&address;
		msg.msg_namelen = addr_len;
	}
	if (sock->file->f_flags & O_NONBLOCK)
		flags |= MSG_DONTWAIT;
	msg.msg_flags = flags;
	err = sock_sendmsg(sock, &msg);

out_put:
	fput_light(sock->file, fput_needed);
out:
	return err;
}


==err = import_single_range(WRITE, buff, len, &iov, &msg.msg_iter);

int import_single_range(int rw, void __user *buf, size_t len,
		 struct iovec *iov, struct iov_iter *i)
{
	if (len > MAX_RW_COUNT)
		len = MAX_RW_COUNT;
	if (unlikely(!access_ok(buf, len)))
		return -EFAULT;

	iov->iov_base = buf;----------------------user space buffer address!!!!!!!!!!!!!!!!!!!! similar as copy_from_user(x,x,x)
	iov->iov_len = len;
	iov_iter_init(i, rw, iov, 1, len);
	return 0;
}


```
