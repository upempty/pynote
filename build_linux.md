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
