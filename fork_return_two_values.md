## fork() child return value eax == 0 when child scheduled  

```
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 */
long do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      struct pt_regs *regs,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr)
{
	struct task_struct *p;
	int trace = 0;


	p = copy_process(clone_flags, stack_start, regs, stack_size,
			 child_tidptr, NULL, trace);
...
		nr = task_pid_vnr(p);
...
...		wake_up_new_task(p);

...
	return nr;---------------------------------------------------------------------------------------------parent process return nr=pid>0 


static struct task_struct *copy_process(unsigned long clone_flags,
					unsigned long stack_start,
					struct pt_regs *regs,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace)

	retval = copy_thread(clone_flags, stack_start, stack_size, p, regs);


==ARM code:
int
copy_thread(unsigned long clone_flags, unsigned long stack_start,
	    unsigned long stk_sz, struct task_struct *p, struct pt_regs *regs)
{
	struct thread_info *thread = task_thread_info(p);
	struct pt_regs *childregs = task_pt_regs(p);

	memset(&thread->cpu_context, 0, sizeof(struct cpu_context_save));

	if (likely(regs)) {
		*childregs = *regs;
		childregs->ARM_r0 = 0;-----------------------------------------eax in arm for fork return, when scheduled to this child.
		childregs->ARM_sp = stack_start;


	thread->cpu_context.pc = (unsigned long)ret_from_fork;-----------------------------------------------ARM
	thread->cpu_context.sp = (unsigned long)childregs;



==x86 code:
int copy_thread(unsigned long clone_flags, unsigned long sp,
	unsigned long arg,
	struct task_struct *p, struct pt_regs *regs)
{
	struct pt_regs *childregs = task_pt_regs(p);
	struct task_struct *tsk;
	int err;

	p->thread.sp = (unsigned long) childregs;
	p->thread.sp0 = (unsigned long) (childregs+1);

	if (unlikely(!regs)) {
		/* kernel thread */
		memset(childregs, 0, sizeof(struct pt_regs));
		p->thread.ip = (unsigned long) ret_from_kernel_thread;
		task_user_gs(p) = __KERNEL_STACK_CANARY;
		childregs->ds = __USER_DS;
		childregs->es = __USER_DS;
		childregs->fs = __KERNEL_PERCPU;
		childregs->bx = sp;	/* function */
		childregs->bp = arg;
		childregs->orig_ax = -1;
		childregs->cs = __KERNEL_CS | get_kernel_rpl();
		childregs->flags = X86_EFLAGS_IF | X86_EFLAGS_BIT1;
		p->fpu_counter = 0;
		p->thread.io_bitmap_ptr = NULL;
		memset(p->thread.ptrace_bps, 0, sizeof(p->thread.ptrace_bps));
		return 0;
	}
	*childregs = *regs;
	childregs->ax = 0;-----------------------------------------eax in arm for fork return, when scheduled to this child.

	childregs->sp = sp;

	p->thread.ip = (unsigned long) ret_from_fork;------------------------------------------------x86 EIP
	task_user_gs(p) = get_user_gs(regs);



ENTRY(ret_from_fork)
	CFI_STARTPROC
	pushl_cfi %eax
	call schedule_tail
	GET_THREAD_INFO(%ebp)
	popl_cfi %eax
	pushl_cfi $0x0202		# Reset kernel eflags
	popfl_cfi
	jmp syscall_exit
	CFI_ENDPROC
END(ret_from_fork)



ARM	Description	x86
R0	General Purpose	EAX----------------
R1-R5	General Purpose	EBX, ECX, EDX, ESI, EDI
R6-R10	General Purpose	–
R11 (FP)	Frame Pointer	EBP
R12	Intra Procedural Call	–
R13 (SP)	Stack Pointer	ESP
R14 (LR)	Link Register	–
R15 (PC)	<- Program Counter / Instruction Pointer ->	EIP
CPSR	Current Program State Register/Flags	EFLAGS
```

```
       CLONE_VM (since Linux 2.0)
              If CLONE_VM is set, the calling process and the child
              process run in the same memory space.

              If CLONE_VM is not set, the child process runs in a
              separate copy of the memory space of the calling process

```
