
```
If to display into vga (screen 0xb8000), it shall not use -nographic param in qemu commands.
Refer https://stackoverflow.com/questions/6710555/how-to-use-qemu-to-run-a-non-gui-os-on-the-terminal.

qemu server in aliyun server:
qemu-system-i386 -drive format=raw,file=disk.img -vnc 172.23.245.135:5901
netstat -lntup
tcp        0      0 172.23.245.135:11801    0.0.0.0:*               LISTEN      1289/qemu-system-i3

client on win10 pc:
RealVNC Viewer:
121.43.133.208:11801

```

```
multiple defination:
-CF_GLOBAL      := -g -I${SYS_HEADERS} -ffreestanding -nostdlib -nodefaultlibs -nostdinc -fpack-struct -O0 -m32 -march=i386 -Wall -Werror -pedantic -ansi -std=c99
+CF_GLOBAL      := -g -I${SYS_HEADERS} -ffreestanding -nostdlib -nodefaultlibs -nostdinc -fpack-struct -O0 -m32 -march=i386 -pedantic -Wimplicit-function-declaration -ansi -std=c99 -fcommon

```

```
qemu-system-i386 -drive format=raw,file=disk.img -nographic
qemu-system-x86_64 -drive format=raw,file=disk.img -nographic

```

```
 yum install binutils
 yum install qemu-kvm -- installed but not used.

cd /usr/src && wget http://download.qemu.org/qemu-2.8.0.tar.xz
tar xvJf qemu-2.8.0.tar.xz 
cd qemu-2.8.0
./configure  ---require 2-3 version, not support python3 higher version.

cd /usr/src && wget http://download.qemu.org/qemu-8.0.0.tar.xz

tar xvJf qemu-8.0.0.tar.xz && cd qemu-8.0.0
./configure  
make && make install

--configure failed, then continue


yum install gcc-c++
git clone https://github.com/ninja-build/ninja.git
cd ninja
git checkout release
./configure.py --bootstrap
./ninja --version
1.11.1

cd qemu-8.0.0
./configure  
ERROR: glib-2.56 gthread-2.0 is required to compile QEMU in centos

--issue
yum install glib2 glib2-devel

ERROR: Dependency "pixman-1" not found, tried pkgconfig

yum install pixman-devel


make && make install

then to check qemu- prompt, whether includes qemu-system-i386

```
