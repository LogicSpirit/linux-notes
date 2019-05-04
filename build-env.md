### 1. 安装ubuntu-16.04虚拟机
该虚拟机总领全局，用来建立各个项目的workspace，其中包含多个版本的linux和其他GUN软件的源码以及调试环境。但是很多软件的编译环境相当苛刻，难以在此虚拟机上完成编译，所以把特定项目的源码共享给相应的辅助虚拟机完成编译。
调试学习老版本linux的时候，可以找到相当老的RedHat Linux，其内核有1.x,2.x的等等，用来做编译环境。这些编译虚拟机通过nfs访问主虚拟机的项目目录。

需要的配置：
- nfs server
- samba server
- git
- vim
- qemu
- kpartx

```sh
apt-get -y install vim qemu kpartx git samba nfs-kernel-server
cat >> /etc/export << "EOF"
/home/calvin/workspace	*(rw,sync)
EOF
# 打开samba的homes共享
# smbpasswd设置密码

```


### 2. qemu&gdb 用法
调试方法对于多数版本都一样。
```
qemu-system-i386 -curses -kernel bzImage -hda rootfs.img -append "root=/dev/hda earlyprintk=vga" -S -s
```
参数解析：
- `-curses` 利用curses库在终端上模拟VGA显示。
- `-kernel bzImage` 指定内核镜像文件，qemu会像bootloader一样加载镜像到相应的物理地址。
- `-hda rootfs.img` 把制作的rootfs镜像放到虚拟hda磁盘中。
- `-append` 指定bootloader的命令行参数，这个参数qemu bootloader会传递给内核。
- `-S -s` 让qemu启动后停下，并连接本地调试器。

**在xshell中使用curses模拟vga的方式显示，使用<esc>+[1/2]可以切换显示界面和控制台，在控制台下输入quit可退出qemu。**

再开一个终端：
```
gdb vmlinux
target remote :1234
```

每次输入两条命令挺烦，可以创建.gdbinit，之后直接gdb就OK了。
```sh
cat >.gdbinit << "EOF"
file vmlinux
target remote localhost:1234
EOF
```

### 3. linux-2.4.0环境
用RedHat Linux 7.1发行版（不是RedHat Enterprise Linux）。
光盘的软件包中有sshd,samba,nfsd等。
##### 安装注意事项：
1. 创建IDE磁盘，总线选0:0，否则它的编号不是hda，会安装失败。
2. 需要使用两张disc，切换时别忘了连接上。
3. 挂载cdrom，rpm -ivh xxx.rpm安装应用，软件包分布在两个disc上。不会自动安装sshd启动配置，需要`ln -sv ../init.d/sshd /etc/rc3.d/S55sshd`

##### 环境配置
```sh
mkdir -vp /home/calvin/workspace/linux-2.4.0
mkdir -vp /home/calvin/workspace/busybox-0.60.3
mkdir -vp /home/calvin/workspace/grub-0.92
cat >> /etc/rc.local << "EOF"
mount 111.0.0.2:/home/calvin/workspace/linux-2.4.0 /home/calvin/workspace/linux-2.4.0
mount 111.0.0.02:/home/calvin/workspace/busybox-0.60.3 /home/calvin/workspace/busybox-0.60.3
mount 111.0.0.02:/home/calvin/workspace/grub-0.92 /home/calvin/workspace/grub-0.92
EOF
```

##### 编译linux-2.4.0
```sh
cd /home/calvin/workspace/linux-2.4.0
make menuconfig
make bzImage
```
##### 编译busybox-0.60.3
```sh
cd /home/calvin/workspace/busybox-0.60.3
# 这个版本没有menuconfig，要想编译静态连接的程序只能修改Makefile。原理是链接的时候使用--static选项。
sed -e "s/\(^DOSTATIC =\).*$/\1 true/g" -i Makefile
make && make install
```

##### 编译grub-0.92
linux-2.4.0不能直接被qemu-2.5使用-kernel启动，由于qemu内嵌的bootloader不支持x86 boot protocol v2.02，所以才决定在rootfs.img文件中安装grub-0.92来启动内核。
```sh
./configure
make
```

##### 调试
```sh
# @主虚拟机
cd /home/calvin/workspace/qemu-debug

# 创建磁盘镜像rootfs.img-2.4.0
dd if=/dev/zero of=rootfs.img-2.4.0 bs=1M count=100
losetup /dev/loop0 rootfs.img-2.4.0
# 安装grub
dd if=../grub-0.92/stage1/stage1 of=rootfs.img-2.4.0 bs=512 count=1
dd if=../grub-0.92/stage2/stage2 of=rootfs.img-2.4.0 bs=512 seek=1
# 创建分区，一定要在写bootloader之后，否则MBR分区表会被覆盖。这里用默认操作，创建一个从2048扇区(1MB)开始的分区。
fdisk /dev/loop0
kpartx -av rootfs.img-2.4.0		# 会在/dev/mapper/下挂载虚拟硬盘中的分区
mkfs.ext2 /dev/mapper/loop0p1
mkdir -pv /mnt/rootfs-2.4.0
mount /dev/mapper/loop0p1 /mnt/rootfs-2.4.0

mkdir -vp /mnt/rootfs-2.4.0/{boot/grub,dev,sys,proc,etc}
cp -v ../grub-0.92/stage1/stage1 /mnt/rootfs-2.4.0/boot/grub/
cp -v ../grub-0.92/stage2/stage2 /mnt/rootfs-2.4.0/boot/grub/
cp -v ../grub-0.92/stage2/e2fs_stage1_5 /mnt/rootfs-2.4.0/boot/grub/
cat >> /mnt/rootfs-2.4.0/boot/grub/menu.lst << "EOF"
# Begin /boot/grub/menu.lst

# By default boot the first menu entry.
default 0

# Allow 10 seconds before booting the default.
timeout 10

# Use prettier colors.
#color green/black light-green/black

# The first entry is for LFS.
title linux-2.4.0
root (hd0,0)
kernel --no-mem-option /boot/bzImage root=/dev/hda1
EOF

# 安装内核。如果需要安装模块，还要拷贝System.map。
cp -v ../linux-2.4.0/arch/i386/boot/bzImage /mnt/rootfs-2.4.0/boot/

# 安装busybox
cp -vR ../busybox-0.60.3/_install/* /mnt/rootfs-2.4.0/
cp -v ../busybox-0.60.3/scripts/inittab /mnt/rootfs-2.4.0/etc/inittab

# 额外配置
mknod /mnt/rootfs-2.4.0/dev/console c 5 1
mknod /mnt/rootfs-2.4.0/dev/null c 1 3
mknod /mnt/rootfs-2.4.0/dev/hda b 3 0

```

```sh
ln -svf ../linux-2.4.0/vmlinux vmlinux-2.4.0
qemu-system-i386 -curses -hda rootfs.img-2.4.0 -S -s
gdb vmlinux-2.4.0
```
注：
1. 第一次启动的时候会停在grub命令行，先root (hd0,0)再setup (hd0)就行。
2. rootfs.img挂载的时候可以调试。如需卸载:
```sh
umount /mnt/rootfs-2.4.0
kpartx -dv rootfs.img-2.4.0
losetup -d /dev/loop0
```

==========================================================================================
TODO:
- 启动时的打印信息刷得太快，qemu的输出需要能保存到文档中。(#L3)
尝试过加入-serial file:xxx -append "console=ttyS0"，但是一加入console=ttyS0，启动过程就会卡在Uncompress kernel。
