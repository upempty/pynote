- bin file to be loaded for executing

```
cycle the list of binary formats handler, until one recognizes the image via search_binary_handler

where to find formats?
static LIST_HEAD(formats);--- struct list_head head
--#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)----#define LIST_HEAD_INIT(name) { &(name), &(name) }

static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}


==https://elixir.bootlin.com/linux/latest/source/fs/binfmt_elf.c#L101
static struct linux_binfmt elf_format = {
	.module		= THIS_MODULE,
	.load_binary	= load_elf_binary,---------------includes ...START_THREAD(elf_ex, regs, elf_entry, bprm->p);....

static int __init init_elf_binfmt(void)
{
	register_binfmt(&elf_format);
	return 0;
}

core_initcall(init_elf_binfmt);
module_exit(exit_elf_binfmt);

==https://elixir.bootlin.com/linux/latest/source/fs/binfmt_flat.c
static struct linux_binfmt flat_format = {
	.module		= THIS_MODULE,
	.load_binary	= load_flat_binary,
};

static int __init init_flat_binfmt(void)
{
	register_binfmt(&flat_format);
	return 0;
}
core_initcall(init_flat_binfmt);


==registers elf, flat ect to target: formats

static inline void register_binfmt(struct linux_binfmt *fmt)
{
	__register_binfmt(fmt, 0);
}

void __register_binfmt(struct linux_binfmt * fmt, int insert)
{
	write_lock(&binfmt_lock);
	insert ? list_add(&fmt->lh, &formats) :
		 list_add_tail(&fmt->lh, &formats);
	write_unlock(&binfmt_lock);
}


==https://github.com/torvalds/linux/blob/39cd87c4eb2b893354f3b850f916353f2658ae6f/fs/exec.c#L1773
/*
 * cycle the list of binary formats handler, until one recognizes the image
 */
static int search_binary_handler(struct linux_binprm *bprm)
{
	bool need_retry = IS_ENABLED(CONFIG_MODULES);
	struct linux_binfmt *fmt;
	int retval;

	retval = prepare_binprm(bprm);
	if (retval < 0)
		return retval;

	retval = security_bprm_check(bprm);
	if (retval)
		return retval;

	retval = -ENOENT;
 retry:
	read_lock(&binfmt_lock);
	list_for_each_entry(fmt, &formats, lh) {
		if (!try_module_get(fmt->module))
			continue;
		read_unlock(&binfmt_lock);

		retval = fmt->load_binary(bprm);

		read_lock(&binfmt_lock);
		put_binfmt(fmt);
		if (bprm->point_of_no_return || (retval != -ENOEXEC)) {
			read_unlock(&binfmt_lock);
			return retval;
		}
	}


static int load_elf_binary(struct linux_binprm *bprm)--------------
{
	struct file *interpreter = NULL; /* to shut gcc up */
	unsigned long load_bias = 0, phdr_addr = 0;
	int first_pt_load = 1;
	unsigned long error;
	struct elf_phdr *elf_ppnt, *elf_phdata, *interp_elf_phdata = NULL;
	struct elf_phdr *elf_property_phdata = NULL;
	unsigned long elf_brk;
	int retval, i;
	unsigned long elf_entry;
	unsigned long e_entry;
	unsigned long interp_load_addr = 0;
	unsigned long start_code, end_code, start_data, end_data;
	unsigned long reloc_func_desc __maybe_unused = 0;
	int executable_stack = EXSTACK_DEFAULT;
	struct elfhdr *elf_ex = (struct elfhdr *)bprm->buf;-------------------
	struct elfhdr *interp_elf_ex = NULL;
	struct arch_elf_state arch_state = INIT_ARCH_ELF_STATE;
	struct mm_struct *mm;
	struct pt_regs *regs;

	retval = -ENOEXEC;
	/* First of all, some simple consistency checks */
	if (memcmp(elf_ex->e_ident, ELFMAG, SELFMAG) != 0)
	------------checks inside of iterate/traverse fmt->load_binary(bprm) with load_elf_binary e.g. , is it ELF format==#define	ELFMAG		"\177ELF"
		goto out;
	---------if not ELF format, then exit to loop check next format via list_for_each_entry(fmt, &formats, lh)
    ---------e.g to fmt->load_binary(bprm) with load_flat_binary
	
	
```
