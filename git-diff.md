- https://github.com/chobits/tinyos/tree/b5db0bfc78fbcac4bf69644a0d69feb63ff21213

```

[root@VM-16-14-centos tinyos]# git status
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   Makefile
        modified:   fs/file.c
        modified:   kernel/execute.c
        modified:   kernel/schedule.c
        modified:   user/Makefile
        modified:   user/init.c
        modified:   user/libc/print.c

no changes added to commit (use "git add" and/or "git commit -a")
[root@VM-16-14-centos tinyos]# git diff
diff --git a/Makefile b/Makefile
index 48e5ad7..706c47a 100644
--- a/Makefile
+++ b/Makefile
@@ -11,7 +11,7 @@ MKFS  = tools/mkfs.minix
 CTAGS  = ctags
 OBJDUMP        = objdump
 OBJCOPY        = objcopy
-CFLAGS = -Wall -Werror -fno-builtin -nostdinc -nostdlib -I../include -g -m32
+CFLAGS = -Wall -fno-builtin -nostdinc -nostdlib -I../include -g -m32
 LDFLAGS = -m elf_i386
 export Q LD AS CC NM OBJDUMP OBJCOPY CFLAGS LDFLAGS

diff --git a/fs/file.c b/fs/file.c
index 6eb487d..509386d 100644
--- a/fs/file.c
+++ b/fs/file.c
@@ -5,7 +5,7 @@
 #include <file.h>
 #include <fs.h>

-struct slab *file_slab;
+extern struct slab *file_slab;

 static void free_file(struct file *file)
 {
diff --git a/kernel/execute.c b/kernel/execute.c
index 3566b1d..c209455 100644
--- a/kernel/execute.c
+++ b/kernel/execute.c
@@ -211,7 +211,7 @@ err_out:
        return -1;
 }

-pde_t *kern_page_dir;
+extern pde_t *kern_page_dir;
 int create_userspace(struct task *t, struct file *file, struct elf *elf)
 {
        struct userspace *us = &t->us;
diff --git a/kernel/schedule.c b/kernel/schedule.c
index 355079c..dd16cc5 100644
--- a/kernel/schedule.c
+++ b/kernel/schedule.c
@@ -3,8 +3,8 @@
 #include <mm.h>
 #include <print.h>

-struct list_head task_list;
-struct task init_task;
+//struct list_head task_list;
+extern struct task init_task;
 static struct task dummy_task; /* for switching to the same ctack */

 /* It is safe to switch to current task, which is not recommended. */
diff --git a/user/Makefile b/user/Makefile
index 44a68c9..c9c5d0f 100644
--- a/user/Makefile
+++ b/user/Makefile
@@ -1,5 +1,5 @@
 SUBDIR = user
-CFLAGS = -Wall -Werror -fno-builtin -nostdinc -nostdlib -Iinclude -g -m32
+CFLAGS = -Wall -fno-builtin -nostdinc -nostdlib -Iinclude -g -m32
 export SUBDIR CFLAGS
 APPS   = $(patsubst user/%, %, $(USER_APPS))
 ASMS   = $(patsubst %, %.asm, $(APPS))
diff --git a/user/init.c b/user/init.c
index c404882..1dbb797 100644
--- a/user/init.c
+++ b/user/init.c
@@ -37,8 +37,10 @@ void test_fork(void)
                        iprintf("Create child %d\n", pid);
                }
        }
-       while (1); yield();
-               //iprintf("> pid %d <\n", getpid());
+       while (1) {
+           yield();
+            //iprintf("> pid %d <\n", getpid());
+       }

 }

@@ -81,7 +83,7 @@ refork_child:
                goto refork;
        }

-       while (1) ;
+       while (1);

 }

diff --git a/user/libc/print.c b/user/libc/print.c
index 569d410..43dc34e 100644
--- a/user/libc/print.c
+++ b/user/libc/print.c
@@ -104,9 +104,9 @@ static char ustrbuf[0x1000];
 int printf(char *format, ...)
 {
        va_list args;
-       int i;
+       //int i;
        va_start(args, format);
-       i = vsprintf(ustrbuf, format, args);
+       //int i = vsprintf(ustrbuf, format, args);
        va_end(args);
        return usys_puts(ustrbuf);
 }
(END)
 SUBDIR = user
-CFLAGS = -Wall -Werror -fno-builtin -nostdinc -nostdlib -Iinclude -g -m32
+CFLAGS = -Wall -fno-builtin -nostdinc -nostdlib -Iinclude -g -m32
 export SUBDIR CFLAGS
 APPS   = $(patsubst user/%, %, $(USER_APPS))
 ASMS   = $(patsubst %, %.asm, $(APPS))
diff --git a/user/init.c b/user/init.c
index c404882..1dbb797 100644
--- a/user/init.c
+++ b/user/init.c
@@ -37,8 +37,10 @@ void test_fork(void)
                        iprintf("Create child %d\n", pid);
                }
        }
-       while (1); yield();
-               //iprintf("> pid %d <\n", getpid());
+       while (1) {
+           yield();
+            //iprintf("> pid %d <\n", getpid());
+       }

 }

@@ -81,7 +83,7 @@ refork_child:
                goto refork;
        }

-       while (1) ;
+       while (1);

 }

diff --git a/user/libc/print.c b/user/libc/print.c
index 569d410..43dc34e 100644
--- a/user/libc/print.c
+++ b/user/libc/print.c
@@ -104,9 +104,9 @@ static char ustrbuf[0x1000];
 int printf(char *format, ...)
 {
        va_list args;
-       int i;
+       //int i;
        va_start(args, format);
-       i = vsprintf(ustrbuf, format, args);
+       //int i = vsprintf(ustrbuf, format, args);
        va_end(args);
        return usys_puts(ustrbuf);
 }

[1]+  Stopped                 git diff

```
