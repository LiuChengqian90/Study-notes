## 第1章 Linux 内核简介

## 第2章 从内核出发

## 第3章 进程管理

## 第4章 进程调度

## 第5章 系统调用

## 第6章 内核数据结构

## 第7章 中断和中断管理

## 第8章 下半部和推后执行的工作

## 第9章 内核同步介绍

## 第10章 内核同步方法

## 第11章 定时器和时间管理

## 第12章 内存管理

在内核里分配内存不像在其他地方分配内存那么容易。从根本上讲，是因为内核本身不能像用户空间那样奢侈地使用内存。内核不支持简单便捷的内存分配方式。比如，内核一般不能睡眠。再加上内存分配机制不能太复杂，所以内核中获取内存要比在用户空间复杂得多。

### 页

内核把物理页作为内存管理的基本单位。尽管处理器的最小可寻址单元通常为字（甚至字节），但是，内存管理单元（MMU，管理内存并把虚拟地址转换为物理地址的硬件）通常以页为单位进行处理。正因为如此，MMU以页（page）大小为单位来管理系统中的页表。从虚拟内存的角度来看，页就是最小单位。

体系结构不同，支持的页的大小也不同，还有些体系结构甚至支持几种不同的页大小。大多数32位体系结构支持4KB的页，而64位体系结构一般支持8KB的页。也就是说，支持4KB页大小、1GB物理内存的机器上，物理内存会被划分为262144个页。

内核用struct page结构表示系统中的每个物理页，该结构位于<linux/mm_types.h>中。

```c
struct page {
	unsigned long flags;		/* Atomic flags, some possibly updated asynchronously(异步) */
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

virtual域是页的虚拟地址。通常，它是页在虚拟内存中的地址。但是高端内存并不永久地映射到内核地址空间。此时，这个域的值为NULL，需要时，必须动态地映射这些页。

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

ZONE_HIGHMEM的工作方式也差不多。能否直接映射取决于体系结构。在32位x86系统上，ZONE_HIGHMEM为高于896MB的所有物理内存^非虚拟内存^。在其他体系结构撒花姑娘，由于所有内存都被直接映射，所以ZONE_HIGHMEM为空。ZONE_HIGHMEM所在的内存就是所谓的高端内存（high memory）。系统的其余内存就是所谓的低端内存（low memory）。

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
	 * frequently read in pr0ximity to zone->lock.  It's good to
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
  | GFP_NOWAIT   | (\_\_GFP_ATOMIC & ~__GFP_HIGH)           | 类似GFP_ATOMIC，但是，调用不会退给紧急内存池。增加了内存分配失败的可能性。 |
  | GFP_ATOMIC   | __GFP_HIGH                               | 用在中断处理程序、下半部、持有自旋锁以及其他不能睡眠的地方            |
  | GFP_NOIO     | __GFP_WAIT                               | 可阻塞，但不会启动磁盘I/O。这个标志在不能引发更多磁盘I/O时能阻塞I/O代码，可能导致令人不快的递归 |
  | GFP_NOFS     | (\_\_GFP_WAIT \| \_\_GFP_IO)             | 必要时可能引起阻塞，也可能启动磁盘I/O，但是不会启动文件系统操作。       |
  | GFP_KERNEL   | (\_\_GFP_WAIT \| \_\_GFP_IO \| __GFP_FS) | 常规分配方式，可能会阻塞。在睡眠安全时用在进程上下文代码中。           |
  | GFP_USER     | (\_\_GFP_WAIT \| \_\_GFP_IO \| \_\_GFP_FS \| \_\_GFP_HARDWALL) | 常规分配方式，可能会阻塞。用于为用户空间进程分配内存。              |
  | GFP_HIGHUSER | (\_\_GFP_WAIT \| \_\_GFP_IO \| \_\_GFP_FS \| \_\_GFP_HARDWALL \| __GFP_HIGHMEM) | 从ZONE_HIGHMEM进行分配，可能会阻塞。用于为用户空间进程分配内存。   |
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

```c
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

函数会睡眠。撤销一个高速缓存，调用：

```c
void kmem_cache_destroy(struct kmem_cache *cachep)
```

撤销函数通常在模块的注销代码中被调用。同样，也不能从中断上下文中调用这个函数，因为它也可能睡眠。调用该函数之前必须确保存在以下两个条件:1、高速缓存中的所有slab都必须为空。2、在调用kmem_cache_destroy过程中(更不用说在调用之后了)不再访问这个高速缓存。调用者必须确保这种同步。

```c
/*从缓存中分配*/
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
/*释放*/
void kmem_cache_free(struct kmem_cache *cachep, void *objp)
```

```c
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

每个进程的内核栈大小既依赖体系结构，也与编译时的选项有关。历史上，每个进程都有两页的内核栈。因为32位和64位体系结构的页面大小分别是4KB和8KB，所以通常它们的内核栈的大小分别是8KB和16KB。但是，在2.6系列内核的早期，引入了一个选项设置单页内核栈。当激活这个选项时，每个进程的内核栈只有一页那么大，根据体系结构的不同，或为4KB,或为8KB。这样做出于两个原因：首先，可以让每个进程减少内存消耗。其次，也是最重要的，随着机器运行时间的增加，寻找两个未分配的、连续的页变得越来越困难。物理内存渐渐变为碎片，因此，给一个新进程分配虚拟内存(vM)的压力也在增大。

还有一个更复杂的原因每个进程的整个调用链必须放在自己的内核栈中。不过，中断处理程序也曾经使用它们所中断的进程的内核栈，这样，中断处理程序也要放在内核栈中。这当然有效而简单，但是，这同时会把更严格的约束条件加在这可怜的内核栈上。当我们转而使用只有一个页面的内核栈时，中断处理程序就不放在栈中了。

为了矫正这个问题，内核开发者们实现了一个新功能:中断栈。中断栈为每个进程提供一个用干中断处理程序的栈。有了这个选项，中断处理程序不用再和被中断进程共享一个内核栈，它们可以使用自己的栈了。对每个进程来说仅仅耗费了一页而已。

总的来说，内核栈可以是1页，也可以是2页，这取决于编译时配置选项。栈大小因此在4~16KB的范围内。历史上，中断处理程序和被中断进程共享一个栈。当1页栈的选项激活时中断处理程序获得了自己的栈。在任何情况下，无限制的递归和alloc()显然是不被允许的。

### 高端内存的映射

根据定义，在高端内存中的页不能永久地映射到内核地址空间上。因此，通过alloc_pages()函数以__GFP_HIGHMEM标志获得的页不可能有逻辑地址。

在x86体系结构上，高于896MB的所有物理内存的范围大都是高端内存，它并不会永久地或自动地映射到内核地址空间，尽管X86处理器能够寻址物理RAM的范围达到4GB（启用PAE可以寻址到64GB）。一旦这些页被分配，就必须映射到内核的逻辑地址空间上。在x86上，高端内存中的页被映射到3GB-4GB。

```c
void *kmap(struct page *page)
```

永久映射一个给定的page结构到内核地址空间，定义在文件<linux/highmem.h>中。此函数在高端内存或低端内存上都能用。如果page结构对应的是低端内存中的一页，函数只会单纯地返回该页的虚拟地址。如果页位于高端内存，则会建立一个永久映射，再返回地址。这个函数可以睡眠，因此只能用在进程上下文中。由于允许永久映射的数量是有限的(如果没有这个限制，就不必搞得这么复杂，把所有内存通通映射为永久内存就行了)，当不再需要高端内存时，应该解除映射，可通过下列函数完成:

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
  	/*SKB相关，网卡地址映射到vm，之后进行数据拷贝*/
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

```c
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

第二个好处是使用每个CPU数据可以大大减少缓存失效。失效发生在处理器试图使它们的缓存保持同步时。如果一个处理器操作某个数据，而该数据又存放在其他处理器缓存中，那么存放该数据的那个处理器必须清理或刷新自己的缓存。持续不断的缓存失效称为缓存抖动，这样对系统性能影响颇大。使用每个CPU数据将使得缓存影响降至最低，因为理想情况下只会访问自己的数据。percpu接口缓存一对齐（cache-align）所有数据，以便确保在访问一个处理器的数据时，不会将另一个处理器的数据带入同一个缓存线上。

综上所述，使用每个CPU数据会省去许多〔或最小化)数据上锁，它唯一的安全要求就是要禁止内核抢占。而这点代价相比上锁要小得多，而且接口会自动帮你完成这个步骤。每个CPU数据在中断上下文或进程上下文中使用都很安全。但要注意，不能在访向每个CPU数据过程中睡眠，否则，你就可能醒来后已经到了其他处理器上了。

目前并不要求必须使用每个CPU的新接口，但是新接口在将来更容易使用，而且功能也会得到长足的优化。如果确实决定在你的内核中使用每个CPU数据，请考虑使用新接口。但是，新接口并不向后兼容之前的内核。

### 分配函数的选择

如果需要连续的物理页，就可以使用某个低级页分配器或kmalloc()。这是内核中内存分配的常用方式，也是大多数情况下你自己应该使用的内存分配方式。

如果想从高端内存进行分配，就使用alloc_pages()，该函数返回一个指向 struct page结构的指针，而不是一个指向某个逻辑地址的指针。因为高端内存很可能并没有被映射，因此，访问它的唯一方式就是通过相应的struct page结构。为了获得真正的指针，应该调用kmap()，把高端内存映射到内核的逻辑地址空间。

如果不需要物理上连续的页，而仅仅需要虚拟地址上连续的页，那么就使用vmalloc（不过vmalloc()相对kmalloc()来说，有一定的性能损失）。vmalloc()函数分配的内存虚地址是连续的，但它本身并不保证物理上的连续。这与用户空间的分配非常类似，它也是把物理内存块映射到连续的逻辑地址空间上。

如果要创建和撤销很多大的数据结构，那么考虑创建slab高速缓存。slab层会给每个处理器维持一个对象高速缓存(空闲链表)，这种高速缓存会极大地提高对象分配和回收的性能。slab层不是频繁地分配和释放内存，而是为你把事先分配好的对象存放到高速缓存中。当你需要一块新的内存来存放数据结构时，slab层一般无须另外去分配内存，而只需要从高速缓存中得到一个对象就可以了。

## 第13章 虚拟文件系统

## 第14章 块I/O层

## 第15章 进程地址空间

内核除了管理本身的内存外，还必须管理用户空间的内存，即进程地址空间，也就是系统中每个用户空间进程所看到的内存。Linux采用虚拟内存技术，因此，系统中的所有进程之间以虚拟方式共享内存。对一个进程而言，它好像都可以访问整个系统的所有物理内存。更重要的是，即使单独一个进程，它拥有的地址空间也可以远远大于系统物理内存。

### 地址空间

进程地址空间由进程可寻址的虚拟内存组成，更为重要的特点是内核允许进程使用这种虚拟内存的地址。每个进程都有一个32位或64位的平坦（flat）地址空间，空间的具体大小取决于体系结构。“平坦”指的是地址空间范围是一个独立的连续区间（比如，地址从0x00000000扩展到0xffffffff的32位地址空间）。

一些操作系统提供了段地址空间，这种地址空间并非是一个独立的线性区域，而是被分段的，但现代采用虚拟内存的操作系统通常都使用平坦地址空间而不是分段式的内存模式。通常情况下，每个进程都有唯一的这种平坦地址空间。一个进程的地址空间与另一个进程的地址空间即使有相同的内存地址，实际上也彼此互不相干。我们称这样的进程为线程。

内存地址是一个给定的值，它要在地址空间范围之内，比如0x4021f000。这个值表示的是进程32位地址空间中的一个特定的字节。尽管一个进程可以寻址4GB的虚拟内存（在32位的地址空间中），但这并不代表它就有权访问所有的虚拟地址。在地址空间中，我们更为关心的是一些虚拟内存的地址区间，它们可被进程访问。这些可被访问的合法地址空间称为内存区域（memory areas）。通过内核，进程可以给自己的地址空间动态地添加或减少内存区域。

进程只能访问有效内存区域内的内存地址。每个内存区域也具有相关权限，如对相关进程有可读、可写、可执行属性。如果一个进程访问了不在有效范围中的内存区域，或以不正确的方式访问了有效地址，那么内核就会终止该进程，并返回“段错误”信息。

内存区域可以包含各种内存对象，比如:

- 可执行文件代码的内存映射，称为代码段（text section）。
- 可执行文件的“已初始化”全局变量的内存映射，称为数据段（data section）。
- 包含“未初始化”全局变量，也就是bss段（Block Started by Symbol，符号开始的块）的零页(页面中的信息全部为0值，所以可用于映射bss段等目的)的内存映射。
- 用于进程用户空间栈（不要和进程内核栈混淆，进程的内核栈独立存在并由内核维护）的零页的内存映射。
- 每一个诸如C库或动态连接程序等共享库的代码段、数据段和bss也会被载入进程的地址空间。
- 任何内存映射文件。
- 任何共享内存段。
- 任何匿名的内存映射，比如由malloc()分配的内存。

进程地址空间中的任何有效地址都只能位于唯一的区域，这些内存区域不能相互覆盖。可以看到，在执行的进程中，每个不同的内存片段都对应一个独立的内存区域:栈、对象代码、全局变量、被映射的文件等。

### 内存描述符

内核使用内存描述符结构体表示进程的地址空间，该结构包含了和进程地址空间有关的全部信息。内存描述符由mm_struct结构体表示，定义在<linux/sched.h>中。

```c
struct mm_struct {
	struct vm_area_struct * mmap;		/*（虚拟）内存区域*/
	struct rb_root mm_rb;				/*VMA形成的红黑树*/
	struct vm_area_struct * mmap_cache;	/*最近使用的内存区域*/
#ifdef CONFIG_MMU
	unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
	void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
#endif
	unsigned long mmap_base;		/* map映射的基地址 */
	unsigned long task_size;		/* 任务空间大小 */
	unsigned long cached_hole_size; 	/* free_area_cache下最大的空洞 */
	unsigned long free_area_cache;		/* 地址空间第一个空洞 */
	pgd_t * pgd;				/*全局页表，进行地址转换*/
	atomic_t mm_users;			/*使用地址空间的用户数*/
	atomic_t mm_count;			/* 对此结构有多少引用 */
	int map_count;				/* 内存区域数 */
	struct rw_semaphore mmap_sem;	/*内存区域信号量*/
	spinlock_t page_table_lock;		/* 页表锁 */
	struct list_head mmlist;		/*所有mm_struct形成的链表*/
	unsigned long hiwater_rss;	/* RSS高水位*/
	unsigned long hiwater_vm;	/* 虚拟内存高水位 */

	unsigned long total_vm, locked_vm, shared_vm, exec_vm;
	unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
	unsigned long start_code, end_code, start_data, end_data;	/*代码段的首地址、尾地址，数据段的首地址、尾地址*/
	unsigned long start_brk, brk, start_stack;	/*堆的首地址、尾地址、进程栈的首地址*/
	unsigned long arg_start, arg_end, env_start, env_end;	/*命令行参数的首地址、尾地址，环境变量的首地址、尾地址*/
	unsigned long saved_auxv[AT_VECTOR_SIZE]; /* 保存的auxv /proc/PID/auxv */
	/*
	 * Special counters, in some configurations protected by the
	 * page_table_lock, in other configurations by being atomic.
	 */
	struct mm_rss_stat rss_stat;
	struct linux_binfmt *binfmt;
	cpumask_t cpu_vm_mask;		/*lazy TLB交换掩码*/	
	mm_context_t context;		/* 体系结构特殊数据 */
	/* Swap token stuff */
	/*
	 * Last value of global fault stamp as seen by this process.
	 * In other words, this value gives an indication of how long
	 * it has been since this task got the token.
	 * Look at mm/thrash.c
	 */
	unsigned int faultstamp;
	unsigned int token_priority;
	unsigned int last_interval;

	unsigned long flags; /* 状态标志 */
	struct core_state *core_state; /* 核心转储支持 coredumping support */
#ifdef CONFIG_AIO
	spinlock_t		ioctx_lock;		/*AIO I/O链表锁*/
	struct hlist_head	ioctx_list;	/*AIO I/O链表*/
#endif
#ifdef CONFIG_MM_OWNER
	/*
	 * "owner" points to a task that is regarded as the canonical
	 * user/owner of this mm. All of the following must be true in
	 * order for it to be changed:
	 *
	 * current == mm->owner
	 * current->mm != mm
	 * new_owner->mm == mm
	 * new_owner->alloc_lock is held
	 */
	struct task_struct *owner;
#endif

#ifdef CONFIG_PROC_FS
	/* store ref to file /proc/<pid>/exe symlink points to */
	struct file *exe_file;
	unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
	struct mmu_notifier_mm *mmu_notifier_mm;
#endif
};
```

mm_users域记录正在使用该地址的进程数目。如果两个线程共享该地址空间，那么mm_users的值便等于2；mm_count域是mm_struct结构体的主引用计数。所有的mm_users都等于mm_count的增加量。这样，在前面的例子中，mm_count就仅仅为1。如果有9个线程共享某个地址空间，那么mm_user将会是9，而mm_count的值将再次为1。当mm_user的值减为0（所有使用该地址的线程全部退出）时，mm_count域的值才变为0。当mm_count的值等于0，说明已经没有任何指向该结构体的引用了，这时该结构体会被撤销。当内核在一个地址空间上操作，并需要使用与该地址相关联的引用计数时，内核便增加mm_count。内核同时使用这两个计数器是为了区别主使用计数器和使用该地址空间的进程的数目。

mmap和mm_rt这两个不同数据结构体描述的对象是相同的:该地址空间中的全部内存区域。但是前者以链表形式存放而后者以红-黑树的形式存放。

内核通常会避免使用两种数据结构组织同一种数据，但此处内核这样的冗余确实派得上用场。mmap结构体作为链表，利于简单、高效地遍历所有元素;而mm_rb结构体作为红一黑树，更适合搜索指定元素。内核并没有复制mm_struct结构体，而仅仅被包含其中。

所有的mm_struct结构体都通过自身的mmlist域连接在一个双向链表中，该链表的首元素是init_mm内存描述符，它代表init进程的地址空间。另外，操作该链表的时候需要使用mmlist_lock锁来防止并发访问，该锁定义在文件<kernel/fork.c>中。

在进程描述符（<linux/sched.h>中定义的task_struct结构体）中，mm域存放着该进程的内存描述符，所以current->mm便指向当前进程的内存描述符。

fork()函数利用copy_mm()函数复制父进程的内存描述符，也就是current->mm域给其子进程，而子进程中的mm_struct结构体实际是通过文件kernel/fork.c中的allocate_mm()宏从mm_cachep slab缓存中分配得到的。

```c
#define allocate_mm()	(kmem_cache_alloc(mm_cachep, GFP_KERNEL))
oldmm = current->mm;
memcpy(mm, oldmm, sizeof(*mm));
```

通常，每个进程都有唯一的mm_struct结构体，即唯一的进程地址空间。

如果父进程希望和其子进程共享地址空间，可以在调用clone()时，设置CLONE_VM标志。我们把这样的进程称作线程。是否共享地址空间几乎是进程和Linux中所谓的线程间本质上的唯一区别。除此以外，Linux内核并不区别对待它们，线程对内核来说仅仅是一个共享特定资源的进程而已。

当CLONE_VM被指定后，内核就不再需要调用allocate_mm函数了，而仅仅需要在调用copy_mm函数中将mm域指向其父进程的内存描述符就可以了：

```c
/*代码有省略*/
oldmm = current->mm;
if (clone_flags & CLONE_VM) {
		atomic_inc(&oldmm->mm_users);
		mm = oldmm;
		goto good_mm;
	}
tsk->mm = mm;
```

当进程退出时，内核会调用定义在<kernel/exit.c>中的exit_mm()函数，该函数执行一些常规的撤销工作，同时更新一些统计量。其中，该函数会调用mmput()函数减少内存描述符中的mm_user用户计数，如果用户计数降到零，将调用mmdrop()函数，减少mm_count使用计数。如果使用计数也等于零了，说明该内存描迷符不再有任何使用者了，那么调用free_mm()宏通过kmem_cache_free()函数将mm_struct结构体归还到mm_cachep slab缓存中。

内核线程没有进程地址空间，也没有相关的内存描述符。所以内核线程对应的进程描述符中mm域为空。事实上，这也正是内核线程的真实含义—它们没有用户上下文。

内核线程并不需要访问任何用户空间的内存(那它们访问谁的呢?)，而且因为内核线程在用户空间中没有任何页，所以实际上它们并不需要有自己的内存描述符和页表。尽管如此，即使访问内核内存，内核线程也还是需要使用一些数据的，比如页表。为了避免内核线程为内存描述符和页表浪费内存，也为了当新内核线程运行时，避免浪费处理器周期向新地址空间进行切换，内核线程将直接使用前一个进程的内存描述符。

当一个进程被调度时，该进程的mm域指向的地址空间被装载到内存，进程描述符中的active_mm域会被更新，指向新的地址空间。内核线程没有地址空间，所以mm城为NULL。于是，当一个内核线程被调度时，内核发现它的mm域为NULL，就会保留前一个进程的地址空间，随后内核更新内核线程对应的进程描述符中的active_mm域，使其指向前一个进程的内存描述符。所以在需要时，内核线程便可以使用前一个进程的页表。因为内核线程不访问用户空间的内存，所以它们仅仅使用地址空间中和内核内存相关的信息，这些信息的含义和普通进程完全相同。

### 虚拟内存区域

内存区域由vm_area_struct结构体描述，定义在文件<linux/mm_types.h>中。内存区域在Linux内核中也经常称作虚拟内存区域（virtual memory Areas，VMAs）。

vm_area_struct结构体描述了指定地址空间内连续区间上的一个独立内存范围。内核将每个内存区域作为一个单独的内存对象管理，每个内存区域都拥有一致的属性，比如访问权限等，另外，相应的操作也都一致。按照这样的方式，每一个VMA就可以代表不同类型的内存区域（比如内存映射文件或者进程用户空间栈），下面给出该结构定义和各个域的描述:

```c
struct vm_area_struct {
	struct mm_struct * vm_mm;	/* 所属的mm_struct结构体 */
	unsigned long vm_start;		/* 区间的首地址 */
	unsigned long vm_end;		/* 区间的尾地址 */	
	struct vm_area_struct *vm_next;		/* VMA链表 */

	pgprot_t vm_page_prot;		/* 访问此区间的权限 */
	unsigned long vm_flags;		/* 标志 mm.h. */

	struct rb_node vm_rb;		/*红黑树上的该VMA节点*/
	/* 或者关联于 address_space->i_mmap字段，或者关联于 address_space->i_mmap_nonlinear字段*/
	union {
		struct {
			struct list_head list;
			void *parent;	/* aligns with prio_tree_node parent */
			struct vm_area_struct *head;
		} vm_set;
		struct raw_prio_tree_node prio_tree_node;
	} shared;

	struct list_head anon_vma_chain; /* anon_vma项，Serialized by 	mmap_sem & * page_table_lock*/
	struct anon_vma *anon_vma;	/* 匿名VMA对象 page_table_lock */
	const struct vm_operations_struct *vm_ops;	/* 相关操作表 */
	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units, *not* PAGE_CACHE_SIZE */
	struct file * vm_file;		/* 被映射的文件(can be NULL). */
	void * vm_private_data;		/* 私有数据was vm_pte (shared mem) */
	unsigned long vm_truncate_count;/* truncate_count or restart_addr */
#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
};
```

每个内存描述符都对应于进程地址空间中的唯一区间。vm_start域指向区间的首地址〔最低地址)，vm_end域指向区间的尾地址(最高地址)之后的第一个字节，也就是说，vm_start是内存区间的开始地址(它本身在区间内)，而vm_end是内存区间的结束地址（它本身在区间外），因此，vm_end-vm_start的大小便是内存区间的长度，内存区域的位置就在[vm_start, vm_end]之中。注意，在同一个地址空间内的不同内存区间不能重叠。

vm_mm域指向和VMA相关的mm_struct结构体，注意，每个VMA对其相关的mm_struct 结构体来说都是唯一的，所以即使两个独立的进程将同一个文件映射到各自的地址空间，它们分别都会有一个vm_area_struct结构体来标志自己的内存区域;反过来，如果两个线程共享一个地址空间，那么它们也同时共享其中的所有vm_area_struct结构体。

VMA标志（vm_flags）是一种位标志，定义在<linux/mm.h>，标志了内存区域所包含的页面的行为和信息。和物理页的访问权限不同，VMA标志反映了内核处理页面所需要遵守的行为准则，而不是硬件要求。而且，vm_flags同时也包含了内存区域中每个页面的信息，或内存区域的整体信息，而不是具体的独立页面。

VM_READ、VM_WRITE和VM_EXEC标志了内存区域中页面的读、写和执行权限。这些标志根据要求组合构成VMA的访问控制权限。

VM_SHARED指明了内存区域包含的映射是否可以在多进程间共享。如果设置，则成为共享映射，否则称为私有映射。

VM_IO标志内存区域中包含了对设备I/O空间的映射。通常在设备驱动程序执行mmap()函数进行I/O空间映射时才被设置，同时该标志也表示该内存区域不能被包含在任何进程的存放转存（core dump）中。VM_RESERVED标志规定了内存区域不能被换出，它也是在设备驱动程序进行映射时被设置。

VM_SEQ_READ标志暗示内核应用程序对映射内容执行有序的(线性和连续的)读操作。这样，内核可以有选择地执行预读文件。VM_RAND_READ标志的意义正好相反，暗示应用程序对映射内容执行随机的(非有序的)读操作。因此内核可以有选择地减少或彻底取消文件预读。这两个标志可以通过系统调用madvise()设置，设置参数分别是MADV_SEQUENTIAL和MADV_RANDOM。

vm_area_struct中的vm_ops域指向与指定内存区域相关的操作函数表，内核使用表中的方法操作VMA。vm_area_struct作为通用对象代表了任何类型的内存区域。函数表由vm_operations_struct结构体表示，定义在<linux/mm.h>中：

```c
struct vm_operations_struct {
  	/*指定的内存区域被加入到一个地址空间时调用*/
	void (*open)(struct vm_area_struct * area);
  	/*指定的内存区域从地址空间删除时调用*/
	void (*close)(struct vm_area_struct * area);
  	/*没有出现在物理内存中的页被访问时，页面故障处理调用*/
	int (*fault)(struct vm_area_struct *vma, struct vm_fault *vmf);
	/* notification that a previously read-only page is about to become
	 * writable, if an error is returned it will cause a SIGBUS */
	int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);
	/* called by access_process_vm when get_user_pages() fails, typically
	 * for use by special VMAs that can switch between memory and hardware
	 */
	int (*access)(struct vm_area_struct *vma, unsigned long addr,
		      void *buf, int len, int write);
#ifdef CONFIG_NUMA
	/*
	 * set_policy() op must add a reference to any non-NULL @new mempolicy
	 * to hold the policy upon return.  Caller should pass NULL @new to
	 * remove a policy and fall back to surrounding context--i.e. do not
	 * install a MPOL_DEFAULT policy, nor the task or system default
	 * mempolicy.
	 */
	int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);
	/*
	 * get_policy() op must add reference [mpol_get()] to any policy at
	 * (vma,addr) marked as MPOL_SHARED.  The shared policy infrastructure
	 * in mm/mempolicy.c will do this automatically.
	 * get_policy() must NOT add a ref if the policy at (vma,addr) is not
	 * marked as MPOL_SHARED. vma policies are protected by the mmap_sem.
	 * If no [shared/vma] mempolicy exists at the addr, get_policy() op
	 * must return NULL--i.e., do not "fallback" to task or system default
	 * policy.
	 */
	struct mempolicy *(*get_policy)(struct vm_area_struct *vma,
					unsigned long addr);
	int (*migrate)(struct vm_area_struct *vma, const nodemask_t *from,
		const nodemask_t *to, unsigned long flags);
#endif
};
```

内存描述符中，mmap和mm_rb，这两个域各自独立地指向与内存描述符相关的全体内存区域对象。包含完全相同的vm_area_struct结构体指针，仅仅组织方法不同。

mmap域使用单链表连接所有的内存区域对象。vm_area_struct结构体通过自身的vm_next域被连入链表，所有的区域按地址增长的方向排序，mmap域指向链表中第一个内存区域，链中最后一个结构体指针指向空。

mm_rb域使用红-黑树连接所有的内存区域对象。mm_rb域指向红-黑树的根节点，地址空间中每一个vm_aera_struct结构体通过自身的vm_rb域连接到树中。

链表用于需要追历全部节点的时候，而红-黑树适用于在地址空间中定位特定内存区域的时候。内核为了内存区域上的各种不同操作都能获得高性能，所以同时使用了这两种数据结构。

可以使用/proc文件系统和pmap(1)根据查看指定进程的内存空间和其中所含的内存区域。下面列出某个进程地址空间中包含的内存区域。其中有代码段、数据段和bss段等。若进程与C库动态链接，那么该地址空间中还分别包含libc.so和ld.so对应的上述三种内存区域。此外，地址空间中还要包含进程栈对应的内存区域。

```c
#include<stdio.h>
#include<stdlib.h>

int  a;
int  b=1;

void main()
{
    int  n = 0;
    char *p1 = NULL;
    char *p2 = NULL;
    const int s = 10;
    p1 = (char*)malloc(200);
    p2 = "hello";

    printf("main  %p\n",main);
    printf("未初始化 a   %p\n",&a);
    printf("初始化   b   %p\n",&b);
    printf("局部变量 n   %p\n",&n);
    printf("动态内存 p1  %p\n",p1);
    printf("常量     s   %p\n",&s);
    printf("常字符串 p2  %p\n",p2);

    while(1)
    {}
}
```

```shell
[root@lcq-com centos]# ./test 
main  0x40057d
未初始化 a   0x601044
初始化   b   0x60103c
局部变量 n   0x7ffe5de20b0c
动态内存 p1  0x11d2010
常量     s   0x7ffe5de20b08
常字符串 p2  0x4006e0

[root@lcq-com centos]# ps aux| grep test
root     22611 96.0  0.0   4288   344 pts/0    R+   03:22   0:22 ./test
root     22669  0.0  0.0 112644   968 pts/1    R+   03:22   0:00 grep --color=auto test
[root@lcq-com centos]# pmap 22611
22611:   ./test
0000000000400000      4K r-x-- test
0000000000600000      4K r---- test
0000000000601000      4K rw--- test
00000000011d2000    132K rw---   [ anon ]
00007f73a174c000   1756K r-x-- libc-2.17.so
00007f73a1903000   2044K ----- libc-2.17.so
00007f73a1b02000     16K r---- libc-2.17.so
00007f73a1b06000      8K rw--- libc-2.17.so
00007f73a1b08000     20K rw---   [ anon ]
00007f73a1b0d000    128K r-x-- ld-2.17.so
00007f73a1d21000     12K rw---   [ anon ]
00007f73a1d2b000      8K rw---   [ anon ]
00007f73a1d2d000      4K r---- ld-2.17.so
00007f73a1d2e000      4K rw--- ld-2.17.so
00007f73a1d2f000      4K rw---   [ anon ]
00007ffe5de02000    132K rw---   [ stack ]
00007ffe5de58000      8K r-x--   [ anon ]
ffffffffff600000      4K r-x--   [ anon ]
 total             4292K
[root@lcq-com centos]# cat /proc/22611/maps 
00400000-00401000 r-xp 00000000 fd:01 37818796                           /home/centos/test
00600000-00601000 r--p 00000000 fd:01 37818796                           /home/centos/test
00601000-00602000 rw-p 00001000 fd:01 37818796                           /home/centos/test
011d2000-011f3000 rw-p 00000000 00:00 0                                  [heap]
7f73a174c000-7f73a1903000 r-xp 00000000 fd:01 38400                      /usr/lib64/libc-2.17.so
7f73a1903000-7f73a1b02000 ---p 001b7000 fd:01 38400                      /usr/lib64/libc-2.17.so
7f73a1b02000-7f73a1b06000 r--p 001b6000 fd:01 38400                      /usr/lib64/libc-2.17.so
7f73a1b06000-7f73a1b08000 rw-p 001ba000 fd:01 38400                      /usr/lib64/libc-2.17.so
7f73a1b08000-7f73a1b0d000 rw-p 00000000 00:00 0 
7f73a1b0d000-7f73a1b2d000 r-xp 00000000 fd:01 35193                      /usr/lib64/ld-2.17.so
7f73a1d21000-7f73a1d24000 rw-p 00000000 00:00 0 
7f73a1d2b000-7f73a1d2d000 rw-p 00000000 00:00 0 
7f73a1d2d000-7f73a1d2e000 r--p 00020000 fd:01 35193                      /usr/lib64/ld-2.17.so
7f73a1d2e000-7f73a1d2f000 rw-p 00021000 fd:01 35193                      /usr/lib64/ld-2.17.so
7f73a1d2f000-7f73a1d30000 rw-p 00000000 00:00 0 
7ffe5de02000-7ffe5de23000 rw-p 00000000 00:00 0                          [stack]
7ffe5de58000-7ffe5de5a000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

/proc下maps每行数据格式如下：

```c
开始-结束	访问权限	偏移	主设备号：次设备号	i节点		文件
```

前两行为可执行对象的代码段、数据段（第三行或为bss段），之后为链接程序的代码段、数据段和bss段，倒数第三行(stack)是进程的栈，后两行 vdso、vsyscall 为系统的快速调用。

注意，代码段具有我们所要求的可读且可执行权限；另一方面，数据段和bss(它们都包含全局数据变量)具有可读、可写但不可执行权限。而堆栈则可读、可写，甚至还可执行——虽然这点并不常用到。

该进程的全部地址空间大约为4292KB，但是只有大约504KB的内存区域是可写和私有的（物理内存 ，pmap -X pid）。如果一片内存范围是共享的或不可写的，那么内核只需要在内存中为文件(backing file)保留一份映射。对于共享映射来说，这样做没什么特别的，但是对于不可写内存区域也这样做，就有些让人奇怪了。如果考虑到映射区域不可写意味着该区域不可被改变(映射只用来读)，就应该清楚只把该映像读入一次是很安全的。所以C库在物理内存中仅仅需要占用316KB空间，而不需要为每个使用C库的进程在内存中都保存一个316KB的空间。进程访问了4292KB的数据和代码空间，然而仅仅消耗了504KB的物理内存，可以看出利用这种共享不可写内存的方法节约了大量的内存空间。

注意没有映射文件的内存区域的设备标志为00:00，索引接点标志也为0，这个区域就是零页——零页映射的内容全为零。如果将零页映射到可写的内存区域，那么该区域将全被初始化为0。这是零页的一个重要用处，而bss段需要的就是全0的内存区域。由于内存未被共享，所以只要一有进程写该处数据，那么该处数据就将被拷贝出来(就是我们所说的写时拷贝)，然后才被更新。

内核时常需要在某个内存区域上执行一些操作，比如某个指定地址是否包含在某个内存区域中。这类操作非常频繁，为了方便执行这类对内存区域的操作，内核定义了许多的辅助函数。它们都声明在文件<linux/mm.h>中。

进程编译后可执行文件，分布的段（代码段、数据段、bss段）如下所示：

```shell
[root@lcq-com centos]# readelf -s test

Symbol table '.dynsym' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND malloc@GLIBC_2.2.5 (2)

Symbol table '.symtab' contains 68 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000400238     0 SECTION LOCAL  DEFAULT    1 
     2: 0000000000400254     0 SECTION LOCAL  DEFAULT    2 
     3: 0000000000400274     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000400298     0 SECTION LOCAL  DEFAULT    4 
     5: 00000000004002b8     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000400330     0 SECTION LOCAL  DEFAULT    6 
     7: 0000000000400376     0 SECTION LOCAL  DEFAULT    7 
     8: 0000000000400380     0 SECTION LOCAL  DEFAULT    8 
     9: 00000000004003a0     0 SECTION LOCAL  DEFAULT    9 
    10: 00000000004003b8     0 SECTION LOCAL  DEFAULT   10 
    11: 0000000000400418     0 SECTION LOCAL  DEFAULT   11 
    12: 0000000000400440     0 SECTION LOCAL  DEFAULT   12 
    13: 0000000000400490     0 SECTION LOCAL  DEFAULT   13 
    14: 00000000004006c4     0 SECTION LOCAL  DEFAULT   14 
    15: 00000000004006d0     0 SECTION LOCAL  DEFAULT   15 
    16: 000000000040076c     0 SECTION LOCAL  DEFAULT   16 
    17: 00000000004007a0     0 SECTION LOCAL  DEFAULT   17 
    18: 0000000000600e10     0 SECTION LOCAL  DEFAULT   18 
    19: 0000000000600e18     0 SECTION LOCAL  DEFAULT   19 
    20: 0000000000600e20     0 SECTION LOCAL  DEFAULT   20 
    21: 0000000000600e28     0 SECTION LOCAL  DEFAULT   21 
    22: 0000000000600ff8     0 SECTION LOCAL  DEFAULT   22 
    23: 0000000000601000     0 SECTION LOCAL  DEFAULT   23 
    24: 0000000000601038     0 SECTION LOCAL  DEFAULT   24 
    25: 0000000000601040     0 SECTION LOCAL  DEFAULT   25 
    26: 0000000000000000     0 SECTION LOCAL  DEFAULT   26 
    27: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    28: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   20 __JCR_LIST__
    29: 00000000004004c0     0 FUNC    LOCAL  DEFAULT   13 deregister_tm_clones
    30: 00000000004004f0     0 FUNC    LOCAL  DEFAULT   13 register_tm_clones
    31: 0000000000400530     0 FUNC    LOCAL  DEFAULT   13 __do_global_dtors_aux
    32: 0000000000601040     1 OBJECT  LOCAL  DEFAULT   25 completed.6344
    33: 0000000000600e18     0 OBJECT  LOCAL  DEFAULT   19 __do_global_dtors_aux_fin
    34: 0000000000400550     0 FUNC    LOCAL  DEFAULT   13 frame_dummy
    35: 0000000000600e10     0 OBJECT  LOCAL  DEFAULT   18 __frame_dummy_init_array_
    36: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS test.c
    37: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    38: 0000000000400890     0 OBJECT  LOCAL  DEFAULT   17 __FRAME_END__
    39: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   20 __JCR_END__
    40: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 
    41: 0000000000600e18     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_end
    42: 0000000000600e28     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
    43: 0000000000600e10     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_start
    44: 0000000000601000     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
    45: 00000000004006c0     2 FUNC    GLOBAL DEFAULT   13 __libc_csu_fini
    46: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
    47: 0000000000601038     0 NOTYPE  WEAK   DEFAULT   24 data_start
    48: 000000000060103c     4 OBJECT  GLOBAL DEFAULT   24 b
    49: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT   24 _edata
    50: 00000000004006c4     0 FUNC    GLOBAL DEFAULT   14 _fini
    51: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@@GLIBC_2.2.5
    52: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
    53: 0000000000601038     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
    54: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    55: 00000000004006d8     0 OBJECT  GLOBAL HIDDEN    15 __dso_handle
    56: 00000000004006d0     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
    57: 0000000000400650   101 FUNC    GLOBAL DEFAULT   13 __libc_csu_init
    58: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND malloc@@GLIBC_2.2.5
    59: 0000000000601048     0 NOTYPE  GLOBAL DEFAULT   25 _end
    60: 0000000000400490     0 FUNC    GLOBAL DEFAULT   13 _start
    61: 0000000000601044     4 OBJECT  GLOBAL DEFAULT   25 a
    62: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    63: 000000000040057d   210 FUNC    GLOBAL DEFAULT   13 main
    64: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
    65: 0000000000601040     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
    66: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
    67: 0000000000400418     0 FUNC    GLOBAL DEFAULT   11 _init
    
    [root@lcq-com centos]# readelf -S test
There are 30 section headers, starting at offset 0x1a28:
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002b8  000002b8
       0000000000000078  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400330  00000330
       0000000000000046  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           0000000000400376  00000376
       000000000000000a  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000400380  00000380
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             00000000004003a0  000003a0
       0000000000000018  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000004003b8  000003b8
       0000000000000060  0000000000000018  AI       5    12     8
  [11] .init             PROGBITS         0000000000400418  00000418
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000400440  00000440
       0000000000000050  0000000000000010  AX       0     0     16
  [13] .text             PROGBITS         0000000000400490  00000490
       0000000000000234  0000000000000000  AX       0     0     16
  [14] .fini             PROGBITS         00000000004006c4  000006c4
       0000000000000009  0000000000000000  AX       0     0     4
  [15] .rodata           PROGBITS         00000000004006d0  000006d0
       000000000000009b  0000000000000000   A       0     0     8
  [16] .eh_frame_hdr     PROGBITS         000000000040076c  0000076c
       0000000000000034  0000000000000000   A       0     0     4
  [17] .eh_frame         PROGBITS         00000000004007a0  000007a0
       00000000000000f4  0000000000000000   A       0     0     8
  [18] .init_array       INIT_ARRAY       0000000000600e10  00000e10
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .fini_array       FINI_ARRAY       0000000000600e18  00000e18
       0000000000000008  0000000000000000  WA       0     0     8
  [20] .jcr              PROGBITS         0000000000600e20  00000e20
       0000000000000008  0000000000000000  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000600e28  00000e28
       00000000000001d0  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000600ff8  00000ff8
       0000000000000008  0000000000000008  WA       0     0     8
  [23] .got.plt          PROGBITS         0000000000601000  00001000
       0000000000000038  0000000000000008  WA       0     0     8
  [24] .data             PROGBITS         0000000000601038  00001038
       0000000000000008  0000000000000000  WA       0     0     4
  [25] .bss              NOBITS           0000000000601040  00001040
       0000000000000008  0000000000000000  WA       0     0     4
  [26] .comment          PROGBITS         0000000000000000  00001040
       000000000000002d  0000000000000001  MS       0     0     1
  [27] .shstrtab         STRTAB           0000000000000000  0000106d
       0000000000000108  0000000000000000           0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00001178
       0000000000000660  0000000000000018          29    45     8
  [29] .strtab           STRTAB           0000000000000000  000017d8
       0000000000000250  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```



### 操作内存区域

find_vma()函数：找到给定的内存地址属于哪一个内存区域。定义在<mmap.c>中。该函数在指定的地址空间中搜索第一个vm_end大于addr的内存区域。换句话说，该函数寻找第一个包含addr或首地址大于addr的内存区域，如果没有发现这样的区域，该函数返回NULL；否则返回指向匹配的内存区域的vm_area_struct结构体指针。注意，由于返回的VMA首地址可能大于addr，所以指定的地址并不一定就包含在返回的VMA中。因为很有可能在对某个VMA执行操作后，还有其他更多的操作会对该VMA接着进行操作，所以find_vma()函数返回的结果披缓存在内存描述符的mmap-cache域中。（先搜索缓存，因此结果可能不正确）

find_vma_prev() 返回第一个小于addr的VMA。

### mmap()和do_mmap()：创建地址区间

内核使用do_mmap()函数创建一个新的线性地址区间。但是说该函数创建了一个新VMA井不非常谁确，因为如果创建的地址区间和一个已经存在的地址区间相邻，并且它们具有相同的访问权限的话，两个区间将合并为一个。如果不能合并，就确实需要创建一个新的VMA了。但无论哪种情况，do_mmap()函数都会将一个地址区间加入到进程的地址空间中——无论是扩展已存在的内存区域还是创建一个新的区域。函数定义在文件<linux/mm.h>中。

```c
static inline unsigned long do_mmap(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long offset)
```

该函数映射由file指定的文件，具体映射的是文件中从偏移offset处开始，长度为len字节的范围内的数据。如果file参数是NULL并且offset参数也是0，那么就代表这次映射没有和文件相关，该情况称作匿名映射（anonymous mapping）。如果指定了文件名和偏移量，那么该映射称为文件映射（file-backed mapping）。

addr是可选参数，它指定搜索空闲区域的起始位置。

prot参数指定内存区域中页面的访问权限。访问权限标志定义在<asm/mman.h>中。

| 标志         | 权限         |
| ---------- | ---------- |
| PROT_READ  | 对应于VM_READ |
| PROT_WRITE | VM_WRITE   |
| PROT_EXEC  | VM_EXEC    |
| PROT_NONE  | 页不可被访问     |

flag指定了VMA标志，指定类型并改变映射的行为。

如果系统调用do_map()的参数中有无效参数，那么它返回一个负值；否则，它会在虚拟内存中分配一个合适的新内存区域。如果有可能的话，将新区域和邻近区域进行合并，否则内核从vm_area_cachep长字节slab缓存中分配一个vm_area_struct结构体，并使用vma_link()函数将新分配的内存区域添加到地址空间的内存区域链表和红-黑树中，随后还要更新内存描述符中的total_vm域，然后才返回新分配的地址区间的初始地址。

通过mmap()族函数系统调用获取内核函数do_mmap()的功能。

### mummap()和do_mummap()：删除地址区间

do_mummap()从特定的进程地址空间中删除指定地址区间，定义在<linux/mm.h>中。

```c
int do_munmap(struct mm_struct *, unsigned long, size_t);
```

第一个参数指向要删除区域所在的地址空间，删除从第二个参数开始，长度为第三个参数的地址区间。系统调用在<mm/mmap.c>中。sys_munmap()。

### 页表

应用程序操作的对象是映射到物理内存之上的虚拟内存，但处理器直接操作的却是物理内存。所以当程序访问一个虚拟地址时，首先必须将虚拟地址转化成物理地址，然后处理器才能解析地址访问请求。地址的转换工作需要通过查询页表才能完成，概括地讲，地址转换需要将虚拟地址分段，使每段虚拟地址都作为一个索引指向页表，而页表项则指向下一级别的页表或者指向最终的物理页面。

Linux中使用三级页表完成地址转换。利用多级页表能够节约地址转换需占用的存放空间。如果利用三级页表转换地址，即使是64位机器，占用的空间也很有限。但是如果使用静态数组实现页表，那么即便在32位机器上，该数组也将占用巨大的存放空间。Linux对所有体系结构，包括对那些不支持三级页表的体系结构(比如，有些体系结构只使用两级页表或者使用散列表完成地址转换)都使用三级页表管理，因为使用三级页表结构可以利用“最大公约数”的思想—一种设计简单的体系结构，可以按照需要在编译时简化使用页表的二级结构，比如只使用两级。

顶级页表是页全局目录（PGD, Page Global Directory），它包含了一个pgd_t类型数组，多数体系结构中pgd_t类型等同于无符号长整型类型。PGD中的表项指向二级页目录中的表项:PMD。

二级页表是中间页目录(PMD, Page Middle Directory)，它是个pmd_t类型数组，其中的表项指向PTE中的表项。

最后一级的页表简称页表（PTE, Page Table Entry），其中包含了pte_t类型的页表项，该页表项指向物理页面。

多数体系结构中，搜索页表的工作是由硬件完成的(至少某种程度上)。虽然通常操作中，很多使用页表的工作都可以由硬件执行，但是只有在内核正确设置页表的前提下，硬件才能方便地操作它们。![虚拟-物理地址查询.jpg](https://github.com/LiuChengqian90/Study-notes/blob/master/image/Linux/%E8%99%9A%E6%8B%9F-%E7%89%A9%E7%90%86%E5%9C%B0%E5%9D%80%E6%9F%A5%E8%AF%A2.jpg?raw=true)

每个进程都有自己的页表(当然，线程会共享页表)。内存描述符（mm_struct）的pgd域指向的就是进程的页全局目录。注意，操作和检索页表时必须使用page_table_lock锁，该锁在相应的进程的内存描述符中，以防止竟争条件。页表对应的结构体依赖于具体的体系结构，所以定义在<asm/page.h>中。

由于几乎每次对虚拟内存中的页面访间都必须先解析它，从而得到物理内存中的对应地址，所以页表操作的性能非常关键。但不幸的是，搜索内存中的物理地址速度很有限，因此为了加快搜素，多数体系结构都实现了一个翻译后缓冲器（Translate Lookaside Buffer, TLB）作为一个将虚拟地址映射到物理地址的硬件缓存。当请求访问一个虚拟地址时，处理器将首先检查TLB中是否缓存了该虚拟地址到物理地址的映射。

虽然硬件完成了有关页表的部分工作，但是页表的管理仍然是内核的关键部分——而且在不断改进。2.6版内核对页表管理的主要改进是：从高端内存分配部分页表。今后可能的改进包括：通过在写时拷贝(cogy-}n-write)的方式共亨页表。这种机制使得在fork()操作中可由父子进程共享页表。因为只有当子进程或父进程试图修改特定页表项时，内核才去创建该页表项的新拷贝，此后父子进程才不再共享该页表项。可以看到，利用共享页表可以消除fork()操作中页表拷贝所带来的消耗。

## 第16章 页高速缓存和页回写

## 第17章 设备和模块

## 第18章 调试

## 第19章 可移植性

## 第20章 补丁、开发和社区