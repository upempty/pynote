To build linux 
```
wget https://mirrors.ustc.edu.cn/lfs/lfs-packages/12.0/linux-6.4.12.tar.xz
tar xvf linux-6.4.12.tar.xz
cd linux-6.4.12
make defconfig
make -j4 (when cpu cores=2)

output:
[root@iZbp1evissossq564e432eZ linux-6.4.12]# ls -lrt arch/x86/boot/bzImage 
-rw-r--r-- 1 root root 12545408 Apr 13 00:02 arch/x86/boot/bzImage
[root@iZbp1evissossq564e432eZ linux-6.4.12]# ls -lrt vmlinux
-rwxr-xr-x 1 root root 85337456 Apr 13 00:01 vmlinux
[root@iZbp1evissossq564e432eZ linux-6.4.12]# 
```

Build kernel modules on running linux system
- sourecode
```
[root@iZbp1evissossq564e432eZ hellow]# ls -lrt
total 8
-rw-r--r-- 1 root root 155 Apr 14 23:13 Makefile
-rw-r--r-- 1 root root 336 Apr 14 23:17 hello.c
[root@iZbp1evissossq564e432eZ hellow]# cat hello.c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");

static int __init module_start(void)

{
    printk(KERN_INFO "Hello, WorldMe!\n");
    return 0;
}


static void __exit module_end(void)
{
    printk(KERN_INFO "Bye, WorldMe!\n");
}

module_init(module_start);
module_exit(module_end);
[root@iZbp1evissossq564e432eZ hellow]# cat Makefile

obj-m += hello.o
all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

```
- build  
make
- kernel install/uninstall  
  ismod hello.ko/rmmod hello(rmmod hello.ko)  
  dmesg  

How to add kernel module to linux x86 source code?  
