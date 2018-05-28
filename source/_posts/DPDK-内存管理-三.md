---
title: DPDK 内存管理(三)
date: 2018-05-28 22:17:55
tag:
categories:
    - DPDK
---
前面讲述了 DPDK 堆的结构和使用，但是我们知道 DPDK 是高性能的数据面开发组件，如果所有的消息都从通过 DPDK 堆中申请，每次申请和释放都需要锁保护，可想而知，性能肯定不高。所以，DPDK 提供了内存池（rte_mempool）和环形缓冲队列（rte_ring）来实现高效内部消息转发。
<!-- more -->

## 内存区 (memzone)

在介绍内存池之前，先介绍一下内存区（ rte_memzone ）的概念。内存区是 DPDK 内存管理最终向客户程序提供的基础接口，通过 `rte_memzone_reverse` 族接口可以获取基于巨页的内存块。其实其底层同样使用 `rte_malloc` 族接口，从内存堆中获取内存。不同的是，`rte_memzone_reverse` 族接口还会将内存块的信息保存到全局变量 `rte_config->mem_config.memzone[]` 中。
```
struct rte_memzone {
#define RTE_MEMZONE_NAMESIZE 32       /**< Maximum length of memory zone name.*/
	char name[RTE_MEMZONE_NAMESIZE];  /**< Name of the memory zone. */
	phys_addr_t phys_addr;            /**< Start physical address. */
	RTE_STD_C11
	union {
		void *addr;                   /**< Start virtual address. */
		uint64_t addr_64;             /**< Makes sure addr is always 64-bits */
	};
	size_t len;                       /**< Length of the memzone. */
	uint64_t hugepage_sz;             /**< The page size of underlying memory */
	int32_t socket_id;                /**< NUMA socket ID. */
	uint32_t flags;                   /**< Characteristics of this memzone. */
	uint32_t memseg_id;               /**< Memseg it belongs. */
} __attribute__((__packed__));
```

## 内存池 (mempool)

内存池是 DPDK 提供给用户的高效内存管理系统，一般用于存储用户面报文数据。DPDK 提供了 `rte_pktmbuf_pool_create` 接口方便应用创建内存池。内存池的创建过程大致分以下几步。

#### 1. 申请链表节点
首先从 DPDK 堆中分配一个 `rte_tailq_entry` 备用, 这个节点的数据指针会在后面指向 `rte_mempool` 的结构体，这样我们便可以通过全局的链表找到需要的内存池。
```
rte_zmalloc("MEMPOOL_TAILQ_ENTRY", sizeof(*te), 0);
```

`rte_tailq_entry` 的定义如下，直接采用了 Linux 提供的 `<queue.h>` 中的链表结构和处理函数。
```
//defined in linux <queue.h>
#define	_TAILQ_ENTRY(type, qual)					\
struct {								\
	qual type *tqe_next;		/* next element */		\
	qual type *qual *tqe_prev;	/* address of previous next element */\
}
#define TAILQ_ENTRY(type)	_TAILQ_ENTRY(struct type,)

struct rte_tailq_entry {
	TAILQ_ENTRY(rte_tailq_entry) next; /**< Pointer entries for a tailq list */
	void *data; /**< Pointer to the data referenced by this tailq entry */
};
```

#### 2. 申请一个空的内存池（申请一块管理内存区）
接着，申请一块管理内存区，这块内存区的大小为 `sizeof(rte_mempool) + RTE_MAX_LCORE*sizeof(rte_mempool_cache) + sizeof(rte_pktmbuf_pool_private)` . 其主要用来存放管理内存池的数据结构 `rte_mempool` ，以及为每个物理核实现的软件 cache 。
```
//此时可以认为我们申请了一块空的内存池。
rte_memzone_reserve(..., mempool_size, ...);
```

这块管理内存区的结构如下所示。内存池的管理结构中 `pool_id` 指向的是管理该内存池的 DPDK ring, `size` 大小为该内存池内存块的数量，`cache_size` 则指定了每个核的 `rte_mempool_cache` 中可以保存的内存块的最大数量，`elt_list` 下挂着由该内存池下所有内存块串成的单向链表。

![](rte_mempool.png)

`rte_mempool_cache` 用于每个核上内存块的软件缓存，用户不管是申请内存块还是释放内存块，总是优先考虑本核的 `local_cache` 。可以简单的理解，最近访问的内存总是大概率的缓存在物理 cache 中，所以我们从软件 cache 中拿到的地址 cache 命中的概率总是比从 ring 中拿出来的地址要高。

![](localcache.png)

#### 3. 将节点插入全局链表
将前面申请的 `rte_tailq_entry` 的数据指针指向刚申请的内存池数据结构，并将该节点插入到全局双向链表 `rte_mempool_tailq` 中做统一管理。
```
TAILQ_INSERT_TAIL(mempool_list, te, next);
```

![](mempoollist.png)

#### 4. 申请真正给用户使用的内存池空间
接下来需要申请真正用来存放用户数据的内存池。这是一块单独的内存区，但是会将相关详细保存到前面的 `rte_mempool` 结构体中，这样通过 `rte_mempool_tailq` , `rte_mempool` 便可以找到这块内存。这里还会根据该内存池的 block 数量创建出一个长度为 `blocknum + 1` 的 DPDK ring 。同时，，将这块内存池根据 block 数量进行切割，每个 block 前面填入 `rte_mempool_objhdr` 并串成单向链表，即 `rte_mempool.elt_list` 。
```
rte_mempool_populate_default(mp)
-->rte_mempool_populate_phys(...);
	-->rte_mempool_ops_alloc(...); //alloc dpdk ring
```

#### 5. 内存池初始化
用 `rte_pktmbuf_init` 函数对每块内存块进行初始化，其中 `H` 和 `T` 是内存块的头和尾，在第四步已经填好。`MH` 即 `mbuf header`；`P` 为 `rte_pktmbuf_pool_private` 结构体；`HR` 即 `header room`，一般为用户消息保留的管理头大小；`subpool[i].size` 即是用户申请时期望的每块内存块的大小。
```
rte_mempool_obj_iter(mp, rte_pktmbuf_init, NULL);
```
![](mempool.png)
