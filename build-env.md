### 1. 安装ubuntu 05.04
镜像选择i386虚拟机，创建虚拟磁盘时选择IDE，否则可能不兼容;
在ubuntu 05.04，修改/etc/apt/source.list，配置old-releases.ubuntu.com源;
安装编译工具、Samba等
编译linux-2.6.10和busybox-1.0.0。如果make menuconfig时没有ncurses库，安装libncurses5-dev。

##### 把busybox打包成rootfs.img
```
dd if=/dev/zero of=rootfs.img bs=1M count=5
mkfs.ext3 ./rootfs.img
mount -t ext3 -o loop ./rootfs.img /path/to/rootfs
cp busybox/_install/* -R /path/to/rootfs/
mknod, edit init file, etc…
umount /path/to/rootfs （执行时不要在该目录，否则它会busy）
```
这样rootfs.img就已经是一个有根目录的硬盘镜像了。

### 2. 安装ubuntu 16.04虚拟机
安装qemu-system-x86；
安装nfs-kernel-server，配置/etc/exports nfs共享目录，用来在两个系统之间传文件。
在05.04那边挂载一下。(mount -t nfs 187.0.0.45:/nfs /nfs执行时会卡很长时间，所以没有放到启动脚本中#L9)

### 3. 开搞：
```
qemu-system-i386 -curses -kernel bzImage -hda rootfs.img -append "root=/dev/hda earlyprintk=vga" -S -s
```
参数解析：
- `-curses` 利用curses库在终端上模拟VGA显示。
- `-kernel bzImage` 指定内核镜像文件，qemu会像bootloader一样加载镜像到相应的物理地址。
- `-hda rootfs.img` 把制作的rootfs镜像放到虚拟hda磁盘中。
- `-append` 指定bootloader的命令行参数，这个参数qemu bootloader会传递给内核。
- `-S -s` 让qemu启动后停下，并连接本地调试器。


再开一个终端：
```
gdb vmlinux
target remote :1234
```

每次输入两条命令挺烦，可以创建.gdbinit，之后直接gdb就OK了。
```
cat >.gdbinit << "EOF"
file vmlinux
target remote localhost:1234
EOF
```

==========================================================================================
TODO:
- 中断进入qemu curses之后无法正常退出，只能强行关闭。(#L2)
- 启动时的打印信息刷得太快，qemu的输出需要能保存到文档中。(#L3)
尝试过加入-serial file:xxx -append "console=ttyS0"，但是一加入console=ttyS0，启动过程就会卡在Uncompress kernel。
- mount -t nfs 187.0.0.45:/nfs /nfs执行时会卡很长时间，所以没有放到启动脚本中(#L9)
- 目前需要两个虚拟机，不方便。可考虑在ubuntu1604上构建i386的交叉编译环境，编出合适版本的工具链很难。(#L20)
