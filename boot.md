上电后`CS=0xf000,EIP=0xfff0`，此后执行BIOS代码，BISO从引导介质中加载bootloader，bootloader加载内核镜像`bzImage`。
由Makefile可知，`bzImage`由`bootsect`、`setup`和`vmlinux.bin`构成，`bootsect`和`setup`是运行在实模式的代码，`vmlinux.bin`包括压缩内核及其解压程序。
具体家在过程，见内核文档`Documentation/x86/boot.txt`。
`0203`版本的`boot protocol`中，`bootsect`和`setup`被`bootloader`加载到尽可能低的地址，但是gdb调试的时候不知道该位置，只能用内存搜索的方式，查找`HdrS`字符串的
位置，得知`bootsect`和`setup`被加载到`0x10000`的位置。

```
(gdb) x /80x 0x101f0
0x101f0:0x00010500 0x0000dda9 0xffff0000 0xaa551601
0x10200:0x64482eeb 0x02035372 0x00000000 0x07b61000
0x10210:0x800081b0 0x00100000 0x07fe0000 0x00000000
0x10220:0x00000000 0x0000fe00 0x00020000 0x1fffffff
0x10230:0x000400e8 0x00000000 0x00000000 0x00000000
0x10240:0x00000000 0x00000000 0x00000000 0x00000000
0x10250:0x00000000 0x00000000 0x00000000 0x00000000
```
这里有boot header的所有参数。比如0x10228处的四个字节表示cmdline_ptr，即0x20000处存放的是bootloader启动参数。
```
(gdb) x /s 0x20000
0x20000:"root=/dev/hda"
```

`0x10210`处的`0xb0`表示qemu的BootLoader。
`0x10224`处的`heap_end_ptr`
`0x10214`处的`0x100000`表示`code32_start`，内核被加载的位置。
`0x10218/4` `ramdisk_image`: The 32-bit linear address of the initial ramdisk or ramfs.  Leave at zero if there is no initial ramdisk/ramfs.
`0x1021C/4` `ramdisk_size`:  Size of the initial ramdisk or ramfs.  Leave at zero if there is no initial ramdisk/ramfs.

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

`vmlinux.bin`被`BootLoader`加载到`0x100000`的物理地址上，该地址写在`setup`的`header`中，`setup`最后会跳转到这里。
`vmlinux.bin`由`arch/i386/boot/compressed/vmlinux`经过`objcopy`得到，而`compressed/vmlinux`由`compressed`下的`head.S、misc.S、piggy.S`构成。`piggy.S`其实包含着顶层`vmlinux`压缩后的二进制数据。
`startup_32@compressed/head.S`是程序入口，这里的函数名跟`vmlinux`中的重复并没有关系。主要完成内核代码的解压缩，并跳转到`vmlinux`。`vmlinux`的链接脚本是`arch/i386/kernel/vmlinux.lds`，其中定义的内核的段结构和程序入口。

内核解压到`0x100000`物理地址上，开始执行`startup_32@arch/i386/kernel/head.S`。
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
刚开始这段代码连接到`3GB+1MB`之上，但是加载到`1MB`开始的物理地址，所以访存用的地址都需要`-PAGE_OFFSET`。
链接脚本指出在`_end`之后的下一个页是`pg0`，其之后的几个页帧用来处理`early boot page tables`。

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
这段代码设置最初启动时候的页表。`swapper_pg_dir`是在`.bss.page_aligned`段分配的静态空间，用来存放启动阶段的页目录。`pg0`是链接脚本中定义的,该位置是`_end`之后的4k对其的地址，其后 用于存放启动时候使用的页表，由于页表的大小跟启动阶段要管理的内存大小相关（从0到页表结束），所以页表的建立比较难懂。最终页表结束位置的物理地址保存在`init_pg_tables_end`变量 中，该变量在某个C文件中定义。在2.6.10内核中该地址不超过`8M`，即启动时才用了两张页表。

引导CPU(`BSP`)把`ebx`清零，跳过`smp`代码。
Enable paging:设置`cr3`位`swapper_pg_dir`的物理地址，修改`cr0`启动分页。设置栈空间，该栈分配在`swapper_pg_dir`之后的一个页，也在`.bss.page_aligned`段。
设置stack:`lss stack_start,%esp`，该指令把stack_start处存放的selector:offset加载到ss:esp。init_thread_union是一个栈大小的union，其低地址处存放的是一个sturct thread_info。所以任何时候只要把esp对齐到栈大小就能得到当前thread的thread_info指针。
```
.data
ENTRY(stack_start)
	.long init_thread_union+THREAD_SIZE
	.long __BOOT_DS

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/*
 * Initial thread structure.
 *
 * We need to make sure that this is THREAD_SIZE aligned due to the
 * way process stacks are handled. This is done by having a special
 * "init_task" linker map entry..
 */
union thread_union init_thread_union 
	__attribute__((__section__(".data.init_task"))) =
		{ INIT_THREAD_INFO(init_task) };

/*
 * Initial task structure.
 *
 * All other task structs will be allocated on slabs in fork.c
 */
struct task_struct init_task = INIT_TASK(init_task);

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
#define INIT_THREAD_INFO(tsk)			\
{						\
	.task		= &tsk,			\
	.exec_domain	= &default_exec_domain,	\
	.flags		= 0,			\
	.cpu		= 0,			\
	.preempt_count	= 1,			\
	.addr_limit	= KERNEL_DS,		\
	.restart_block = {			\
		.fn = do_no_restart_syscall,	\
	},					\
}


```

清零`EFLAGS`
`setup_idt`：`idt_table[256]`数组在`trap.c`中定义，放置在`.data.idt`段。`setup_idt`在`idt_table`中写入默认表项，所有的中断处理过程都是`ignore_int`，只是`printk`打印一些参数。
把启动参数从`bootloader`存放的位置复制到`boot_params`数组中，`boot_params`在`.init.data`段。其中启动命令行被保存到`saved_command_line`中，是`BootLoader`传给内核的启动字符串。
`checkCPUtype`:(#L5)
`check_x87`:(#L6)
加载新的`gdt`和`idt`:使用`cpu_gdt_table`，有`32`个表项，进行平坦地址映射。`idt`没变化。
```
ENTRY(cpu_gdt_table)
	.quad 0x0000000000000000	/* NULL descriptor */
	.quad 0x0000000000000000	/* 0x0b reserved */
	.quad 0x0000000000000000	/* 0x13 reserved */
	.quad 0x0000000000000000	/* 0x1b reserved */
	.quad 0x0000000000000000	/* 0x20 unused */
	.quad 0x0000000000000000	/* 0x28 unused */
	.quad 0x0000000000000000	/* 0x33 TLS entry 1 */
	.quad 0x0000000000000000	/* 0x3b TLS entry 2 */
	.quad 0x0000000000000000	/* 0x43 TLS entry 3 */
	.quad 0x0000000000000000	/* 0x4b reserved */
	.quad 0x0000000000000000	/* 0x53 reserved */
	.quad 0x0000000000000000	/* 0x5b reserved */

	.quad 0x00cf9a000000ffff	/* 0x60 kernel 4GB code at 0x00000000 */
	.quad 0x00cf92000000ffff	/* 0x68 kernel 4GB data at 0x00000000 */
	.quad 0x00cffa000000ffff	/* 0x73 user 4GB code at 0x00000000 */
	.quad 0x00cff2000000ffff	/* 0x7b user 4GB data at 0x00000000 */

	.quad 0x0000000000000000	/* 0x80 TSS descriptor */
	.quad 0x0000000000000000	/* 0x88 LDT descriptor */

	/* Segments used for calling PnP BIOS */
	.quad 0x00c09a0000000000	/* 0x90 32-bit code */
	.quad 0x00809a0000000000	/* 0x98 16-bit code */
	.quad 0x0080920000000000	/* 0xa0 16-bit data */
	.quad 0x0080920000000000	/* 0xa8 16-bit data */
	.quad 0x0080920000000000	/* 0xb0 16-bit data */
	/*
	 * The APM segments have byte granularity and their bases
	 * and limits are set at run time.
	 */
	.quad 0x00409a0000000000	/* 0xb8 APM CS    code */
	.quad 0x00009a0000000000	/* 0xc0 APM CS 16 code (16 bit) */
	.quad 0x0040920000000000	/* 0xc8 APM DS    data */

	.quad 0x0000000000000000	/* 0xd0 - unused */
	.quad 0x0000000000000000	/* 0xd8 - unused */
	.quad 0x0000000000000000	/* 0xe0 - unused */
	.quad 0x0000000000000000	/* 0xe8 - unused */
	.quad 0x0000000000000000	/* 0xf0 - unused */
	.quad 0x0000000000000000	/* 0xf8 - GDT entry 31: double-fault TSS */
```

重新加载段寄存器，刷新影子寄存器（详见Intel Manual）。这里搞不懂`ds`寄存器为什么用的是`__USER_DS`(#L1)。

这时候基本的内核内存映射都建立好了，跳转到`start_kernel@init/main.c`。

====================================================================================
TODO: 
1. smp启动分析(#L4)

====================================================================================
参考资料：`ULK（understanding the Linux kernel）`的`system startup`章节讲述了启动过程。
