上电后CS=0xf000,EIP=0xfff0，此后执行BIOS代码，BISO从引导介质中加载bootloader，bootloader加载内核镜像bzImage。
由Makefile可知，bzImage由bootsect、setup和vmlinux.bin构成，bootsect和setup是运行在实模式的代码，vmlinux.bin包括压缩内核及其解压程序。
具体家在过程，见内核文档Documentation/x86/boot.txt。
0203版本的boot protocol中，bootsect和setup被bootloader加载到尽可能低的地址，但是gdb调试的时候不知道该位置，只能用内存搜索的方式，查找"HdrS"字符串的
位置，得知kernel boot sector和setup被加载到0x10000的位置。

(gdb) x /80x 0x101f0
0x101f0:0x00010500 0x0000dda9 0xffff0000 0xaa551601
0x10200:0x64482eeb 0x02035372 0x00000000 0x07b61000
0x10210:0x800081b0 0x00100000 0x07fe0000 0x00000000
0x10220:0x00000000 0x0000fe00 0x00020000 0x1fffffff
0x10230:0x000400e8 0x00000000 0x00000000 0x00000000
0x10240:0x00000000 0x00000000 0x00000000 0x00000000
0x10250:0x00000000 0x00000000 0x00000000 0x00000000
这里有boot header的所有参数。比如0x10228处的四个字节表示cmdline_ptr，即0x20000处存放的是bootloader启动参数。
(gdb) x /s 0x20000
0x20000:"root=/dev/hda"

0x10210处的0xb0表示qemu的BootLoader。
0x10224处的heap_end_ptr
0x10214处的0x100000表示code32_start，内核被加载的位置。
0x10218/4 ramdisk_image: The 32-bit linear address of the initial ramdisk or ramfs.  Leave at zero if there is no initial ramdisk/ramfs.
0x1021C/4 ramdisk_size:  Size of the initial ramdisk or ramfs.  Leave at zero if there is no initial ramdisk/ramfs.

ULK（understanding the Linux kernel）的system startup章节讲述了启动过程。
