- socket recv
```


recv(client_sock, client_message, sizeof(client_message), 0) < 0)

glibc:
extern ssize_t recv (int __fd, void *__buf, size_t __n, int __flags);
https://elixir.bootlin.com/glibc/latest/source/socket/sys/socket.h#L138

https://github.com/lattera/glibc/blob/master/sysdeps/unix/sysv/linux/recv.c
ssize_t
__libc_recv (int fd, void *buf, size_t len, int flags)
{
#ifdef __ASSUME_RECV_SYSCALL
  return SYSCALL_CANCEL (recv, fd, buf, len, flags);
#elif defined __ASSUME_RECVFROM_SYSCALL
  return SYSCALL_CANCEL (recvfrom, fd, buf, len, flags, NULL, NULL);
#else
  return SOCKETCALL_CANCEL (recv, fd, buf, len, flags);
#endif
}
weak_alias (__libc_recv, recv)
weak_alias (__libc_recv, __recv)


SYSCALL_CANCEL (recv, fd, buf, len, flags);

#define SYSCALL_CANCEL(...) \
  ({									     \
    long int sc_ret;							     \
    if (NO_SYSCALL_CANCEL_CHECKING)					     \
      sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__); 			     \
    else								     \
      {									     \
	int sc_cancel_oldtype = LIBC_CANCEL_ASYNC ();			     \
	sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__);			     \
        LIBC_CANCEL_RESET (sc_cancel_oldtype);				     \
      }									     \
    sc_ret;								     \
  })


#define INLINE_SYSCALL_CALL(...) \
  __INLINE_SYSCALL_DISP (__INLINE_SYSCALL, __VA_ARGS__)

#define __INLINE_SYSCALL_DISP(b,...) \
  __SYSCALL_CONCAT (b,__INLINE_SYSCALL_NARGS(__VA_ARGS__))(__VA_ARGS__)

#define __INLINE_SYSCALL_NARGS(...) \
  __INLINE_SYSCALL_NARGS_X (__VA_ARGS__,7,6,5,4,3,2,1,0,)

#define __INLINE_SYSCALL_NARGS_X(a,b,c,d,e,f,g,h,n,...) n


#define __SYSCALL_CONCAT(a,b)       __SYSCALL_CONCAT_X (a, b)
#define __SYSCALL_CONCAT_X(a,b)     a##b

__INLINE_SYSCALL0(recv, fd,......................................)

#define __INLINE_SYSCALL0(name) \
  INLINE_SYSCALL (name, 0)

#define INLINE_SYSCALL(name, nr, args...) __syscall_##name (args)---XXXXXXXXXXXXXXX


#define INLINE_SYSCALL(name, nr, args...)				\
  ({									\
    long int sc_ret = INTERNAL_SYSCALL (name, nr, args);		\
    __glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (sc_ret))		\
    ? SYSCALL_ERROR_LABEL (INTERNAL_SYSCALL_ERRNO (sc_ret))		\
    : sc_ret;								\
  })

https://elixir.bootlin.com/glibc/glibc-2.35/source/sysdeps/unix/sysv/linux/sysdep.h#L42
INTERNAL_SYSCALL (name, nr, args);

#define INTERNAL_SYSCALL(name, nr, args...)				\
	internal_syscall##nr (SYS_ify (name), args)

or
	INTERNAL_SYSCALL_RAW(SYS_ify(name), nr, args)

ARM/x86: #define SYS_ify(syscall_name)	(__NR_##syscall_name)
=>__NR_recv
#define __NR_recv 291



#undef internal_syscall0
#define internal_syscall0(number, dummy...)				\
({									\
    unsigned long int resultvar;					\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number)							\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})

or


#if defined(__thumb__)
/* We can not expose the use of r7 to the compiler.  GCC (as
   of 4.5) uses r7 as the hard frame pointer for Thumb - although
   for Thumb-2 it isn't obviously a better choice than r11.
   And GCC does not support asms that conflict with the frame
   pointer.

   This would be easier if syscall numbers never exceeded 255,
   but they do.  For the moment the LOAD_ARGS_7 is sacrificed.
   We can't use push/pop inside the asm because that breaks
   unwinding (i.e. thread cancellation) for this frame.  We can't
   locally save and restore r7, because we do not know if this
   function uses r7 or if it is our caller's r7; if it is our caller's,
   then unwinding will fail higher up the stack.  So we move the
   syscall out of line and provide its own unwind information.  */
# undef INTERNAL_SYSCALL_RAW
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


arm:
ENTRY (__libc_do_syscall)
	.fnstart
	push	{r7, lr}
	.save	{r7, lr}
	cfi_adjust_cfa_offset (8)
	cfi_rel_offset (r7, 0)
	cfi_rel_offset (lr, 4)
	mov	r7, ip
	swi	0x0
	pop	{r7, pc}
	.fnend
END (__libc_do_syscall)

!!!r7---system call number
swi 0x0


https://stackoverflow.com/questions/12946958/what-is-the-interface-for-arm-system-calls-and-where-is-it-defined-in-the-linux

So it depends whether the system uses OABI or EABI.

So in EABI you use r7 to pass the system call number, use r0-r6 to pass the arguments, use SWI 0 to make the system call, expect the result in r0.

In OABI everything is the same except you use SWI <number> to make a system call.

More generic answer than what you asked.

On Linux the man syscall (2) is a good start to find out how to make a system call in various architectures.

Copied from that manpage:

Architecture calling conventions
    Every architecture has its own way of invoking and passing arguments
    to the kernel.  The details for various architectures are listed in
    the two tables below.

    The first table lists the instruction used to transition to kernel
    mode (which might not be the fastest or best way to transition to the
    kernel, so you might have to refer to vdso(7)), the register used to
    indicate the system call number, the register used to return the sys‐
    tem call result, and the register used to signal an error.

    arch/ABI    instruction           syscall #  retval  error    Notes
    ────────────────────────────────────────────────────────────────────
    alpha       callsys               v0         a0      a3       [1]
    arc         trap0                 r8         r0      -
    arm/OABI    swi NR                -          a1      -        [2]
    arm/EABI    swi 0x0               r7         r0      -
    arm64       svc #0                x8         x0      -
    blackfin    excpt 0x0             P0         R0      -
    i386        int $0x80             eax        eax     -
    ia64        break 0x100000        r15        r8      r10      [1]
    m68k        trap #0               d0         d0      -
    microblaze  brki r14,8            r12        r3      -
    mips        syscall               v0         v0      a3       [1]
    nios2       trap                  r2         r2      r7
    parisc      ble 0x100(%sr2, %r0)  r20        r28     -
    powerpc     sc                    r0         r3      r0       [1]
    riscv       scall                 a7         a0      -
    s390        svc 0                 r1         r2      -        [3]
    s390x       svc 0                 r1         r2      -        [3]
    superh      trap #0x17            r3         r0      -        [4]
    sparc/32    t 0x10                g1         o0      psr/csr  [1]
    sparc/64    t 0x6d                g1         o0      psr/csr  [1]
    tile        swint1                R10        R00     R01      [1]
    x86-64      syscall               rax        rax     -        [5]
    x32         syscall               rax        rax     -        [5]
    xtensa      syscall               a2         a2      -

    Notes:

        [1] On a few architectures, a register is used as a boolean (0
            indicating no error, and -1 indicating an error) to signal
            that the system call failed.  The actual error value is still
            contained in the return register.  On sparc, the carry bit
            (csr) in the processor status register (psr) is used instead
            of a full register.

        [2] NR is the system call number.

        [3] For s390 and s390x, NR (the system call number) may be passed
            directly with svc NR if it is less than 256.

        [4] On SuperH, the trap number controls the maximum number of
            arguments passed.  A trap #0x10 can be used with only 0-argu‐
            ment system calls, a trap #0x11 can be used with 0- or
            1-argument system calls, and so on up to trap #0x17 for
            7-argument system calls.

        [5] The x32 ABI uses the same instruction as the x86-64 ABI and
            is used on the same processors.  To differentiate between
            them, the bit mask __X32_SYSCALL_BIT is bitwise-ORed into the
            system call number for system calls under the x32 ABI.  Both
            system call tables are available though, so setting the bit
            is not a hard requirement.




In ARM world, you do a software interrupt (mechanism to signal the kernel) by supervisor call / svc (previously called SWI).

ARM assembly (UAL) syntax looks like this:

SVC{<c>}{<q>} {#}<imm>

=sw
send()->glibc(send)-> SEND_NR, int 0x80(system call)/swi/swc 0 intrrupt.
=hardware
CPU response: save pc, registers, find 0x80 IDT entry:get previlleage , has authorize to trigger this interrupt, switch to kernel mode,
ISR address assign to PC of CPU.
--> IDT descriptor entry(0x80th) on LIDT
--> selector(GDT/LDT's x-th entry->GDT==kernel code base address, and limit.
--> offset, then offset + base address, then it is Destination code segment = syscall
=sw
then jump to OS code of ISR syscall entry.

ENTRY (syscall).
-save registers if possible.
-.................

syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
		ret = __invoke_syscall(regs, syscall_fn);

sys_recv = SYSCALL_DEFINE4(recv,..)

SYSCALL_DEFINE4(recv, int, fd, void __user *, ubuf, size_t, size,
		unsigned int, flags)
{
	return sys_recvfrom(fd, ubuf, size, flags, NULL, NULL);
}
sys_recvfrom



/*
 *	Receive a datagram from a socket.
 */

SYSCALL_DEFINE4(recv, int, fd, void __user *, ubuf, size_t, size,
		unsigned int, flags)
{
	return sys_recvfrom(fd, ubuf, size, flags, NULL, NULL);
}




static void el0_svc_common(struct pt_regs *regs, int scno, int sc_nr,
			   const syscall_fn_t syscall_table[])
{
	unsigned long flags = read_thread_flags();

	regs->orig_x0 = regs->regs[0];
	regs->syscallno = scno;

	/*
	 * BTI note:
	 * The architecture does not guarantee that SPSR.BTYPE is zero
	 * on taking an SVC, so we could return to userspace with a
	 * non-zero BTYPE after the syscall.
	 *
	 * This shouldn't matter except when userspace is explicitly
	 * doing something stupid, such as setting PROT_BTI on a page
	 * that lacks conforming BTI/PACIxSP instructions, falling
	 * through from one executable page to another with differing
	 * PROT_BTI, or messing with BTYPE via ptrace: in such cases,
	 * userspace should not be surprised if a SIGILL occurs on
	 * syscall return.
	 *
	 * So, don't touch regs->pstate & PSR_BTYPE_MASK here.
	 * (Similarly for HVC and SMC elsewhere.)
	 */

	if (flags & _TIF_MTE_ASYNC_FAULT) {
		/*
		 * Process the asynchronous tag check fault before the actual
		 * syscall. do_notify_resume() will send a signal to userspace
		 * before the syscall is restarted.
		 */
		syscall_set_return_value(current, regs, -ERESTARTNOINTR, 0);
		return;
	}

	if (has_syscall_work(flags)) {
		/*
		 * The de-facto standard way to skip a system call using ptrace
		 * is to set the system call to -1 (NO_SYSCALL) and set x0 to a
		 * suitable error code for consumption by userspace. However,
		 * this cannot be distinguished from a user-issued syscall(-1)
		 * and so we must set x0 to -ENOSYS here in case the tracer doesn't
		 * issue the skip and we fall into trace_exit with x0 preserved.
		 *
		 * This is slightly odd because it also means that if a tracer
		 * sets the system call number to -1 but does not initialise x0,
		 * then x0 will be preserved for all system calls apart from a
		 * user-issued syscall(-1). However, requesting a skip and not
		 * setting the return value is unlikely to do anything sensible
		 * anyway.
		 */
		if (scno == NO_SYSCALL)
			syscall_set_return_value(current, regs, -ENOSYS, 0);
		scno = syscall_trace_enter(regs);
		if (scno == NO_SYSCALL)
			goto trace_exit;
	}

	invoke_syscall(regs, scno, sc_nr, syscall_table);

void do_el0_svc(struct pt_regs *regs)
{
	el0_svc_common(regs, regs->regs[8], __NR_syscalls, sys_call_table);
}




static void noinstr el0_svc(struct pt_regs *regs)
{
	enter_from_user_mode(regs);
	cortex_a76_erratum_1463225_svc_handler();
	fp_user_discard();
	local_daif_restore(DAIF_PROCCTX);
	do_el0_svc(regs);
	exit_to_user_mode(regs);
}


asmlinkage void noinstr el0t_64_sync_handler(struct pt_regs *regs)
{
	unsigned long esr = read_sysreg(esr_el1);

	switch (ESR_ELx_EC(esr)) {
	case ESR_ELx_EC_SVC64:
		el0_svc(regs);

https://notes.z-dd.online/2021/10/23/Linux%E4%B9%8B%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/index.html



sys_recvfrom ->  __sys_recvfrom
-> 
sock = sockfd_lookup_light(fd, &err, &fput_needed);
err = sock_recvmsg(sock, &msg, flags);
->
	int err = security_socket_recvmsg(sock, msg, msg_data_left(msg), flags);
	return err ?: sock_recvmsg_nosec(sock, msg, flags);
->
-LSM_HOOK_INIT(socket_recvmsg, selinux_socket_recvmsg),
-inet6_recvmsg, inet_recvmsg, sock, msg,

->
	err = INDIRECT_CALL_2(sk->sk_prot->recvmsg, tcp_recvmsg, udp_recvmsg,
			      sk, msg, size, flags, &addr_len);


	lock_sock(sk);
	ret = tcp_recvmsg_locked(sk, msg, len, flags, &tss, &cmsg_flags);
	release_sock(sk);

->
skb_queue_walk(&sk->sk_receive_queue, skb)
err = skb_copy_datagram_msg(skb, offset, msg, used);

skb_copy_datagram_iter(from, offset, &msg->msg_iter, size);
->__skb_datagram_iter

```
