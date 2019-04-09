# bootmem分配器
===============================================================================

在内核刚启动时，先使用bootmem分配器管理物理内存,只包括低地址内存,用于初始化过程中的内存管理(原因#L3)。它用一个bit表示一个物理page的状态，要么是保留的要么是可用的。在内核启动初期,可以通过e820获取物理内存映射表，该表包含许多不同类型的物理段，以此建立bootmem内存管理器。（在平坦内存模型中，从1M空间之后没有内存坑洞。）
在setup_arch的setup_memory中，初始化了bootmem分配器。`bootmap_size = init_bootmem(start_pfn, max_low_pfn);`初始化bootmem内存分配器，使用bitmap的方式表示每个物理page的状态，初始时全部设为reserved。bitmap存放在start_pfn开始的位置（mem_frame.md有图示），根据max_low_pfn的大小建立bitmap（这里认为内存从0开始，也暂不考虑内存hole），bitmap所占空间的大小就是bootmap_size。

bootmem_data结构用于管理bootmem，每个Node都有一个对应的指针，指向描述该节点在初始化时所用的bootmem分配器。
```
typedef struct bootmem_data {
	unsigned long node_boot_start;	// 该节点物理内存范围的起始物理地址
	unsigned long node_low_pfn;		// 该节点的最大low_pfn
	void *node_bootmem_map;		// bitmap的起始虚拟地址
	unsigned long last_offset;	// 上次分配内存空间在最后一个page内的偏移，当一个page没用完时，下次分配小块内存会继续使用。
	unsigned long last_pos;		// 上次分配内存占用的最后一个page的索引
	unsigned long last_success;	/* Previous allocation point.  To speed
					 * up searching */
	// last_success存放的是上次查找到可用page时用的起始索引。已被使用。
} bootmem_data_t;
```

对于单一node的平坦地址模型，有静态定义的全局变量：
```
#ifndef CONFIG_DISCONTIGMEM
static bootmem_data_t contig_bootmem_data;
struct pglist_data contig_page_data = { .bdata = &contig_bootmem_data };
```
唯一节点contig_page_data早就与contig_bootmem_data联系起来了。

初始化过程：
```
// start是指存放bitmap的起始位置，不是需要管理的内存区域的起始位置，需要管理的内存认为从0开始（单一node平坦地址模型）；pages是需要管理的内存页数。
unsigned long __init init_bootmem (unsigned long start, unsigned long pages)
{
	max_low_pfn = pages;
	min_low_pfn = start;			// 这个start是存放bitmap的初始位置，不是被管理内存的初始位置（0）。
	return(init_bootmem_core(NODE_DATA(0), start, 0, pages));	// NODE_DATA(0)在UMA下就是contig_page_data。
}

static unsigned long __init init_bootmem_core (pg_data_t *pgdat,
	unsigned long mapstart, unsigned long start, unsigned long end)
{
	bootmem_data_t *bdata = pgdat->bdata;		// bdata是存放在pgdat里的指针，其对应的数据结构和contig_page_data都是静态定义的变量,不需要动态分配空间.
	unsigned long mapsize = ((end - start)+7)/8;			// 计算bitmap需要的字节数

	pgdat->pgdat_next = pgdat_list;							// pgdat_list是全局变量,是所有node的链表头指针
	pgdat_list = pgdat;

	mapsize = (mapsize + (sizeof(long) - 1UL)) & ~(sizeof(long) - 1UL);		// 计算需要long的个数
	bdata->node_bootmem_map = phys_to_virt(mapstart << PAGE_SHIFT);			// 参数mapstart作为bitmap的其实地址，就在min_low_pfn，见mem_frame.md
	bdata->node_boot_start = (start << PAGE_SHIFT);
	bdata->node_low_pfn = end;

	/*
	 * Initially all pages are reserved - setup_arch() has to
	 * register free RAM areas explicitly.
	 */
	memset(bdata->node_bootmem_map, 0xff, mapsize);		// 把所有的内存初始化成保留的状态。setup_memory中会重新把max_low_pfn之下e820中的RAM类型的内存再设为可用，用于分配。

	return mapsize;
}
```

释放某块内存空间：
```
static void __init free_bootmem_core(bootmem_data_t *bdata, unsigned long addr, unsigned long size)
{
	unsigned long i;
	unsigned long start;
	/*
	 * round down end of usable mem, partially free pages are
	 * considered reserved.
	 */
	unsigned long sidx;						// sidx和eidx分别表示参数中的内存范围在bitmap中对应的下标范围。
	//node_boot_start表示该node的起始内存地址，在多Node环境中，bdata的bitmap只管理当前node内的内存，而该node的其实内存不一定是0。
	unsigned long eidx = (addr + size - bdata->node_boot_start)/PAGE_SIZE;	
	unsigned long end = (addr + size)/PAGE_SIZE;

	BUG_ON(!size);
	BUG_ON(end > bdata->node_low_pfn);

	if (addr < bdata->last_success)
		bdata->last_success = addr;

	/*
	 * Round up the beginning of the address.
	 */
	start = (addr + PAGE_SIZE-1) / PAGE_SIZE;
	sidx = start - (bdata->node_boot_start/PAGE_SIZE);

	for (i = sidx; i < eidx; i++) {
		if (unlikely(!test_and_clear_bit(i, bdata->node_bootmem_map)))
			BUG();
	}
}
```

保留某块地址空间：
`static void __init reserve_bootmem_core(bootmem_data_t *bdata, unsigned long addr, unsigned long size)`
reserve_bootmem_core的操作跟free_bootmem_core类似。略过。

分配size大小的连续内存：
```
// align是指分配内存起始地址的对齐大小，goal是指从哪个地址之后开始分配。
static void * __init
__alloc_bootmem_core(struct bootmem_data *bdata, unsigned long size,
		unsigned long align, unsigned long goal)
{
	unsigned long offset, remaining_size, areasize, preferred;
	unsigned long i, start = 0, incr, eidx;
	void *ret;

	if(!size) {
		printk("__alloc_bootmem_core(): zero-sized request\n");
		BUG();
	}
	BUG_ON(align & (align-1));

	eidx = bdata->node_low_pfn - (bdata->node_boot_start >> PAGE_SHIFT);
	offset = 0;
	if (align &&
	    (bdata->node_boot_start & (align - 1UL)) != 0)
		offset = (align - (bdata->node_boot_start & (align - 1UL)));
	offset >>= PAGE_SHIFT;		
	// offset是由于该node起始地址不是align对齐导致的分配时偏移地址。
	// 比如node_boot_start是3KB，而align=4KB，那么offset就是1KB；如果node_boot_start是5KB，align是4KB，offset就是3KB。

	/*
	 * We try to allocate bootmem pages above 'goal'
	 * first, then we try to allocate lower pages.
	 */
	if (goal && (goal >= bdata->node_boot_start) && 
	    ((goal >> PAGE_SHIFT) < bdata->node_low_pfn)) {
		preferred = goal - bdata->node_boot_start;

		if (bdata->last_success >= preferred)
			preferred = bdata->last_success;
	} else
		preferred = 0;

	preferred = ((preferred + align - 1) & ~(align - 1)) >> PAGE_SHIFT;
	preferred += offset;
	// preferred是搜索可用内存的起始索引。
	areasize = (size+PAGE_SIZE-1)/PAGE_SIZE;
	incr = align >> PAGE_SHIFT ? : 1;

restart_scan:
	// 找一块连续的大块内存
	for (i = preferred; i < eidx; i += incr) {
		unsigned long j;
		i = find_next_zero_bit(bdata->node_bootmem_map, eidx, i);
		i = ALIGN(i, incr);
		if (test_bit(i, bdata->node_bootmem_map))
			continue;
		for (j = i + 1; j < i + areasize; ++j) {
			if (j >= eidx)
				goto fail_block;
			if (test_bit (j, bdata->node_bootmem_map))
				goto fail_block;
		}
		start = i;
		goto found;
	fail_block:
		i = ALIGN(j, incr);
	}

	if (preferred > offset) {
		preferred = offset;
		goto restart_scan;
	}
	return NULL;

found:
	// last_success存放的是上次查找到可用page时用的起始索引。已被使用。
	bdata->last_success = start << PAGE_SHIFT;
	BUG_ON(start >= eidx);

	/*
	 * Is the next page of the previous allocation-end the start
	 * of this allocation's buffer? If yes then we can 'merge'
	 * the previous partial page with this allocation.
	 */
	if (align < PAGE_SIZE &&
	    bdata->last_offset && bdata->last_pos+1 == start) {
		// 接着上次未分配完的Page继续使用
		offset = (bdata->last_offset+align-1) & ~(align-1);
		BUG_ON(offset > PAGE_SIZE);
		remaining_size = PAGE_SIZE-offset;
		if (size < remaining_size) {
			areasize = 0;
			/* last_pos unchanged */
			bdata->last_offset = offset+size;
			ret = phys_to_virt(bdata->last_pos*PAGE_SIZE + offset +
						bdata->node_boot_start);
		} else {
			remaining_size = size - remaining_size;
			areasize = (remaining_size+PAGE_SIZE-1)/PAGE_SIZE;
			ret = phys_to_virt(bdata->last_pos*PAGE_SIZE + offset +
						bdata->node_boot_start);
			bdata->last_pos = start+areasize-1;
			bdata->last_offset = remaining_size;
		}
		bdata->last_offset &= ~PAGE_MASK;
	} else {
		bdata->last_pos = start + areasize - 1;
		bdata->last_offset = size & ~PAGE_MASK;
		ret = phys_to_virt(start * PAGE_SIZE + bdata->node_boot_start);
	}

	/*
	 * Reserve the area now:
	 */
	for (i = start; i < start+areasize; i++)
		if (unlikely(test_and_set_bit(i, bdata->node_bootmem_map)))
			BUG();
	memset(ret, 0, size);
	return ret;
}
```

`void * __init __alloc_bootmem (unsigned long size, unsigned long align, unsigned long goal)`
该函数在所有的node中尝试分配内存。通过便利padat_list实现。

`void * __init __alloc_bootmem_node (pg_data_t *pgdat, unsigned long size, unsigned long align, unsigned long goal)`
先尝试在指定pgdat中分配，若失败则在所有node中分配。

