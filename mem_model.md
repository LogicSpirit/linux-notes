linux内存模型有三种：
CONFIG_FLATMEM，CONFIG_DISCONTIGMEM，CONFIG_SPARSEMEM。
具体定义在include/asm-generic/memory-model.h中；
memory model针对的是物理内存的分布，主要涉及PFN跟page结构的转换；


PFN是page frame number的缩写，所谓page frame，就是针对物理内存而言的，把物理内存分成一个个的page size的区域，并且给每一个page编号，这个号码就是PFN。假设物理内存从0地址开始，那么PFN等于0的那个页帧就是0地址（物理地址）开始的那个page。假设物理内存从x地址开始，那么第一个页帧号码就是(x>>PAGE_SHIFT);
页帧可以理解为以4K为一个item，对整个物理内存空间进行编号；


1.flat memory model:访问物理内存的时候，物理地址空间是一个连续的，没有空洞的地址空间;
物理地址跟线性地址空间就有简单的线性映射关系，struct page数组mem_map中的每一项都可以对应某个具体
的物理页面；


#define __pfn_to_page(pfn) (mem_map + ((pfn) - ARCH_PFN_OFFSET))
#define __page_to_pfn(page) ((unsigned long)((page) - mem_map) + \
ARCH_PFN_OFFSET)


ARCH_PFN_OFFSET 跟架构相关的物理起始地址的PFN，也就是physical_start_address>>PAGE_SHIFT


2.Discontiguous Memory Model: cpu在访问物理内存的时候，其地址空间有一些空洞，是不连续的;
Discontiguous Memory Model可以看做是flat memory model的扩展，单个连续的物理地址空间作为一个
node，按flat memory model管理；


先由pfn找到所在的node，pfn在node内的偏移加上struct page数组起始地址
#define __pfn_to_page(pfn) \
({ unsigned long __pfn = (pfn);\
unsigned long __nid = arch_pfn_to_nid(__pfn);  \
NODE_DATA(__nid)->node_mem_map + arch_local_page_offset(__pfn, __nid);\
})




#define __page_to_pfn(pg) \
({ const struct page *__pg = (pg);\
struct pglist_data *__pgdat = NODE_DATA(page_to_nid(__pg));\
(unsigned long)(__pg - __pgdat->node_mem_map) +\
__pgdat->node_start_pfn;\
})


3.Sparse Memory Model：
为了支持内存hotplug，引入了sparse memory model，热插拔导致了一个node上的内存可能都变得更加"稀疏";
在sparsememory内存模型下，连续的地址空间按照SECTION（例如1G）被分成了一段一段的，其中每一section都是hotplug的，
因此sparse memory下，内存地址空间可以被切分的更细，支持更离散的Discontiguous memory;
如图：




整个连续的物理地址空间是按照一个section一个section来切断的，每一个section内部，其memory是连续的（即符合flat memory的特点），因此，mem_map的page数组依附于section结构（struct mem_section）而不是node结构了（struct pglist_data）。




先从pfn找到对应的section，再从section中找到对应的page
#define __page_to_pfn(pg) \
({ const struct page *__pg = (pg);\
int __sec = page_to_section(__pg);\
(unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec)));\
})


#define __pfn_to_page(pfn) \
({ unsigned long __pfn = (pfn);\
struct mem_section *__sec = __pfn_to_section(__pfn);\
__section_mem_map_addr(__sec) + __pfn;\
})


如果定义了CONFIG_SPARSEMEM_VMEMMAP的话，
struct page 统一存储在vmemmap开始的连续空间，
但是在这段空间的struct page还没有分配具体的物理地址空间，
需要建立页表，关联page跟物理地址；
#define __pfn_to_page(pfn) (vmemmap + (pfn))
#define __page_to_pfn(page) (unsigned long)((page) - vmemmap)
