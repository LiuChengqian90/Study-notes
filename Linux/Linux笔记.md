| 编号   | 书名              |
| ---- | --------------- |
| A    | Linux 内核设计与实现   |
| B    | 深入理解Linux内核     |
| C    | 深入理解Linux网络技术内幕 |
| D    | Linux设备驱动程序     |



32位，虚拟地址 寻址空间为4G，0x00000000~0xffffffff  （或者 32根地址总线，2^32），另外，若支持PAE可以进行扩展。

# 内存管理

## A：第12章 内存管理

在内核里分配内存可不像在其他地方分配内存那么容易。从根本上讲，是因为内核本身不能像用户空间那样奢侈地使用内存。内核不支持简单便捷的内存分配方式。比如，内核一般不能睡眠。再加上内存分配机制不能太复杂，所以内核中获取内存要比在用户空间复杂得多。

### 页

内核把物理页作为内存管理的基本单位。尽管处理器的最小可寻址单元通常为字（甚至字节），但是，内存管理单元（MMU，管理内存并把虚拟地址转换为物理地址的硬件）通常以页为单位进行处理。正因为如此，MMU以页（page）大小为单位来管理系统中的页表。从虚拟内存的角度来看，页就是最小单位。

体系结构不同，支持的页的大小也不同，还有些体系结构甚至支持几种不同的页大小。大多数32位体系结构支持4KB的页，而64位体系结构一般支持8KB的页。也就是说，支持4KB页大小、1GB物理内存的机器上，物理内存会被划分为262144个页。

内核用struct page结构表示系统中的每个物理页，该结构位于<linux/mm_types.h>中。

```c
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	atomic_t _count;		/* Usage count, see below. */
	union {
		atomic_t _mapcount;	/* Count of ptes mapped in mms,
					 * to show when page is mapped
					 * & limit reverse map searches.
					 */
		struct {		/* SLUB */
			u16 inuse;
			u16 objects;
		};
	};
	union {
	    struct {
		unsigned long private;		/* Mapping-private opaque data:
					 	 * usually used for buffer_heads
						 * if PagePrivate set; used for
						 * swp_entry_t if PageSwapCache;
						 * indicates order in the buddy
						 * system if PG_buddy is set.
						 */
		struct address_space *mapping;	/* If low bit clear, points to
						 * inode address_space, or NULL.
						 * If page mapped as anonymous
						 * memory, low bit is set, and
						 * it points to anon_vma object:
						 * see PAGE_MAPPING_ANON below.
						 */
	    };
#if USE_SPLIT_PTLOCKS
	    spinlock_t ptl;
#endif
	    struct kmem_cache *slab;	/* SLUB: Pointer to slab */
	    struct page *first_page;	/* Compound tail pages */
	};
	union {
		pgoff_t index;		/* Our offset within mapping. */
		void *freelist;		/* SLUB: freelist req. slab lock */
	};
	struct list_head lru;		/* Pageout list, eg. active_list
					 * protected by zone->lru_lock !
					 */
	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */
#ifdef CONFIG_WANT_PAGE_DEBUG_FLAGS
	unsigned long debug_flags;	/* Use atomic bitops on this */
#endif

#ifdef CONFIG_KMEMCHECK
	/*
	 * kmemcheck wants to track the status of each byte in a page; this
	 * is a pointer to such a status block. NULL if not tracked.
	 */
	void *shadow;
#endif
};
```

flag域用来存放页的状态。状态包括页是不是脏的，是不是该锁定在内存中等。flag每一位表示一种状态，因此它至少可同时表示32种不同的状态，这些标志定义在<linux/page-flags.h>中。

_count域存放页的引用计数。值变为-1时，说明内核没有引用这一页。内核代码调用page_count()函数进行检查，该函数唯一的参数是page结构，返回值0表示页空闲，正数表示在使用。一个页可以由页缓存使用（此时，mapping域指向和这个页关联的address_space对象），或者作为私有数据（由private指向），或者作为进程页表中的映射。

virtual域是也的虚拟地址。通常，它是页在虚拟内存中的地址。但是高端内存并不永久地映射到内核地址空间。此时，这个域的值为NULL，需要时，必须动态地映射这些页。

必须理解的一点：page结构与物理页相关，而并非与拟页相关。因此，该结构对页的描述只是短暂的。即使页种所包含的数据继续存在。由于交换等原因，它们[^应该指数据]也可能并不再和同一个page结构相关联。内核仅仅用这个数据结构来描述当前时刻在相关的物理页中存放的东西。这种数据结构的目的在于描述物理页本身，而非其中的数据。

内核用这一结构来管理系统中的所有页，以便了解一个页是否空闲（也就是页有没有被分配）。如果页被分配，内核还需要知道谁拥有这个页。拥有者可能是用户空间进程、动态分配的内核数据、静态内核代码或页高速缓存等。

### 区

由于硬件的限制，内核并不能对页一视同仁。有些页位于内存中特定的物理地址上，不能将其用于特定的任务。由于存在这种限制，所以内核把页划分为不同的区（zone）---对具有相似特征的页进行分组。Linux必须处理如下两种由于邮件存在缺陷而引起的内存寻址问题：

- 一些硬件只能用某些特定的内存地址来执行DMA（直接内存访问）
- 一些体系结构的内存物理寻址范围比虚拟寻址范围大的多。有一些内存不能永久地映射到内核空间。

<linux/mmzone.h>

| ZONE_DMA     | 执行DMA操作                                  |
| ------------ | ---------------------------------------- |
| ZONE_DMA32   | 和ZOME_DMA类似，但是这些页面只能被32位设备访问。在某些体系结构中，该区比ZONE_DMA更大。 |
| ZONE_NORMAL  | 能正常映射的页。                                 |
| ZONE_HIGHMEM | 包含“高端内存”，其中的页并不能永久地映射到内核地址空间。            |

区的使用和分布与体系结构相关。例如，某些体系结构在内存的任何地址上执行DMA都没有问题。在这些体系结构中，ZONE_DMA为空，ZONE_NORMAL就可以直接用于分配。与此相反，x86体系结构中，ISA设备就不在整个32位的地址空间中执行DMA，因为ISA设备只能访问物理内存前16MB。因此，ZONE_DMA在x86上包含的页都在0-16MB的内存范围。

ZONE_HIGHMEM的工作方式也差不多。能否直接映射取决于体系结构。在32位x86系统上，ZONE_HIGHMEM为高于896MB的所有物理内存。在其他体系结构撒花姑娘，由于所有内存都被直接映射，所以ZONE_HIGHMEM为空。ZONE_HIGHMEM所在的内存就是所谓的高端内存（high memory）。系统的其余内存就是所谓的低端内存（low memory）。

| 区            | 描述      | 物理内存     |
| ------------ | ------- | -------- |
| ZONE_DMA     | DMA使用的页 | <16MB    |
| ZONE_NORMAL  | 正常可寻址的页 | 16~896MB |
| ZONE_HIGHMEM | 动态映射的页  | >896MB   |

Linux把系统的页划分为区，形成不同的内存池，这样就可以根据用途进行分配了。注意，区的划分没有任何物理意义，这只不过是内核为了管理页而采取的一种逻辑上的分组。

某些分配可能需要从特定的区中获取页，而另外一些分配则可以从多个区中获取页。比如，DMA内存必须从ZONE_DMA中进行分配，但是一般用途的内存既能从ZONE_DMA分配，也能从ZONE_NORMAL分配，不过不能同时从两个区分配，因为分配是不能跨区界限的。

不是所有的体系结构都定义了全部区，有些64位的体系结构，如Intel的x86-64体系结构可以映射和处理64位的内存空间，所以x86-64没有ZONE_HIGHMEM区，所有内存都处于ZONE_DMA和ZONE_NORMAL区。

每个区都用struct zone表示，在<linux/mmzone.h>定义：

```c
struct zone {
	/* Fields commonly accessed by the page allocator */
	/* zone watermarks, access with *_wmark_pages(zone) macros */
	unsigned long watermark[NR_WMARK];
	/*
	 * We don't know if the memory that we're going to allocate will be freeable
	 * or/and it will be released eventually, so to avoid totally wasting several
	 * GB of ram we must reserve some of the lower zone memory (otherwise we risk
	 * to run OOM on the lower zones despite there's tons of freeable ram
	 * on the higher zones). This array is recalculated at runtime if the
	 * sysctl_lowmem_reserve_ratio sysctl changes.
	 */
	unsigned long		lowmem_reserve[MAX_NR_ZONES];
#ifdef CONFIG_NUMA
	int node;
	/*
	 * zone reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif
	struct per_cpu_pageset __percpu *pageset;
	/*
	 * free areas of different sizes
	 */
	spinlock_t		lock;
	int                     all_unreclaimable; /* All pages pinned */
#ifdef CONFIG_MEMORY_HOTPLUG
	/* see spanned/present_pages for more description */
	seqlock_t		span_seqlock;
#endif
	struct free_area	free_area[MAX_ORDER];
#ifndef CONFIG_SPARSEMEM
	/*
	 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
	 * In SPARSEMEM, this map is stored in struct mem_section
	 */
	unsigned long		*pageblock_flags;
#endif /* CONFIG_SPARSEMEM */
#ifdef CONFIG_COMPACTION
	/*
	 * On compaction failure, 1<<compact_defer_shift compactions
	 * are skipped before trying again. The number attempted since
	 * last failure is tracked with compact_considered.
	 */
	unsigned int		compact_considered;
	unsigned int		compact_defer_shift;
#endif
	ZONE_PADDING(_pad1_)
	/* Fields commonly accessed by the page reclaim scanner */
	spinlock_t		lru_lock;	
	struct zone_lru {
		struct list_head list;
	} lru[NR_LRU_LISTS];

	struct zone_reclaim_stat reclaim_stat;
	unsigned long		pages_scanned;	   /* since last reclaim */
	unsigned long		flags;		   /* zone flags, see below */

	/* Zone statistics */
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
	/*
	 * prev_priority holds the scanning priority for this zone.  It is
	 * defined as the scanning priority at which we achieved our reclaim
	 * target at the previous try_to_free_pages() or balance_pgdat()
	 * invocation.
	 *
	 * We use prev_priority as a measure of how much stress page reclaim is
	 * under - it drives the swappiness decision: whether to unmap mapped
	 * pages.
	 *
	 * Access to both this field is quite racy even on uniprocessor.  But
	 * it is expected to average out OK.
	 */
	int prev_priority;
	/*
	 * The target ratio of ACTIVE_ANON to INACTIVE_ANON pages on
	 * this zone's LRU.  Maintained by the pageout code.
	 */
	unsigned int inactive_ratio;
	ZONE_PADDING(_pad2_)
	/* Rarely used or read-mostly fields */
	/*
	 * wait_table		-- the array holding the hash table
	 * wait_table_hash_nr_entries	-- the size of the hash table array
	 * wait_table_bits	-- wait_table_size == (1 << wait_table_bits)
	 *
	 * The purpose of all these is to keep track of the people
	 * waiting for a page to become available and make them
	 * runnable again when possible. The trouble is that this
	 * consumes a lot of space, especially when so few things
	 * wait on pages at a given time. So instead of using
	 * per-page waitqueues, we use a waitqueue hash table.
	 *
	 * The bucket discipline is to sleep on the same queue when
	 * colliding and wake all in that wait queue when removing.
	 * When something wakes, it must check to be sure its page is
	 * truly available, a la thundering herd. The cost of a
	 * collision is great, but given the expected load of the
	 * table, they should be so rare as to be outweighed by the
	 * benefits from the saved space.
	 *
	 * __wait_on_page_locked() and unlock_page() in mm/filemap.c, are the
	 * primary users of these fields, and in mm/page_alloc.c
	 * free_area_init_core() performs the initialization of them.
	 */
	wait_queue_head_t	* wait_table;
	unsigned long		wait_table_hash_nr_entries;
	unsigned long		wait_table_bits;
	/*
	 * Discontig memory support fields.
	 */
	struct pglist_data	*zone_pgdat;
	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn;
	/*
	 * zone_start_pfn, spanned_pages and present_pages are all
	 * protected by span_seqlock.  It is a seqlock because it has
	 * to be read outside of zone->lock, and it is done in the main
	 * allocator path.  But, it is written quite infrequently.
	 *
	 * The lock is declared along with zone->lock because it is
	 * frequently read in proximity to zone->lock.  It's good to
	 * give them a chance of being in the same cacheline.
	 */
	unsigned long		spanned_pages;	/* total size, including holes */
	unsigned long		present_pages;	/* amount of memory (excluding holes) */
	/*
	 * rarely used fields:
	 */
	const char		*name;
} ____cacheline_internodealigned_in_smp;
```

这个结构体很大，但是，系统中只有三个区，因此，也只有三个这样的结构。

lock域是一个自旋锁，防止该结构被并发访问。注意，这个域只保护结构，而不保护驻留在这个区的所有页。没有特定的锁来保护单个页，但是，部分内核可以锁住在页中驻留的数据。

watermark数组持有该区的最小值、最低和最高水位值。内核使用水位为每个内存区设置合适的内存消耗基准。该水位随空闲内存的多少而变化。

name是一个以NULL的字符串，表示这个区的名字。内核启动期间初始化这个值，其代码位于mm/page_alloc.c中。三个区名字分别为“DMA”、“Normal”、“HighMem”。

### 获得页

内核提供了一种请求内存的底层机制，并提供了对它进行访问的几个接口。所有这些接口都以页为单位分配内存，定义于<linux/gfp.h>中。最核心函数是：

```c
static inline struct page *
alloc_pages(gfp_t gfp_mask, unsigned int order)
```

该函数分配2^order^（1<<order）个连续的物理页，并返回一个指针，该指针指向第一个页的page结构体，出错返回NULL。可以用

```c
void * page_address(struct page *page)
```

将页转换成它的逻辑地址。如果无须用到struct page，可以调用

```c
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);
```

这个函数与alloc_pages()作用相同，不过它直接返回所请求的第一个页的逻辑地址。因为页是连续的，所以其他页也会紧随其后。

如果只需一页，就可以用下面两个封装好的宏：

```c
#define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0)
#define __get_free_page(gfp_mask) __get_free_pages((gfp_mask), 0)

```

```c
unsigned long get_zeroed_page(gfp_t gfp_mask);
```

这个函数与__get_free_pages()工作方式相同，只不过把分配好的页都填充成了0---字节中的每一位都要取消设置。如果分配的页是给用户空间的，这个函数就非常有用，会清除页中“随机地”包含的敏感数据。

可以用下面的函数来释放页：

```c
void __free_pages(struct page *page, unsigned int order)
void free_pages(unsigned long addr, unsigned int order);
#define free_page(addr) free_pages((addr), 0)
```

释放页时要谨慎，只能释放属于自己的页。

```c
unsigned long page;
page = __get_free_pages(GFP_KERNEL, 3);
if(!page){
    /*没有足够的内存*/
  return -ENOMEM;
}/*page指向8个物理页中的第1个页*/
free_pages(page, 3);
/*页现在已经释放，之后不应该再访问page中的地址*/

```

应该在程序开始之前就应该分配内存，会让错误处理得容易一些。

当需要以页为单位的一族连续物理页时，这些低级函数很有用。对于常用的以字节为单位的分配来说，内核提供的函数是kmalloc()。

### kmalloc()

kmalloc()与用户空间的malloc()一族函数非常类似，只不过它多了一个flags参数。

kmalloc()在<linux/slab_def.h>中声明：

```c
static __always_inline void *kmalloc(size_t size, gfp_t flags)
```

函数返回一个指向内存块的指针，其内存块至少要有size大小，所分配的内存区在物理上是连续的。

gfp_mask标志分为三类：行为修饰符、区修饰符及类型。行为修饰符表示内核应当如何分配所需的内存。某些特定情况下，只能使用某些特定的方法分配内存。例如，中断处理程序就要求内核在分配内存过程中不能睡眠。区修饰符表示从哪儿分配内存。类型标志组合了行为修饰符和区修饰符，将各种可能遇到的组合归纳为不同的类型。

- 行为修饰符。在<linux/gfp.h>中声明。

  | 标志            | 描述                            |
  | ------------- | ----------------------------- |
  | __GFP_WAIT    | 分配器可以睡眠                       |
  | __GFP_HIGH    | 分配器可以访问紧急事件缓冲池                |
  | __GFP_IO      | 分配器可以启动磁盘I/O                  |
  | __GFP_FS      | 分配器可以启动文件系统I/O                |
  | __GFP_COLD    | 分配器应该使用高速缓存中快要淘汰出去的页          |
  | __GFP_NOWARN  | 分配器将不打印失败警告                   |
  | __GFP_REPEAT  | 分配器在分配失败时重复进行分配，但这次分配还存在失败的可能 |
  | __GFP_NOFAIL  | 分配器将无限地重复分配，分配不能失败            |
  | __GFP_NORETRY | 分配器分配失败时不会重新分配                |
  | __GFP_COMP    | 添加混合页元数据，在hugetlb的代码          |

- 区修饰符。内核优先从ZONE_NORMAL开始，这样可确保其他区在需要时有足够的空闲页。

  | 标志            | 描述                          |
  | ------------- | --------------------------- |
  | __GFP_DMA     | 从ZONE_DMA分配                 |
  | __GFP_DMA32   | 从ZONE_DMA32分配               |
  | __GFP_HIGHMEM | 从ZONE_HIGHMEM或ZONE_NORMAL分配 |

  不能给_get_free_pages()或kmalloc()指定ZONE_HIGHMEM，因为这两个函数返回的都是逻辑地址，而不是page结构，这两个函数分配的内存当前有可能还没有映射到内核的虚拟地址空间，因此，根本没有逻辑地址。只有alloc_pages()才能分配高端内存。

- 类型标识。

  | 标志           | 修饰符标志                                    | 描述                                       |
  | ------------ | ---------------------------------------- | ---------------------------------------- |
  | GFP_NOWAIT   | (GFP_ATOMIC & ~__GFP_HIGH)               | 类似GFP_ATOMIC，但是，调用不会退给紧急内存池。增加了内存分配失败的可能性。 |
  | GFP_ATOMIC   | __GFP_HIGH                               | 用在中断处理程序、下半部、持有自旋锁以及其他不能睡眠的地方            |
  | GFP_NOIO     | __GFP_WAIT                               | 可阻塞，但不会启动磁盘I/O。这个标志在不能引发更多磁盘I/O时能阻塞I/O代码，可能导致令人不快的递归 |
  | GFP_NOFS     | (__GFP_WAIT \| __GFP_IO)                 | 必要时可能引起阻塞，也可能启动磁盘I/O，但是不会启动文件系统操作。       |
  | GFP_KERNEL   | (__GFP_WAIT \| __GFP_IO \| __GFP_FS)     | 常规分配方式，可能会阻塞。在睡眠安全时用在进程上下文代码中。           |
  | GFP_USER     | (__GFP_WAIT \| __GFP_IO \| __GFP_FS \| __GFP_HARDWALL) | 常规分配方式，可能会阻塞。用于为用户空间进程分配内存。              |
  | GFP_HIGHUSER | (__GFP_WAIT \| __GFP_IO \| __GFP_FS \| __GFP_HARDWALL \| __GFP_HIGHMEM) | 从ZONE_HIGHMEM进行分配，可能会阻塞。用于为用户空间进程分配内存。   |
  | GFP_DMA      | __GFP_DMA                                | 从ZONE_DMA进行分配。                           |

  kmalloc()分配的内存用kfree()释放。

### vmalloc()

类似kmalloc()，不过vmalloc()分配的内存虚拟地址是连续的，而物理地址则无须连续。这也是用户空间分配函数的工作方式。malloc()返回的页在进程的虚拟空间是连续的，但并不保证在物理RAM中也是连续的。kmalloc()函数确保页在物理地址上是连续的（虚拟地址自然也是连续的）。vmalloc()函数通过分配非连续的物理块，再“修正”页表，把内存映射到逻辑空间的连续区域。

出于性能考虑，kmalloc()比vmalloc()更合适。物理上不连续的页转换为虚拟地址空间上连续的页，必须专门建立页表项。糟糕的是，通过vmalloc()获得的页必须一个一个地进行映射，这就会导致比直接内存映射大的多的TLB抖动。vfree()释放分配的内存。

### slab层

分配和释放数据结构是内核中最普遍的操作之一。为了便于数据的频繁分配和回收，空闲链表是一个比较好的选择。当代码需要一个新的数据结构实例时，从空闲链表抓取，而不需要分配内存，当不需要这个数据结构实例时，就把它放回空闲链表，而不是释放它。从这个意义上说，空闲链表相当于对象高速缓存---快速存储频繁使用的对象类型。

内核中，空闲链表面临的主要问题之一是不能全局控制。当可用内存变得紧缺时，内核无法通知每个空闲链表让其收缩缓存大小。实际上，内核根本不知道存在任何空闲链表。为弥补这一缺陷，且使代码更加稳固，Linux内核提供了slab层（slab分配器）。slab分配器扮演了通用数据结构缓存层角色。

slab分配器试图在几个基本原则之间寻求一种平衡：

- 频繁使用的数据结构也会频繁分配和释放，应当缓存它们
- 频繁分配和回收必然会导致内存碎片。为避免，空闲链表的缓存会连续地存放。
- 回收的对象可以立即投入下一次分配。
- 如果分配器知道对象大小、页大小和总的高速缓存的大小这样的概念，它会做出更明智的决策。
- 如果让部分缓存专属于单个处理器，那么，分配和释放就可以在不加SMP锁的情况下进行。
- 分配器与NUMA相关，它可以从相同的内存节点为请求者进行分配。
- 对存放的对象进行着色（color），以防止多个对象映射到相同的高速缓存行（cache line）。

slab层把不同的对象划分为所谓高速缓存组，其中每个高速缓存组存放不同类型的对象。每种对象类型对应一个高速缓存。kmalloc()接口建立在slab层之上，使用了一组通用高速缓存。

然后，这些高速缓存又被划分为slab(这也是这个子系统名字的来由)。slab由一个或多个物理上连续的页组成。一般情况下，slab也就仅仅由一页组成。每个高速缓存可以由多个slab组成。
每个slab都包含一些对象成员，这里的对象指的是被缓存的数据结构。每个slab处于三种状态之一:满、部分满或空。当内核的某一部分需要一个新的对象时，先从部分满的slab中进行分配。如果没有部分满的slab,就从空的slab中进行分配。如果没有空的slab,就要创建一个slab。这种策略能减少碎片。

![高速缓存、slab及对象之间的关系.png](https://github.com/LiuChengqian90/Study-notes/blob/master/image/Linux/%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98%E3%80%81slab%E5%8F%8A%E5%AF%B9%E8%B1%A1%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.png?raw=true)

每个高速缓存都使用kmem_cache结构来表示。这个结构包含三个链表：slabs_full、slabs_partial和slabs_empty，均存放在kmem_list3结构内，该结构在mm/slab.c中定义。这些链表包含高速缓存中的所有slab，slab描述符struct slab用来描述每个slab：

```C
struct slab {
	struct list_head list;
	unsigned long colouroff;
	void *s_mem;		/* including colour offset */
	unsigned int inuse;	/* num of objs active in slab */
	kmem_bufctl_t free;
	unsigned short nodeid;
};
```

如果slab很小，或者slab内部有足够的空间容纳slab描述符，那么描述符就放在slab里（slab自身开始的地方），否则就在slab另行分配。

slab分配器可以创建新的slab：

```c
static void *kmem_getpages(struct kmem_cache *cachep, gfp_t flags, int nodeid)
{
	struct page *page;
	int nr_pages;
	int i;

#ifndef CONFIG_MMU
	flags |= __GFP_COMP;
#endif

	flags |= cachep->gfpflags;
	if (cachep->flags & SLAB_RECLAIM_ACCOUNT)
		flags |= __GFP_RECLAIMABLE;

	page = alloc_pages_exact_node(nodeid, flags | __GFP_NOTRACK, cachep->gfporder);
	if (!page)
		return NULL;

	nr_pages = (1 << cachep->gfporder);
	if (cachep->flags & SLAB_RECLAIM_ACCOUNT)
		add_zone_page_state(page_zone(page),
			NR_SLAB_RECLAIMABLE, nr_pages);
	else
		add_zone_page_state(page_zone(page),
			NR_SLAB_UNRECLAIMABLE, nr_pages);
	for (i = 0; i < nr_pages; i++)
		__SetPageSlab(page + i);

	if (kmemcheck_enabled && !(cachep->flags & SLAB_NOTRACK)) {
		kmemcheck_alloc_shadow(page, cachep->gfporder, flags, nodeid);

		if (cachep->ctor)
			kmemcheck_mark_uninitialized_pages(page, nr_pages);
		else
			kmemcheck_mark_unallocated_pages(page, nr_pages);
	}

	return page_address(page);
}
```

nodeid非负时，分配器试图从相同的内存节点给发出的请求进行分配。

利用kmem_freepages()释放内存。只有在下列情况下才会调用释放函数：当可用内存变得紧缺时，系统试图释放出更多内存以供使用；或者当高速缓存显示地被撤销时。

```c
struct kmem_cache *
kmem_cache_create (const char *name, size_t size, size_t align,	unsigned long flags, void (*ctor)(void *))
```

此函数创建一个新的高速缓存。name 存放高速缓存的名字；size 是高速缓存中每个元素的大小；align 是slab内第一个对象的偏移，用来确保在业内进行特定的对齐，通常 0 就可以满足要求，也就是标准对齐；flags 是可选的设置项，用来控制高速缓存的行为，为 0 表示无特殊的行为，或者与以下标志中的一个或多个进行“或”运算：

- SLAB_HWCACHE_ALIGN：slab层把一个slab内的所有对象按高速缓存行对齐。防止了“错误的共享”（两个或多个对象位于不同的内存对象，但映射到相同的高速缓存行）。提供性能，以空间换时间。
- SLAB_POISON：slab层用已知的值（a5a5a5）填充slab，即所谓的“中毒”，有利于对未初始化内存的访问。
- SLAB_RED_ZONE：slab层在已分配的内存周围插入“红色警戒区”以探测缓冲越界。
- SLAB_PANIC：当分配失败时提醒slab层。这在要求分配只能成功时非常有用。
- SLAB_CACHE_DMA：slab层使用可以执行DMA的内存给每个slab分配空间。只有在分配的对象用于DMA，而且必须驻留在ZONE_DMA区时才需要这个标志。

最后一个参数ctor是高速缓存的构造函数。在新的页追加到高速缓存时，此函数才被调用。

函数会睡眠。

撤销一个高速缓存，调用：

```C
void kmem_cache_destroy(struct kmem_cache *cachep)
```

撤销函数通常在模块的注销代码中被调用。同样，也不能从中断上下文中调用这个函数，因为它也可能睡眠。调用该函数之前必须确保存在以下两个条件:1、高速缓存中的所有slab都必须为空。2、在调用kmem_cache_destroy过程中(更不用说在调用之后了)不再访问这个高速缓存。调用者必须确保这种同步。

```C
/*从缓存中分配*/
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
/*释放*/
void kmem_cache_free(struct kmem_cache *cachep, void *objp)
```

```C
/*ARP相关 创建、分配、释放*/
tbl->kmem_cachep =
			kmem_cache_create(tbl->id, tbl->entry_size, 0,
					  SLAB_HWCACHE_ALIGN|SLAB_PANIC,
					  NULL);
n = kmem_cache_zalloc(tbl->kmem_cachep, GFP_ATOMIC);
kmem_cache_free(neigh->tbl->kmem_cachep, neigh);
```

### 在栈上的静态分配

用户空间能够奢侈地负担起非常大的栈，而且栈空间还可以动态增长，相反，内核却不能这么奢侈---内核栈小且固定。当给每个进程分配一个固定大小的小栈后，不但可以减少内存的消耗，而且内核页无须负担太重的管理任务。

每个进程的内核栈大小既依赖体系结构，也与编译时的选项有关。历史上，每个进程都有两页的内核栈。因为32位和64位体系结构的页面大小分别是4KB和8KB，所以通常它们的内核栈的大小分别是8KB和16KB。但是，在2.6系列内核的早期，引人了一个选项设置单页内核栈。当激活这个选项时，每个进程的内核栈只有一页那么大，根据体系结构的不同，或为4KB,或为8KB。这样做出于两个原因：首先，可以让每个进程减少内存消耗。其次，也是最重要的，随着机器运行时间的增加，寻找两个未分配的、连续的页变得越来越困难。物理内存渐渐变为碎片，因此，给一个新进程分配虚拟内存(vM)的压力也在增大。

还有一个更复杂的原因每个进程的整个调用链必须放在自己的内核栈中。不过，中断处理程序也曾经使用它们所中断的进程的内核栈，这样，中断处理程序也要放在内核栈中。这当然有效而简单，但是，这同时会把更严格的约束条件加在这可怜的内核栈上。当我们转而使用只有一个页面的内核栈时，中断处理程序就不放在栈中了。

为了矫正这个问题，内核开发者们实现了一个新功能:中断栈。中断栈为每个进程提供一个用干中断处理程序的栈。有了这个选项，中断处理程序不用再和被中断进程共享一个内核栈，它们可以使用自己的栈了。对每个进程来说仅仅耗费了一页而已。

总的来说，内核栈可以是1页，也可以是2页，这取决于编译时配置选项。栈大小因此在4~16KB的范围内。历史上，中断处理程序和被中断进程共享一个栈。当1页栈的选项激活时中断处理程序获得了自己的栈。在任何情况下，无限制的递归和alloc()显然是不被允许的。

### 高端内存的映射

根据定义，在高端内存中的页不能永久地映射到内核地址空间上。因此，通过alloc_pages()函数以__GFP_HIGHMEM标志获得的页不可能有逻辑地址。

在x86体系结构上，高于896MB的所有物理内存的范围大都是高端内存，它并不会永久地或自动地映射到内核地址空间，尽管X86处理器能够寻址物理RAM的范围达到4GB（启用PAE可以寻址到64GB）。一旦这些页被分配，就必须映射到内核的逻辑地址空间上。在x86上，高端内存中的页被映射到3GB-4GB。

```c
void *kmap(struct page *page)
```

永久映射一个给定的page结构到内核地址空间，定义在文件<linux/highmem.h>中。此函数在高端内存或低端内存上都能用。如果page结构对应的是低端内存中的一页，函数只会单纯地返回该页的虚拟地址。如果页位于高端内存，则会建立一个永久映射，再返回地址。这个函数可以睡眠，因此只能用在进程上下文中。    由于允许永久映射的数量是有限的(如果没有这个限制，就不必搞得这么复杂，把所有内存通通映射为永久内存就行了)，当不再需要高端内存时，应该解除映射，可通过下列函数完成:

```c
void kunmap(struct page *page)
```

```c
void *kmap_atomic(struct page *page, enum km_type type)
```

临时映射，不可睡眠。内核原子地把高端内存中的一个页映射到某个保留的映射中。参数type是下列枚举之一，描述了临时映射的目的。定义于<asm/kmap_types.h>中：

```c
enum km_type {
	KM_BOUNCE_READ,
	KM_SKB_SUNRPC_DATA,
	KM_SKB_DATA_SOFTIRQ,
	KM_USER0,
	KM_USER1,
	KM_BIO_SRC_IRQ,
	KM_BIO_DST_IRQ,
	KM_PTE0,
	KM_PTE1,
	KM_IRQ0,
	KM_IRQ1,
	KM_SOFTIRQ0,
	KM_SOFTIRQ1,
	KM_PPC_SYNC_PAGE,
	KM_PPC_SYNC_ICACHE,
	KM_KDB,
	KM_TYPE_NR
};
```

此函数可用在中断上下文和其他不能重新调度的地方。它也禁止内核抢占，因为映射对每个处理器都是唯一的（调度可能对哪个处理器执行哪个进程做变动）。可通过下面的函数取消映射：

```c
void kunmap_atomic(void *kvaddr, enum km_type type)
```

下一个临时映射可自动覆盖前一个映射，此函数可不调用。

### 每个CPU的分配

支持SMP的现代操作系统使用每个CPU上的数据，对于给定的处理器其数据是唯一的。一般来说，每个CPU的数据存放在一个数组中。数组中的每一项对应着系统上一个存在的处理器。按当前处理器号确定这个数组的当前元素。可声明如下：

```c
unsigned long my_percpu[NR_CPUS];
```

然后，按如下方式访问：

```c
int cpu;
cpu = get_cpu();
my_percpu(cpu)++;
printk("my_percpu on cpu=%d is %lu\n",cpu, my_percpu(cpu));
put_cpu();
```

注意，上面的代码中并没有出现锁，这是因为所操作的数据对当前处理器来说是唯一的。

现在，内核抢占成为了唯一需要关注的问题，内核抢占会引起下面提到的两个问题：

- 如果你的代码被其他处理器抢占并重新调度，那么这时CPU变量就会无效，因为它指向的是错误的处理器(通常，代码获得当前处理器后是不可以睡眠的)。
- 如果另一个任务抢占了你的代码，那么有可能在同一个处理器上发生并发访问my_percpu的情况，显然这属于一个竞争条件。

调用get_cpu()时，就已经禁止了内核抢占。相应的在调用put_cpu()时又会重新激活当前处理器号。注意，只要你总使用上述方法来保护数据安全，那么，内核抢占就不需要你自己去禁止。

### 新的每个CPU的分配

2.6内核为了方便创建和操作每个CPU数据，而引进了新的操作接口，称作percpu。头文件<linux/percpu.h>声明了所有的接口操作例程，可以在文件<mm/slab.c>和<asm/percpu.h>中找到它们的定义。

在编译时定义每个CPU：

```c
DEFINE_PER_CPU(type, name);
```

这个语句为系统中的每一个处理器都创建了一个类型为type，名字为name的变量实例，如果需要在别处声明变量，以防编译时警告，则：

```c
DECLARE_PER_CPU(type, name);
```

```c
get_cpu_var(name)++;/*增加该处理器上的name值，并禁止抢占*/
put_cpu_var(name);/*完成，重新激活内核抢占*/
per_cpu(name, cpu)++;/*增加指定处理器上的name变量的值，不会禁止抢占，也不会提供任何形式的锁保护*/
```

这些编译时每个CPU数据的例子并不能在模块内使用，因为连接程序实际上将它们创建在一个唯一的可执行段中（.data.percpu）。需要从模块中访问，则需要动态创建这些数据。

内核实现每个CPU数据的动态分配方法类似于kmalloc()。

```C
/*原型在<linux/percpu.h>*/
#define alloc_percpu(type)	\
	(typeof(type) __percpu *)__alloc_percpu(sizeof(type), __alignof__(type))
void __percpu *__alloc_percpu(size_t size, size_t align)
void free_percpu(void __percpu *__pdata)
```

alloc_percpu()给系统中的每个处理器分配一个指定类型对象的实例。是对__alloc_percpu()的封装（size 要分配的实际字节数，align 按多少字节对齐）。封装后按单字节对齐。

\__alignof__是gcc的一个功能，返回指定类型或value所需（或建议的，某些古怪的体系结构并没有字节对齐的要求）的对齐字节数。语义和sizeof一样。若指定value，那么将返回value的最大对齐字节数。

动态创建时返回一个指针，用来间接引用每个CPU数据：

```c
get_cpu_var(ptr);/*会禁止内核抢占*/
put_cpu_var(ptr);
```

### 使用每个CPU数据的原因

首先是减少了数据锁定。因为按照每个处理器访问每个CPU数据的逻辑，你可以不再需要任何锁。“只有这个处理器能访问这个数据”的规则纯粹是一个编程约定。编程者需要确保本地处理器只会访问它自己的唯一数据。系统本身并不存在任何措施禁止你从事欺骗活动。

第二个好处是使用每个CPU数据可以大大减少缓存失效。失效发生在处理器试图使它们的缓存保持同步时。如果一个处理器操作某个数据，而该数据又存放在其他处理器缓存中，那么存放该数据的那个处理器必须清理或刷新自己的缓存。持续不断的缓存失效称为缓存抖动，这样对系统性能影响颇大。使用每个CPU数据将使得缓存影响降至最低，因为理想情况下只会访问自己的数据。percpu接口缓存一对齐（cache-align）所有数据，以便确保在访问一个处理器的数据时，不会将另一个处理器的数据带人同一个缓存线上。

综上所述，使用每个CPU数据会省去许多〔或最小化)数据上锁，它唯一的安全要求就是要禁止内核抢占。而这点代价相比上锁要小得多，而且接口会自动帮你完成这个步骤。每个CPU数据在中断上下文或进程上下文中使用都很安全。但要注意，不能在访向每个CPU数据过程中睡眠，否则，你就可能醒来后已经到了其他处理器上了。

目前并不要求必须使用每个CPU的新接口，但是新接口在将来更容易使用，而且功能也会得到长足的优化。如果确实决定在你的内核中使用每个CPU数据，请考虑使用新接口。但是，新接口并不向后兼容之前的内核。

### 分配函数的选择

如果需要连续的物理页，就可以使用某个低级页分配器或kmalloc()。这是内核中内存分配的常用方式，也是大多数情况下你自己应该使用的内存分配方式。

如果想从高端内存进行分配，就使用alloc_pages()，该函数返回一个指向 struct page结构的指针，而不是一个指向某个逻辑地址的指针。因为高端内存很可能并没有被映射，因此，访问它的唯一方式就是通过相应的struct page结构。为了获得真正的指针，应该调用kmap()，把高端内存映射到内核的逻辑地址空间。

如果不需要物理上连续的页，而仅仅需要虚拟地址上连续的页，那么就使用vmalloc（不过vmalloc()相对kmalloc()来说，有一定的性能损失）。vmalloc()函数分配的内存虚地址是连续的，但它本身并不保证物理上的连续。这与用户空间的分配非常类似，它也是把物理内存块映射到连续的逻辑地址空间上。

如果要创建和撤销很多大的数据结构，那么考虑创建slab高速缓存。slab层会给每个处理器维持一个对象高速缓存(空闲链表)，这种高速缓存会极大地提高对象分配和回收的性能。slab层不是频繁地分配和释放内存，而是为你把事先分配好的对象存放到高速缓存中。当你需要一块新的内存来存放数据结构时，slab层一般无须另外去分配内存，而只需要从高速缓存中得到一个对象就可以了。

## A：第15章 进程地址空间	

## A：第16章 页高速缓存和页回写

## B：第2章 内存寻址

## B：第8章 内存管理

## B：第9章 进程地址空间

## ​B：第15章 页高速缓存

## B：第17章 回收页框

