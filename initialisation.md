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
reserve_bootmem用来把物理内存设置为保留的（使用过的）。把下面几块区域设为保留：
- 内核起始位置1M到内核结束、boot页表结束、bootmem bitmap结束那么多物理页
- 从0开始的一个page
- EBDA页。
- 640K之前的一个页，原因看注释。
- SMP栈等
- Initrd，如果有的话。

bootmem分配器的内存分配释放机制另开一篇。(#L5)

smp_alloc_memory() (#L3)

paging_init():
```
// PAE部分已删除
void __init paging_init(void)
{
	pagetable_init();

	load_cr3(swapper_pg_dir);

	__flush_tlb_all();

	kmap_init();
	zone_sizes_init();
}
```
pagetable_init()建立内核映射页表，还是使用swapper_pg_dir作为pgd。（swapper_pg_dir是页目录数组起始地址）

```
//PAE部分已删除
static void __init pagetable_init (void)
{
	unsigned long vaddr;
	pgd_t *pgd_base = swapper_pg_dir;

	/* Enable PSE if available */
	if (cpu_has_pse) {
		set_in_cr4(X86_CR4_PSE);
	}

	/* Enable PGE if available */
	if (cpu_has_pge) {
		set_in_cr4(X86_CR4_PGE);
		__PAGE_KERNEL |= _PAGE_GLOBAL;
		__PAGE_KERNEL_EXEC |= _PAGE_GLOBAL;
	}

	kernel_physical_mapping_init(pgd_base);
	remap_numa_kva();					// 没有配置DISCONTIGMEM时为空

	/*
	 * Fixed mappings, only the page table structure has to be
	 * created - mappings will be set by set_fixmap():
	 */
	vaddr = __fix_to_virt(__end_of_fixed_addresses - 1) & PMD_MASK;
	page_table_range_init(vaddr, 0, pgd_base);

	permanent_kmaps_init(pgd_base);		// 没有配置HIGHMEM时为空

}
```
kernel_physical_mapping_init 给 max_low_pfn以内的内存建立线性映射页表，之前只是在boot阶段映射了内核程序数据所占的物理内存（小于8M，用了2张页表），现在需要用bootmem分配器分配空间作为页表，完成所有低地址空间的线性映射。

关于fixmap，可以再加一篇文档。(#L6)
这里只是给这些固定的虚拟地址分配了页表，但是页表里没有填具体的物理页帧的位置。当需要用这个地址访问物理内存时，使用set_fixmap建立映射，就可以用虚拟地址访存了。

zone_sizes_init():
```
void __init zone_sizes_init(void)
{
	unsigned long zones_size[MAX_NR_ZONES] = {0, 0, 0};
	unsigned int max_dma, high, low;
	
	max_dma = virt_to_phys((char *)MAX_DMA_ADDRESS) >> PAGE_SHIFT;
	low = max_low_pfn;
	high = highend_pfn;
	
	if (low < max_dma)
		zones_size[ZONE_DMA] = low;
	else {
		zones_size[ZONE_DMA] = max_dma;
		zones_size[ZONE_NORMAL] = low - max_dma;
#ifdef CONFIG_HIGHMEM
		zones_size[ZONE_HIGHMEM] = high - low;
#endif
	}
	free_area_init(zones_size);	
}
```
每个node包含有3个zone，分别是DMA、NORMAL和HIGHMEM。DMA传输所用的物理地址不超过16M（原因#L8)。NORMAL指的是16M之后到max_low的内存区域，HIGHMEM是高于max_low_pfn的内存。
上面函数中的zone_size数组描述了每个zone中内存的pfn大小。

```
void __init free_area_init(unsigned long *zones_size)
{
	free_area_init_node(0, &contig_page_data, zones_size,
			__pa(PAGE_OFFSET) >> PAGE_SHIFT, NULL);
}
```
对于平坦地址模型（认为物理地址连续，没有坑洞），只有一个node，其描述结构就是contig_page_data，nid为0，起始地址为0。

```
void __init free_area_init_node(int nid, struct pglist_data *pgdat,
		unsigned long *zones_size, unsigned long node_start_pfn,
		unsigned long *zholes_size)
{
	pgdat->node_id = nid;
	pgdat->node_start_pfn = node_start_pfn;

	// 根据zone的内存描述，给pgdat的node_spanned_pages和node_present_pages成员赋值
	calculate_zone_totalpages(pgdat, zones_size, zholes_size);

	// node_start_pfn是0时，触发mem_map全局变量的赋值
	if (!pfn_to_page(node_start_pfn))
		node_alloc_mem_map(pgdat);

	free_area_init_core(pgdat, zones_size, zholes_size);
}
```
先完成pgdat部分成员的赋值，该node的基本信息。
node_alloc_mem_map:
```
void __init node_alloc_mem_map(struct pglist_data *pgdat)
{
	unsigned long size;

	size = (pgdat->node_spanned_pages + 1) * sizeof(struct page);
	pgdat->node_mem_map = alloc_bootmem_node(pgdat, size);
#ifndef CONFIG_DISCONTIGMEM
	mem_map = contig_page_data.node_mem_map;
#endif
}
```
从bootmem中为struct page数组分配空间，管理该node下的物理page。
如果是平坦内存模型，那就这一个node，直接把page数组的起始地址给mem_map全局变量。


free_area_init_core@page_alloc.c:
```
/*
 * Set up the zone data structures:
 *   - mark all pages reserved
 *   - mark all memory queues empty
 *   - clear the memory bitmaps
 */
static void __init free_area_init_core(struct pglist_data *pgdat,
		unsigned long *zones_size, unsigned long *zholes_size)
{
	unsigned long i, j;
	const unsigned long zone_required_alignment = 1UL << (MAX_ORDER-1);
	int cpu, nid = pgdat->node_id;
	unsigned long zone_start_pfn = pgdat->node_start_pfn;

	pgdat->nr_zones = 0;
	init_waitqueue_head(&pgdat->kswapd_wait);
	
	for (j = 0; j < MAX_NR_ZONES; j++) {
		struct zone *zone = pgdat->node_zones + j;
		unsigned long size, realsize;
		unsigned long batch;

		// zone_table是一个管理所有node的所有zone的全局的zone指针数组。
		zone_table[NODEZONE(nid, j)] = zone;
		realsize = size = zones_size[j];
		if (zholes_size)
			realsize -= zholes_size[j];

		if (j == ZONE_DMA || j == ZONE_NORMAL)
			nr_kernel_pages += realsize;
		nr_all_pages += realsize;

		zone->spanned_pages = size;
		zone->present_pages = realsize;
		zone->name = zone_names[j];
		spin_lock_init(&zone->lock);
		spin_lock_init(&zone->lru_lock);
		zone->zone_pgdat = pgdat;
		zone->free_pages = 0;

		zone->temp_priority = zone->prev_priority = DEF_PRIORITY;

		/*
		 * The per-cpu-pages pools are set to around 1000th of the
		 * size of the zone.  But no more than 1/4 of a meg - there's
		 * no point in going beyond the size of L2 cache.
		 *
		 * OK, so we don't know how big the cache is.  So guess.
		 */
		batch = zone->present_pages / 1024;
		if (batch * PAGE_SIZE > 256 * 1024)
			batch = (256 * 1024) / PAGE_SIZE;
		batch /= 4;		/* We effectively *= 4 below */
		if (batch < 1)
			batch = 1;

		for (cpu = 0; cpu < NR_CPUS; cpu++) {
			struct per_cpu_pages *pcp;

			pcp = &zone->pageset[cpu].pcp[0];	/* hot */
			pcp->count = 0;
			pcp->low = 2 * batch;
			pcp->high = 6 * batch;
			pcp->batch = 1 * batch;
			INIT_LIST_HEAD(&pcp->list);

			pcp = &zone->pageset[cpu].pcp[1];	/* cold */
			pcp->count = 0;
			pcp->low = 0;
			pcp->high = 2 * batch;
			pcp->batch = 1 * batch;
			INIT_LIST_HEAD(&pcp->list);
		}
		printk(KERN_DEBUG "  %s zone: %lu pages, LIFO batch:%lu\n",
				zone_names[j], realsize, batch);
		INIT_LIST_HEAD(&zone->active_list);
		INIT_LIST_HEAD(&zone->inactive_list);
		zone->nr_scan_active = 0;
		zone->nr_scan_inactive = 0;
		zone->nr_active = 0;
		zone->nr_inactive = 0;
		if (!size)
			continue;

		/*
		 * The per-page waitqueue mechanism uses hashed waitqueues
		 * per zone.
		 */
		zone->wait_table_size = wait_table_size(size);
		zone->wait_table_bits =
			wait_table_bits(zone->wait_table_size);
		zone->wait_table = (wait_queue_head_t *)
			alloc_bootmem_node(pgdat, zone->wait_table_size
						* sizeof(wait_queue_head_t));

		for(i = 0; i < zone->wait_table_size; ++i)
			init_waitqueue_head(zone->wait_table + i);

		pgdat->nr_zones = j+1;

		zone->zone_mem_map = pfn_to_page(zone_start_pfn);
		zone->zone_start_pfn = zone_start_pfn;

		if ((zone_start_pfn) & (zone_required_alignment-1))
			printk("BUG: wrong zone alignment, it will crash\n");

		memmap_init(size, nid, j, zone_start_pfn);

		zone_start_pfn += size;

		zone_init_free_lists(pgdat, zone, zone->spanned_pages);
	}
}
```
主要是初始化pgdata和zone中的一些值，具体数据结构的意义见内存管理数据结构解析。
memmap_init用来初始化zone对应的page数据结构的值。
zone_init_free_lists初始化zone中的free_area结构数组，该结构用于buddy system，数组长度默认为11，每个free_area元素描述一个2的n次方大小为单位的块的内存池，所以该结构有个free_list成员。还有一个map成员，是bitmap映射表的起始地址，该bitmap展示该zone中的page按照该种大小划分块时的使用情况。这里只是从bootmem中分配了内存块，并赋值给free_area[x].map。

如果启动参数中有earlyprintk=xxx，setup_early_printk

dmi_scan_machine:读取SMBIOS（System Management BIOS）表，它包含有BIOS/UEFI收集的系统信息。(#L5)

acpi_boot_init:(#L5)
get_smp_config:(#L6)

register_memory(max_low_pfn):
```
static void __init register_memory(unsigned long max_low_pfn)
{
	unsigned long low_mem_size;
	int	      i;

	if (efi_enabled)
		efi_initialize_iomem_resources(&code_resource, &data_resource);
	else
		legacy_init_iomem_resources(&code_resource, &data_resource);

	/* EFI systems may still have VGA */
	request_resource(&iomem_resource, &video_ram_resource);

	/* request I/O space for devices used on all i[345]86 PCs */
	for (i = 0; i < STANDARD_IO_RESOURCES; i++)
		request_resource(&ioport_resource, &standard_io_resources[i]);

	/* Tell the PCI layer not to allocate too close to the RAM area.. */
	low_mem_size = ((max_low_pfn << PAGE_SHIFT) + 0xfffff) & ~0xfffff;
	if (low_mem_size > pci_mem_start)
		pci_mem_start = low_mem_size;
}
```
申请resources，把内核内存中代码数据挂在iomem_resource下，把io端口挂在ioport_resource下。
在setup.c中有：
```
code_resource.start = virt_to_phys(_text);
code_resource.end = virt_to_phys(_etext)-1;
data_resource.start = virt_to_phys(_etext);
data_resource.end = virt_to_phys(_edata)-1;
```


