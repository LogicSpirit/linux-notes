1. 安装ubuntu 05.04
镜像选择i386虚拟机，创建虚拟磁盘时选择IDE，否则可能不兼容;
在ubuntu 05.04配置old-releases.ubuntu.com源;（可以看看还有没有其他源，#L10）
安装编译工具、Samba等
编译linux-2.6.10和busybox-1.0.0。如果menuconfig时没有ncurses库，安装libncurses5-dev。
把busybox打包成rootfs.img:
```
dd if=/dev/zero of=rootfs.img bs=1M count=5
mkfs.ext3 ./rootfs.img
Mount –t ext3 –o loop ./rootfs.img /path/to/rootfs
Cp busybox/_install/* -R /path/to/rootfs/
Mknod, edit init file etc…
Umount /path/to/rootfs
```
这样rootfs.img就已经是一个有根目录的硬盘镜像了。

2. 安装ubuntu 16.04虚拟机
安装qemu-system-x86；
安装nfs-kernel-server，配置/etc/exports nfs共享目录，用来在两个系统之间传文件。在05.04那边挂载一下。

3. 开搞：
```
qemu-system-i386 -curses -kernel bzImage -hda rootfs.img -append "root=/dev/had" -S -s
```

再开一个终端：
```
Gdb vmlinux
Target remote :1234
```

每次输入两条命令挺烦，可以创建.gdbinit
```
cat >.gdbinit << "EOF"
file vmlinux
target remote localhost:1234
EOF
```
