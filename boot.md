# 内核引导过程
上电后`CS=0xf000,EIP=0xfff0`，此后执行BIOS代码，BISO从引导介质中加载bootloader，bootloader加载内核镜像`bzImage`。
由Makefile可知，`bzImage`由`bootsect`、`setup`和`vmlinux.bin`构成。`bootsect`和`setup`是运行在实模式的代码，`vmlinux.bin`包括压缩内核及其解压程序。
具体加载过程，见内核文档`Documentation/x86/boot.txt`。
`0203`版本的`boot protocol`中，`bootsect`和`setup`被`bootloader`加载到尽可能低的地址，但是gdb调试的时候不知道该位置，只能用内存搜索的方式，查找`HdrS`字符串的 位置，得知`bootsect`和`setup`被加载到`0x10000`的位置。见mem_frame.md。

## 实模式部分
bootsect和setup用于boot protocol的头部数据：
```
// bootsect.S前半部分的代码不会被BootLoader使用。仅用来提示不要直接启动内核，这种flopy启动方式已被废弃。
// 末尾一些有用的数据。
	.org 497	//用来指定数据的起始地址，0x1f1
setup_sects:	.byte SETUPSECTS	//setup的大小（扇区数），如果为0那么实际值为4。实模式代码包括bootsect和setup。代码中默认值是4，qemu改成了5.
root_flags:	.word ROOT_RDONLY		//非零时根目录只读。已废弃，可在命令行用ro或rw选项代替。
syssize:	.word SYSSIZE			//实模式代码长度（16B为单位），但是只有两个字节，最多只能表示1M；所以如果是bzImage，LOAD_HIGH的话，该域就无意义了。

swap_dev:	.word SWAP_DEV

ram_size:	.word RAMDISK	// obsolete

vid_mode:	.word SVGA_MODE //用户可用vga=<mode>向bootloader传参，BootLoader写这个域，告诉内核vga的模式。0xFFFF表示"normal"，0xFFFE表示"ext"，0xFFFD表示"ask"。默认是"normal"。 
root_dev:	.word ROOT_DEV	// 根设备设备号，已废弃，在命令行中使用"root="选项代替。
boot_flag:	.word 0xAA55	// bootsect结束标志
```
```
// setup.S
#include <linux/config.h>
#include <asm/segment.h>
#include <linux/version.h>
#include <linux/compile.h>
#include <asm/boot.h>
#include <asm/e820.h>
#include <asm/page.h>
	
/* Signature words to ensure LILO loaded us right */
#define SIG1	0xAA55
#define SIG2	0x5A5A

//这几个宏貌似没用，qemu把setup加载到0x1020段了。
INITSEG  = DEF_INITSEG		# 0x9000, we move boot here, out of the way
SYSSEG   = DEF_SYSSEG		# 0x1000, system loaded at 0x10000 (65536).
SETUPSEG = DEF_SETUPSEG		# 0x9020, this is the current segment

DELTA_INITSEG = SETUPSEG - INITSEG	# 0x0020

.code16
.globl begtext, begdata, begbss, endtext, enddata, endbss

.text
begtext:
.data
begdata:
.bss
begbss:
.text

start:
	jmp	trampoline

		.ascii	"HdrS"		# 头部签名
		.word	0x0203		# 引导协议版本

realmode_swtch:	.word	0, 0	# 跳转到保护模式之前的钩子函数。cs:ip全为0时用default_switch, SETUPSEG，它关闭了NMI
start_sys_seg:	.word	SYSSEG	# (0x1000) obsolete
		.word	kernel_version	# 指向表示内核版本的字符串

type_of_loader:	.byte	0		# 这里会被BootLoader修改，告诉内核BootLoader的类型。qemu的BootLoader会把这里写0xb0
	
# flags, unused bits must be zero (RFU) bit within loadflags
loadflags:			# 加载标志，每个位表示一个特性。bit0表示内核是否被加载到高地址（1MB）；bit7表示heap_end_ptr有意义，启动时可以使用堆栈。
LOADED_HIGH	= 1			# If set, the kernel is loaded high
CAN_USE_HEAP	= 0x80			# If set, the loader also has set heap_end_ptr to tell how much space behind setup.S can be used for heap purposes.
#ifndef __BIG_KERNEL__
		.byte	0
#else
		.byte	LOADED_HIGH		# 这里的默认值是0x01.qemu的BootLoader给我们分配了heap，将这里改成了0x81.
#endif

setup_move_size: .word  0x8000		# size to move, when setup is not loaded at 0x90000. We will move setup to 0x90000 then just before jumping
					# into the kernel. However, only the loader knows how much data behind us also needs to be loaded.

code32_start:		# 32位代码的起始地址，即setup结束时的跳转目标地址，也是vmlinux.bin的起始地址。内核将vmlinux.bin加载到内存后，需要在这里告诉setup如何跳转到vmlinux.bin
#ifndef __BIG_KERNEL__
		.long	0x1000		#   0x1000 = default for zImage
#else
		.long	0x100000	# 0x100000 = default for big kernel
#endif

ramdisk_image:	.long	0		
ramdisk_size:	.long	0		# 初始的ramdisk/ramfs的大小，BootLoader加载ramdisk后把其起始地址和大小写在这里。如果没有让BootLoader加载ramdisk，此处为0.

bootsect_kludge:
		.long	0		# obsolete

					# BootLoader覆写这里，设置基于当前段的偏移地址，表示setup之后最大可用于heap的地址，从这里往下到setup代码结束都可以用作heap。
heap_end_ptr:	.word	modelist+1024	# default: 1KB after setup.	qemu把setup加载到0x1020段，此处设为0xfe00，所以最大heap地址为0x20000.

pad1:		.word	0
cmd_line_ptr:	.long 0		# BootLoader设置这里告诉内核cmdline的地址，cmdline可以放在setup的heap之后到0xA0000之间的任意位置。qemu将此处设置为0x20000，刚好是heap结束位置。

ramdisk_max:	.long (-__PAGE_OFFSET-(512 << 20)-1) & 0x7fffffff	# ramdisk最大安全地址:512MB。

trampoline:	call	start_of_setup
		.space	1024	# 此指令用于分配一片连续的存储区域并初始化为0。
# End of setup header #####################################################
```

内存探测：
```
loader_ok:
# Get memory size (extended mem, kB)

	xorl	%eax, %eax
	movl	%eax, (0x1e0)
#ifndef STANDARD_MEMORY_BIOS_CALL
	movb	%al, (E820NR)
# Try three different memory detection schemes.  First, try
# e820h, which lets us assemble a memory map, then try e801h,
# which returns a 32-bit memory size, and finally 88h, which
# returns 0-64m

# method E820H:
# the memory map from hell.  e820h returns memory classified into
# a whole bunch of different types, and allows memory holes and
# everything.  We scan through this memory map and build a list
# of the first 32 memory areas, which we return at [E820MAP].
# This is documented at http://www.acpi.info/, in the ACPI 2.0 specification.

#define SMAP  0x534d4150

meme820:
	xorl	%ebx, %ebx			# continuation counter
	movw	$E820MAP, %di		# point into the whitelist so we can have the bios directly write into it.
								# E820MAP = 0x2d0，是存放e820_entry的起始偏移，段是bootsect被加载的段，调试的时候是0x1000

jmpe820:
	movl	$0x0000e820, %eax		# e820, upper word zeroed
	movl	$SMAP, %edx			# ascii 'SMAP'
	movl	$20, %ecx			# size of the e820rec
	pushw	%ds				# data record.
	popw	%es
	int	$0x15				# make the call
	jc	bail820				# fall to e801 if it fails

	cmpl	$SMAP, %eax			# check the return is `SMAP'
	jne	bail820				# fall to e801 if it fails

	# If this is usable memory, we save it by simply advancing %di by sizeof(e820rec).
good820:
	movb	(E820NR), %al			# up to 32 entries
	cmpb	$E820MAX, %al
	jnl	bail820

	incb	(E820NR)		# E820NR = 0x1e8，是存放e820表项个数的偏移地址。
	movw	%di, %ax
	addw	$20, %ax
	movw	%ax, %di
again820:
	cmpl	$0, %ebx			# check to see if
	jne	jmpe820				# %ebx is set to EOF
bail820:


# method E801H:
# memory size is in 1k chunksizes, to avoid confusing loadlin.
# we store the 0xe801 memory size in a completely different place,
# because it will most likely be longer than 16 bits.
# (use 1e0 because that's what Larry Augustine uses in his
# alternative new memory detection scheme, and it's sensible
# to write everything into the same place.)

meme801:
	stc					# fix to work around buggy
	xorw	%cx,%cx				# BIOSes which dont clear/set
	xorw	%dx,%dx				# carry on pass/error of
						# e801h memory size call
						# or merely pass cx,dx though
						# without changing them.
	movw	$0xe801, %ax
	int	$0x15
	jc	mem88

	cmpw	$0x0, %cx			# Kludge to handle BIOSes
	jne	e801usecxdx			# which report their extended
	cmpw	$0x0, %dx			# memory in AX/BX rather than
	jne	e801usecxdx			# CX/DX.  The spec I have read
	movw	%ax, %cx			# seems to indicate AX/BX 
	movw	%bx, %dx			# are more reasonable anyway...

e801usecxdx:
	andl	$0xffff, %edx			# clear sign extend
	shll	$6, %edx			# and go from 64k to 1k chunks
	movl	%edx, (0x1e0)			# store extended memory size
	andl	$0xffff, %ecx			# clear sign extend
 	addl	%ecx, (0x1e0)			# and add lower memory into
						# total size.

# Ye Olde Traditional Methode.  Returns the memory size (up to 16mb or
# 64mb, depending on the bios) in ax.
mem88:

#endif
	movb	$0x88, %ah
	int	$0x15
	movw	%ax, (2)
```
探测基本输入输出设备,外设等.略
开启A20,略
设置早期的idt和gdt，并关闭中断，之前已经调用default_switch关闭了NMI中断:
```
# set up gdt and idt
	lidt	idt_48				# load idt with 0,0
	xorl	%eax, %eax			# Compute gdt_base
	movw	%ds, %ax			# (Convert %ds:gdt to a linear ptr)
	shll	$4, %eax
	addl	$gdt, %eax
	movl	%eax, (gdt_48+2)
	lgdt	gdt_48				# load gdt with whatever is
						# appropriate

# make sure any possible coprocessor is properly reset..
	xorw	%ax, %ax
	outb	%al, $0xf0
	call	delay

	outb	%al, $0xf1
	call	delay

# well, that went ok, I hope. Now we mask all interrupts - the rest
# is done in init_IRQ().
	movb	$0xFF, %al			# mask all interrupts for now
	outb	%al, $0xA1
	call	delay
	
	movb	$0xFB, %al			# mask all irq's but irq2 which
	outb	%al, $0x21			# is cascaded

# Well, that certainly wasn't fun :-(. Hopefully it works, and we don't
# need no steenking BIOS anyway (except for the initial loading :-).
# The BIOS-routine wants lots of unnecessary data, and it's less
# "interesting" anyway. This is how REAL programmers do it.
#
# Well, now's the time to actually move into protected mode. To make
# things as simple as possible, we do no register set-up or anything,
# we let the gnu-compiled 32-bit programs do that. We just jump to
# absolute address 0x1000 (or the loader supplied one),
# in 32-bit protected mode.
#
# Note that the short jump isn't strictly needed, although there are
# reasons why it might be a good idea. It won't hurt in any case.
	movw	$1, %ax				# protected mode (PE) bit
	lmsw	%ax				# This is it! 修改cr0使能PE，此后必须跟一条jmp指令
	jmp	flush_instr

flush_instr:
	xorw	%bx, %bx			# Flag to indicate a boot
	xorl	%esi, %esi			# Pointer to real-mode code
	movw	%cs, %si
	subw	$DELTA_INITSEG, %si
	shll	$4, %esi			# Convert to 32-bit pointer. **esi寄存器一直保存着bootsetc的起始地址，vmlinux中的代码也是借助这个寄存器获取boot参数的。**

# jump to startup_32 in arch/i386/boot/compressed/head.S
#	
# NOTE: For high loaded big kernels we need a
#	jmpi    0x100000,__BOOT_CS
#
#	but we yet haven't reloaded the CS register, so the default size 
#	of the target offset still is 16 bit.
#       However, using an operand prefix (0x66), the CPU will properly
#	take our 48 bit far pointer. (INTeL 80386 Programmer's Reference
#	Manual, Mixing 16-bit and 32-bit code, page 16-6)

	.byte 0x66, 0xea			# prefix + jmpi-opcode 从这里跳转到1MB出的vmlinux.bin，解压内核。
code32:	.long	0x1000				# will be set to 0x100000
						# for big kernels
	.word	__BOOT_CS
```

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


物理内存探测：
```
(gdb) x /8x 0x101e8
0x101e8:	0x00000006	0x00000000	0x00010500	0x0000dda9
0x101f8:	0xffff0000	0xaa551601	0x64482e00	0x02035372
(gdb) x /108x 0x102d0
0x102d0:	0x00000000	0x00000000	0x0009fc00	0x00000000
0x102e0:	0x00000001	0x0009fc00	0x00000000	0x00000400
0x102f0:	0x00000000	0x00000002	0x000f0000	0x00000000
0x10300:	0x00010000	0x00000000	0x00000002	0x00100000
0x10310:	0x00000000	0x07ee0000	0x00000000	0x00000001
0x10320:	0x07fe0000	0x00000000	0x00020000	0x00000000
0x10330:	0x00000002	0xfffc0000	0x00000000	0x00040000
0x10340:	0x00000000	0x00000002	0x00000000	0x00000000
0x10350:	0x00000000	0x00000000	0x00000000	0x00000000
```
e820: int 15h,AX=e820h，3.13内核此处用C语言写的，gcc支持16位汇编。
数据结构如下：
```
#define E820MAP			0x2d0	/* our map */
#define E820MAX			32		/* number of entries in E820MAP */
#define E820NR			0x1e8	/* # entries in E820MAP */
/* 这两个是偏移地址，此时段为0x1000（qemu例子）  */

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

## vmlinux.bin
`vmlinux.bin`被`BootLoader`加载到`0x100000`的物理地址上，该地址写在`setup`的`header`中，`setup`最后会跳转到这里。
`vmlinux.bin`由`arch/i386/boot/compressed/vmlinux`经过`objcopy`得到，而`compressed/vmlinux`由`compressed`下的`head.S、misc.c、piggy.S`构成。`piggy.S`其实包含着顶层`vmlinux`压缩后的二进制数据。
`startup_32@compressed/head.S`是程序入口，这里的函数名跟`vmlinux`中的重复并没有关系。主要完成内核代码的解压缩，并跳转到`vmlinux`。`vmlinux`的链接脚本是`arch/i386/kernel/vmlinux.lds`，其中定义的内核的段结构和程序入口。
startup_32@arch/i386/boot/compressed/head.S是被bootloader加载到1M处的vmlinux.bin的起始点，该处汇编代码会调用misc.c中的decompress_kernel解压内核，并把内核移动到同样是1MB物理地址处。并跳转到vmlinux的起始地址。

内核vmlinux解压到`0x100000`物理地址上，开始执行`startup_32@arch/i386/kernel/head.S`，这也是vmlinux程序的入口。
刚开始这段代码链接到`3GB+1MB`之上，但是加载到`1MB`开始的物理地址，所以访存用的地址都需要`-PAGE_OFFSET`。
```
# early boot GDT descriptor (must use 1:1 address mapping)
	.word 0x1e8		# 32 bit align gdt_desc.address
boot_gdt_descr:
	.word __BOOT_DS+7
	.long boot_gdt_table - __PAGE_OFFSET

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ENTRY(boot_gdt_table)
	.fill GDT_ENTRY_BOOT_CS,8,0
	.quad 0x00cf9a000000ffff/* kernel 4GB code at 0x00000000 */
	.quad 0x00cf92000000ffff/* kernel 4GB data at 0x00000000 */

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ENTRY(startup_32)
	cld
	lgdt boot_gdt_descr - __PAGE_OFFSET
	movl $(__BOOT_DS),%eax
	movl %eax,%ds
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs

	// 清空BSS，前面已经cld设置过DF过了。
	xorl %eax,%eax
	movl $__bss_start - __PAGE_OFFSET,%edi
	movl $__bss_stop - __PAGE_OFFSET,%ecx
	subl %edi,%ecx
	shrl $2,%ecx
	rep ; stosl

//这段代码设置最初启动时候的页表。
// 初始化页表。写一个页目录和几个页表，页表放在_end之后。变量init_pg_tables_end用来记录页表之后可作它用的位置。页表同时将虚拟地址0和PAGE_OFFSET之后的几个页映射到物理地址0.
// esi寄存器还保存着bootsect的起始地址。
page_pde_offset = (__PAGE_OFFSET >> 20);

	movl $(pg0 - __PAGE_OFFSET), %edi	# `pg0`是链接脚本中定义的,该位置是`_end`之后的4k对其的地址，其后 用于存放启动时候使用的页表，页表的大小跟启动阶段要管理的内存大小相关（从0到页表结束）
	movl $(swapper_pg_dir - __PAGE_OFFSET), %edx	# `swapper_pg_dir`是在`.bss.page_aligned`段分配的静态空间，用来存放启动阶段的页目录。
	movl $0x007, %eax			/* 0x007 = PRESENT+RW+USER 页面标志位*/
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
	movl %edi,(init_pg_tables_end - __PAGE_OFFSET)	#最终页表结束位置的物理地址保存在`init_pg_tables_end`变量 中，该变量在某个C文件中定义。在2.6.10内核中该地址不超过`8M`，即启动时才用了两张页表。

#ifdef CONFIG_SMP
	xorl %ebx,%ebx				/* This is the boot CPU (BSP) ，BSP的ebx是0，AP是1，后续的代码根据这个区分当前CPU是BSP还是AP */
	jmp 3f

/*
 * Non-boot CPU entry point; entered from trampoline.S
 * We can't lgdt here, because lgdt itself uses a data segment, but
 * we know the trampoline has already loaded the boot_gdt_table GDT
 * for us.
 */
ENTRY(startup_32_smp)
	cld
	movl $(__BOOT_DS),%eax
	movl %eax,%ds
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs

	xorl %ebx,%ebx
	incl %ebx				/* This is a secondary processor (AP) */

/*
 *	New page tables may be in 4Mbyte page mode and may
 *	be using the global pages. 
 *
 *	NOTE! If we are on a 486 we may have no cr4 at all!
 *	So we do not try to touch it unless we really have
 *	some bits in it to set.  This won't work if the BSP
 *	implements cr4 but this AP does not -- very unlikely
 *	but be warned!  The same applies to the pse feature
 *	if not equally supported. --macro
 *
 *	NOTE! We have to correct for the fact that we're
 *	not yet offset PAGE_OFFSET..
 */
#define cr4_bits mmu_cr4_features-__PAGE_OFFSET
	movl cr4_bits,%edx
	andl %edx,%edx
	jz 3f
	movl %cr4,%eax		# Turn on paging options (PSE,PAE,..)
	orl %edx,%eax
	movl %eax,%cr4

	btl $5, %eax		# check if PAE is enabled
	jnc 6f

	/* Check if extended functions are implemented */
	movl $0x80000000, %eax
	cpuid
	cmpl $0x80000000, %eax
	jbe 6f
	mov $0x80000001, %eax
	cpuid
	/* Execute Disable bit supported? */
	btl $20, %edx
	jnc 6f

	/* Setup EFER (Extended Feature Enable Register) */
	movl $0xc0000080, %ecx
	rdmsr

	btsl $11, %eax
	/* Make changes effective */
	wrmsr

6:
	/* cpuid clobbered ebx, set it up again: */
	xorl %ebx,%ebx
	incl %ebx
3:
#endif /* CONFIG_SMP */

	//开启分页
	movl $swapper_pg_dir-__PAGE_OFFSET,%eax
	movl %eax,%cr3		/* set the page table pointer.. */
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* ..and set paging (PG) bit */
	ljmp $__BOOT_CS,$1f	/* Clear prefetch and normalize %eip */
1:
	/* Set up the stack pointer */
	lss stack_start,%esp

	//清零eflags
	pushl $0
	popfl

#ifdef CONFIG_SMP
	andl %ebx,%ebx
	jz  1f				/* Initial CPU cleans BSS */
	jmp checkCPUtype
1:
#endif /* CONFIG_SMP */

	//设置dummy idt，此时中断处于关闭状态，也不会触发。
	call setup_idt

	//把启动参数拷贝到合适的地方。esi仍然保存着bootsect的起始地址；boot_params是在setup.c中定义的全局字符串数组，长度是PARAM_SIZE，为2K；
	movl $boot_params,%edi
	movl $(PARAM_SIZE/4),%ecx
	cld
	rep
	movsl
	movl boot_params+NEW_CL_POINTER,%esi	//NEW_CL_POINTER是0x228，在boot protocol中是cmdline_ptr的位置
	andl %esi,%esi
	jnz 2f			# New command line protocol
	cmpw $(OLD_CL_MAGIC),OLD_CL_MAGIC_ADDR
	jne 1f
	movzwl OLD_CL_OFFSET,%esi
	addl $(OLD_CL_BASE_ADDR),%esi
2:	// 走这里
	movl $saved_command_line,%edi		//把传给BootLoader的启动参数保存到saved_command_line中。saved_command_line是在main.c中定义的全局字符数组，长度为COMMAND_LINE_SIZE。
	movl $(COMMAND_LINE_SIZE/4),%ecx	//COMMAND_LINE_SIZE定义为256
	rep
	movsl
1:
checkCPUtype:	//检测CPU类型，i386，i486，之后的cpu支持ID位，支持cpuid指令获取cpu信息，并将cpu信息放到new_cpu_data中。

// 引用new_cpu_data成员的宏，在head.S开头定义。CPUINFO_x86这种都是在asm_offsets.h中定义的常数。
#define X86		new_cpu_data+CPUINFO_x86
#define X86_VENDOR	new_cpu_data+CPUINFO_x86_vendor
#define X86_MODEL	new_cpu_data+CPUINFO_x86_model
#define X86_MASK	new_cpu_data+CPUINFO_x86_mask
#define X86_HARD_MATH	new_cpu_data+CPUINFO_hard_math
#define X86_CPUID	new_cpu_data+CPUINFO_cpuid_level
#define X86_CAPABILITY	new_cpu_data+CPUINFO_x86_capability
#define X86_VENDOR_ID	new_cpu_data+CPUINFO_x86_vendor_id

	movl $-1,X86_CPUID		#  -1 for no CPUID initially

/* check if it is 486 or 386. */
/*
 * XXX - this does a lot of unnecessary setup.  Alignment checks don't
 * apply at our cpl of 0 and the stack ought to be aligned already, and
 * we don't need to preserve eflags.
 */

	movb $3,X86		# at least 386
	pushfl			# push EFLAGS
	popl %eax		# get EFLAGS
	movl %eax,%ecx		# save original EFLAGS
	xorl $0x240000,%eax	# flip AC and ID bits in EFLAGS		//测试EFLAGS中的AC,ID是否能被翻转，486之前两个都是保留位不支持翻转，486引入AC,pentium引入ID
	pushl %eax		# copy to EFLAGS
	popfl			# set EFLAGS
	pushfl			# get new EFLAGS
	popl %eax		# put it in eax
	xorl %ecx,%eax		# change in flags
	pushl %ecx		# restore original EFLAGS
	popfl
	testl $0x40000,%eax	# check if AC bit changed
	je is386	//如果AC不能翻转，那么只是i386

	movb $4,X86		# at least 486
	testl $0x200000,%eax	# check if ID bit changed
	je is486	//如果ID没有翻转，那么只是486，不支持pentium。

	//到这里说明ID位可用，新cpu支持cpuid指令。
	/* get vendor info */
	xorl %eax,%eax			# call CPUID with 0 -> return vendor ID
	cpuid
	movl %eax,X86_CPUID		# save CPUID level
	movl %ebx,X86_VENDOR_ID		# lo 4 chars
	movl %edx,X86_VENDOR_ID+4	# next 4 chars
	movl %ecx,X86_VENDOR_ID+8	# last 4 chars

	orl %eax,%eax			# do we have processor info as well?
	je is486

	movl $1,%eax		# Use the CPUID instruction to get CPU type
	cpuid
	movb %al,%cl		# save reg for future use
	andb $0x0f,%ah		# mask processor family
	movb %ah,X86
	andb $0xf0,%al		# mask model
	shrb $4,%al
	movb %al,X86_MODEL
	andb $0x0f,%cl		# mask mask revision
	movb %cl,X86_MASK
	movl %edx,X86_CAPABILITY

is486:	movl $0x50022,%ecx	# set AM, WP, NE and MP
	jmp 2f

is386:	movl $2,%ecx		# set MP
2:	movl %cr0,%eax
	andl $0x80000011,%eax	# Save PG,PE,ET
	orl %ecx,%eax
	movl %eax,%cr0

	call check_x87			# TODO (#L5)
	incb ready
	lgdt cpu_gdt_descr		# 加载新的gdt 和 idt,(idt没变化)。
	lidt idt_descr
	ljmp $(__KERNEL_CS),$1f		# 触发更新CS，刷新影子寄存器。同理，下面几条指令刷新了其他几个段寄存器。
1:	movl $(__KERNEL_DS),%eax	# reload all the segment registers
	movl %eax,%ss			# after changing gdt. 堆栈段用特权级0

	movl $(__USER_DS),%eax		# DS/ES contains default USER segment
	movl %eax,%ds			# 为什么ds/es在用户特权级？？？？？TODO(#L1)
	movl %eax,%es

	xorl %eax,%eax			# Clear FS/GS and LDT
	movl %eax,%fs
	movl %eax,%gs
	lldt %ax
	cld			# gcc2 wants the direction flag cleared at all times
#ifdef CONFIG_SMP
	movb ready, %cl	
	cmpb $1,%cl
	je 1f			# the first CPU calls start_kernel
				# all other CPUs call initialize_secondary
	call initialize_secondary
	jmp L6
1:
#endif /* CONFIG_SMP */
	call start_kernel	# 跳转到start_kernel@init/main.c
L6:
	jmp L6			# main should never return here, but
				# just in case, we know what happens.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
有`32`个表项，进行平坦地址映射。
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


====================================================================================
TODO: 
1. smp启动分析(#L4)

====================================================================================
参考资料：`ULK（understanding the Linux kernel）`的`system startup`章节讲述了启动过程。
