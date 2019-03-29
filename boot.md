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

用BISO中断复位磁盘，BIOS中断见BIOS-int.md。

物理内存探测：
e820: int 15h,AX=e820h，3.13内核此处用C语言写的，gcc支持16位汇编。
数据结构如下：
```
#define E820MAP			0x2d0	/* our map */
#define E820MAX			32		/* number of entries in E820MAP */
#define E820NR			0x1e8	/* # entries in E820MAP */
/* 这两个是偏移地址，此时段为0x1020（qemu例子）  */

/* 内存类型 */
#define E820_RAM		1
#define E820_RESERVED	2
#define E820_ACPI		3 /* usable as RAM once ACPI tables have been read */
#define E820_NVS		4

struct e820entry {
	u64 addr;			/* start of memory segment */
	u64 size;			/* size of memory segment */
	u32 type;			/* type of memory segment */
} __attribute__((packed));

struct e820map {
    int nr_map;
	struct e820entry map[E820MAX];
};

```
e801:
mem88:



内核解压到0x100000物理地址上，开始执行startup_32@arch/i386/kernel/head.S。
```
	lgdt boot_gdt_descr - __PAGE_OFFSET

===============
# early boot GDT descriptor (must use 1:1 address mapping)
	.word 0x1e8		# 32 bit align gdt_desc.address
boot_gdt_descr:
	.word __BOOT_DS+7
	.long boot_gdt_table - __PAGE_OFFSET

=================
ENTRY(boot_gdt_table)
	.fill GDT_ENTRY_BOOT_CS,8,0
	.quad 0x00cf9a000000ffff/* kernel 4GB code at 0x00000000 */
	.quad 0x00cf92000000ffff/* kernel 4GB data at 0x00000000 */

```
刚开始这段代码连接到3GB+1MB之上，但是加载到1MB开始的物理地址，所以访存用的地址都需要- PAGE_OFFSET。
链接脚本指出在_end之后的下一个页是pg0，用来处理early boot page tables。

```
ENTRY(startup_32)

/*
 * Set segments to known values.
 */
	cld
	lgdt boot_gdt_descr - __PAGE_OFFSET
	movl $(__BOOT_DS),%eax
	movl %eax,%ds
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs

/*
 * Clear BSS first so that there are no surprises...
 * No need to cld as DF is already clear from cld above...
 */
	xorl %eax,%eax
	movl $__bss_start - __PAGE_OFFSET,%edi
	movl $__bss_stop - __PAGE_OFFSET,%ecx
	subl %edi,%ecx
	shrl $2,%ecx
	rep ; stosl

/*
 * Initialize page tables.  This creates a PDE and a set of page
 * tables, which are located immediately beyond _end.  The variable
 * init_pg_tables_end is set up to point to the first "safe" location.
 * Mappings are created both at virtual address 0 (identity mapping)
 * and PAGE_OFFSET for up to _end+sizeof(page tables)+INIT_MAP_BEYOND_END.
 *
 * Warning: don't use %esi or the stack in this code.  However, %esp
 * can be used as a GPR if you really need it...
 */
page_pde_offset = (__PAGE_OFFSET >> 20);

	movl $(pg0 - __PAGE_OFFSET), %edi
	movl $(swapper_pg_dir - __PAGE_OFFSET), %edx
	movl $0x007, %eax			/* 0x007 = PRESENT+RW+USER */
10:
	leal 0x007(%edi),%ecx			/* Create PDE entry */
	movl %ecx,(%edx)			/* Store identity PDE entry */
	movl %ecx,page_pde_offset(%edx)		/* Store kernel PDE entry */
	addl $4,%edx
	movl $1024, %ecx
11:
	stosl
	addl $0x1000,%eax
	loop 11b
	/* End condition: we must map up to and including INIT_MAP_BEYOND_END */
	/* bytes beyond the end of our own page tables; the +0x007 is the attribute bits */
	leal (INIT_MAP_BEYOND_END+0x007)(%edi),%ebp
	cmpl %ebp,%eax
	jb 10b
	movl %edi,(init_pg_tables_end - __PAGE_OFFSET)

```
这段代码设置最初启动时候的页表。swapper_pg_dir是在.bss.page_aligned段分配的静态空间，用来存放启动阶段的页目录。pg0是链接脚本中定义的,该位置是_end之后的4k对其的地址，其后
用于存放启动时候使用的页表，由于页表的大小跟启动阶段要管理的内存大小相关（从0到页表结束），所以页表的建立比较难懂。最终页表结束位置的物理地址保存在init_pg_tables_end变量
中，该变量在某个C文件中定义。

引导CPU(BSP)把ebx清零，跳过smp代码。
设置cr3位swapper_pg_dir的物理地址，修改cr0启动分页。设置栈空间，该栈分配在swapper_pg_dir之后的一个页，也在.bss.page_aligned段。
清零EFLAGS
设置idt：


====================================================================================
参考资料：ULK（understanding the Linux kernel）的system startup章节讲述了启动过程。
