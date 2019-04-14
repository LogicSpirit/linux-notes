### 1. 安装ubuntu-16.04-i386虚拟机
```sh
# run as root
apt-get install -y qemu-system-x86	# 安装qemu-system-x86；
apt-get install -y libncurses5-dev # make menuconfig会用到

echo "deb     http://old-releases.ubuntu.com/ubuntu/ hardy universe" >> /etc/apt/sources.list	#使用0804的源
apt-get update

apt-get install -y gcc-3.4	#安装配置gcc-3.4（gcc-3.3应该也行）
ln -svf gcc-3.4 /usr/bin/gcc

# 先是找不到头文件，这是因为gcc自动的头文件搜索目录是在编译gcc时配置的，用gcc-3.3 -xc -E -v -可以看到。
# 添加环境变量，让gcc能找到头文件和库：
cat >>/etc/bash.bashrc << "EOF"
export C_INCLUDE_PATH=/usr/include/i386-linux-gnu
export LIBRARY_PATH=/usr/lib/i386-linux-gnu
EOF
source /etc/bash.bashrc

# 然后还找不到libgcc_s.so，这个文件是有的，在/lib/i386-linux-gnu/libgcc_s.so.1，只不过原来的链接指错了。
#ln -svf /lib/i386-linux-gnu/libgcc_s.so.1 /usr/lib/gcc-lib/i486-linux-gnu/3.3.6/libgcc_s.so
ln -svf /lib/i386-linux-gnu/libgcc_s.so.1 /usr/lib/gcc/i486-linux-gnu/3.4.6/libgcc_s.so

# 用gcc-3.4编译hello world测试能否正常工作
cat >hello.c <<"EOF"
#include <stdio.h>
int main(){
	printf("hello\n");
	return 0;
}
EOF
gcc hello.c -o hello \
./hello \
rm hello.c hello

# make mrproper时出现找不到头文件，创建个链接
ln -svf /usr/include/i386-linux-gnu/bits /usr/include/bits

# 把linux-2.6.10的内核头文件放到好用的位置
mkdir -pv /usr/src/linux-headers-2.6.10/include
cp -vR /path/to/linux-2.6.10/include/* /usr/src/linux-headers-2.6.10/include
```

### 2. 编译

#### linux-2.6.10
好像有个Makefile中`+=`写成了`+ =`，Make 4.1报错。

make的时候报错：
```
CC arch/i386/kernel/process.o
{standard input}: Assembler messages:
{standard input}:861: Error: suffix or operands invalid for `mov'
{standard input}:862: Error: suffix or operands invalid for `mov'
{standard input}:1055: Error: suffix or operands invalid for `mov'
{standard input}:1056: Error: suffix or operands invalid for `mov'
{standard input}:1122: Error: suffix or operands invalid for `mov'
{standard input}:1123: Error: suffix or operands invalid for `mov'
{standard input}:1190: Error: suffix or operands invalid for `mov'
{standard input}:1191: Error: suffix or operands invalid for `mov'
{standard input}:1300: Error: suffix or operands invalid for `mov'
{standard input}:1312: Error: suffix or operands invalid for `mov'
make[1]: *** [arch/i386/kernel/process.o] B d 1
```
这是因为binutils>2.15，跟段寄存相关的mov指令不能加后缀，否则as过不了（不是gcc的问题）。所以按照linux-2.6-seg-5.patch修改一下代码就OK了。

#### busybox-1.0.0
编译时需要libc，但是1604的libc跟2.6.10并不一致，可能导致问题？运行时发现不能正常使用，难道要另外装低版本libc来编译busybox？(#L3)
暂时先用之前在0504中编译的busybox，目前还不需要调试用户程序。
```sh
make defconfig
make menuconfig		# 在build options中选择编译成静态链接程序；<C-BS>可以删除字符串。
make install
```
编译时需要内核头文件，在Makefile中添加
```
CFLAGS += -I/usr/src/linux-headers-2.6.10/include
```

##### 把busybox打包成rootfs.img
```
dd if=/dev/zero of=rootfs.img bs=1M count=10
mkfs.ext3 rootfs.img
mkdir -v rootfs
sudo mount -t ext3 -o loop rootfs.img rootfs
sudo cp _install/* -R rootfs
sudo mkdir -v rootfs/{dev,proc,sys}
sudo mknod rootfs/dev/console c 5 1
sudo mknod rootfs/dev/null c 1 3
# 挂在proc,sys
sudo umount /path/to/rootfs （执行时不要在该目录，否则它会busy）
```
这样rootfs.img就已经是一个有根目录的硬盘镜像了。


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

**在xshell中只能使用curses模拟vga的方式显示，使用<esc>+[1/2]可以切换显示界面和控制台，在控制台下输入quit可退出qemu。**

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

==========================================================================================
TODO:
- 启动时的打印信息刷得太快，qemu的输出需要能保存到文档中。(#L3)
尝试过加入-serial file:xxx -append "console=ttyS0"，但是一加入console=ttyS0，启动过程就会卡在Uncompress kernel。
