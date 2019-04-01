## start_kernel执行过程

setup_arch@arch/i386/kernel/setup.c
`memcpy(&boot_cpu_data, &new_cpu_data, sizeof(new_cpu_data));`
在checkCPUtype@arch/i386kernel/head.S中，好像把部分CPU的信息写到了new_cpu_data中,这里只是给boot_cpu_data赋值。这两个变量是cpuinfo_x86类型的，描述cpu的family,vendor,cpuid_level,capabilities,cache等属性。
early_cpu_init();利用cpuid指令获取更多cpu的信息赋值给boot_cpu_data。

```
 	ROOT_DEV = old_decode_dev(ORIG_ROOT_DEV);
 	drive_info = DRIVE_INFO;
 	screen_info = SCREEN_INFO;
	edid_info = EDID_INFO;
	apm_info.bios = APM_BIOS_INFO;
	ist_info = IST_INFO;
	saved_videomode = VIDEO_MODE;
	if( SYS_DESC_TABLE.length != 0 ) {
		MCA_bus = SYS_DESC_TABLE.table[3] &0x2;
		machine_id = SYS_DESC_TABLE.table[0];
		machine_submodel_id = SYS_DESC_TABLE.table[1];
		BIOS_revision = SYS_DESC_TABLE.table[2];
	}
	aux_device_present = AUX_DEVICE_INFO;

#ifdef CONFIG_BLK_DEV_RAM
	rd_image_start = RAMDISK_FLAGS & RAMDISK_IMAGE_START_MASK;
	rd_prompt = ((RAMDISK_FLAGS & RAMDISK_PROMPT_FLAG) != 0);
	rd_doload = ((RAMDISK_FLAGS & RAMDISK_LOAD_FLAG) != 0);
#endif
```
右边哪些宏都是boot_params的特定成员，从中提取数据赋值给左边的全局变量。

```
if (efi_enabled)
	efi_init();
else {
	printk(KERN_INFO "BIOS-provided physical RAM map:\n");
	print_memory_map(machine_specific_memory_setup());
}
```
machine_specific_memory_setup整理e820map，并且保存到新的全局变量e820中。BIOS中断得到的e820map可能不同的类型有重叠的区域，重叠时取最大的类型号。类型如下。
```
#define E820_RAM		1
#define E820_RESERVED	2
#define E820_ACPI		3 /* usable as RAM once ACPI tables have been read */
#define E820_NVS		4

/*
	Visually we're performing the following (1,2,3,4 = memory types)...

	Sample memory map (w/overlaps):
	   ____22__________________
	   ______________________4_
	   ____1111________________
	   _44_____________________
	   11111111________________
	   ____________________33__
	   ___________44___________
	   __________33333_________
	   ______________22________
	   ___________________2222_
	   _________111111111______
	   _____________________11_
	   _________________4______

	Sanitized equivalent (no overlap):
	   1_______________________
	   _44_____________________
	   ___1____________________
	   ____22__________________
	   ______11________________
	   _________1______________
	   __________3_____________
	   ___________44___________
	   _____________33_________
	   _______________2________
	   ________________1_______
	   _________________4______
	   ___________________2____
	   ____________________33__
	   ______________________4_
*/
```

```
if (!MOUNT_ROOT_RDONLY)
	root_mountflags &= ~MS_RDONLY;
init_mm.start_code = (unsigned long) _text;
init_mm.end_code = (unsigned long) _etext;
init_mm.end_data = (unsigned long) _edata;
init_mm.brk = init_pg_tables_end + PAGE_OFFSET;

code_resource.start = virt_to_phys(_text);
code_resource.end = virt_to_phys(_etext)-1;
data_resource.start = virt_to_phys(_etext);
data_resource.end = virt_to_phys(_edata)-1;
```
init_mm是一个struct mm变量，#L4。

parse_cmdline_early处理saved_command_line.

setup_memory:

```
static unsigned long __init setup_memory(void)
{
	unsigned long bootmap_size, start_pfn, max_low_pfn;

	/*
	 * partially used pages are not usable - thus
	 * we are rounding upwards:
	 */
	start_pfn = PFN_UP(init_pg_tables_end);

	find_max_pfn();

	max_low_pfn = find_max_low_pfn();

	printk(KERN_NOTICE "%ldMB LOWMEM available.\n",
			pages_to_mb(max_low_pfn));
	/*
	 * Initialize the boot-time allocator (with low memory only):
	 */
	bootmap_size = init_bootmem(start_pfn, max_low_pfn);

	register_bootmem_low_pages(max_low_pfn);

	/*
	 * Reserve the bootmem bitmap itself as well. We do this in two
	 * steps (first step was init_bootmem()) because this catches
	 * the (very unlikely) case of us accidentally initializing the
	 * bootmem allocator with an invalid RAM area.
	 */
	reserve_bootmem(HIGH_MEMORY, (PFN_PHYS(start_pfn) +
			 bootmap_size + PAGE_SIZE-1) - (HIGH_MEMORY));

	/*
	 * reserve physical page 0 - it's a special BIOS page on many boxes,
	 * enabling clean reboots, SMP operation, laptop functions.
	 */
	reserve_bootmem(0, PAGE_SIZE);

	/* reserve EBDA region, it's a 4K region */
	reserve_ebda_region();

    /* could be an AMD 768MPX chipset. Reserve a page  before VGA to prevent
       PCI prefetch into it (errata #56). Usually the page is reserved anyways,
       unless you have no PS/2 mouse plugged in. */
	if (boot_cpu_data.x86_vendor == X86_VENDOR_AMD &&
	    boot_cpu_data.x86 == 6)
	     reserve_bootmem(0xa0000 - 4096, 4096);

#ifdef CONFIG_SMP
	/*
	 * But first pinch a few for the stack/trampoline stuff
	 * FIXME: Don't need the extra page at 4K, but need to fix
	 * trampoline before removing it. (see the GDT stuff)
	 */
	reserve_bootmem(PAGE_SIZE, PAGE_SIZE);
#endif
#ifdef CONFIG_ACPI_SLEEP
	/*
	 * Reserve low memory region for sleep support.
	 */
	acpi_reserve_bootmem();
#endif
#ifdef CONFIG_X86_FIND_SMP_CONFIG
	/*
	 * Find and reserve possible boot-time SMP configuration:
	 */
	find_smp_config();
#endif

	return max_low_pfn;
}
```

start_pfn是init_pg_tables_end之后的pfn，该变量在head.S中建立页映射的时候赋过值。

`bootmap_size = init_bootmem(start_pfn, max_low_pfn);`初始化bootmem内存分配器，使用bitmap的方式表示每个物理page的状态，初始时全部设为reserved。bitmap存放在start_pfn开始的位置，根据max_low_pfn的大小建立bitmap（这里认为内存从0开始，也暂不考虑内存hole），bitmap所占空间的大小就是bootmap_size。
`bootmap_sizeregister_bootmem_low_pages(max_low_pfn);` 根据e820中的内存表，把类型为RAM的物理页在bootmem中设置为可用。如果实际物理内存大于896M的话，就会受max_low_pfn的限制。
reserve_bootmem用来把物理内存设置为保留的（使用过的）。
把内核起始位置1M到内核结束、boot页表结束、bootmem bitmap结束那么多物理页设置为保留。
把从0开始的一个page设置为保留。
保留EBDA页。
把640K之前的一个页保留，原因看注释。
保留SMP栈等
如果有Initrd，该区域也要保留。

bootmem分配器的内存分配释放机制另开一篇。(#L5)

smp_alloc_memory() (#L3)

paging_init()
