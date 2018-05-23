---
title: DPDK 内存管理(一)
date: 2018-05-23 21:40:13
tags:
    - Hugepage
categories:
    - DPDK
---

DPDK 使用巨页来减少 TLB 不命中的问题。环境初始化时，DPDK 会将系统预留的 hugepage 全部申请，并且进行统一管理。在一个 DPDK instance 的生命周期中使用的应当都是 hugepage 提供的内存。下面是 DPDK 如何使用巨页，并进行管理的一个介绍。

APP 调用 `rte_eal_init()` 去初始化一个 DPDK instance 环境， 其中包含了内存、中断、PCI、时钟和运行模式的初始化。该函数的入口参数是启动 DPDK 的命令行，具体参数设置可以查看相关文档，或者直接查看参数解析代码。
```
int rte_eal_init(int argc, char **argv);   //初始化 EAL
int eal_parse_args(int argc, char **argv); //解析入口参数
```
<!-- more -->
`eal_hugepage_info_init()` 通过读取 `/sys/kernel/mm/hugepages` 下的巨页信息，将可用的巨页信息保存在全局变量 `internal_config` 中。有一点需要注意的是, 由于系统初始化时还没有得到 `NUMA` 架构的 `socket` 信息，所以这些信息都默认放在 `socket0`, 也就是 `hugepage_info.num_pages[0]`。
```
#define MAX_HUGEPAGE_SIZES 3  /**< support up to 3 page sizes */

struct hugepage_info {
	uint64_t hugepage_sz;   /**< size of a huge page */
	const char *hugedir;    /**< dir where hugetlbfs is mounted */
	uint32_t num_pages[RTE_MAX_NUMA_NODES]; /**< number of hugepages of that size on each socket */
	int lock_descriptor;    /**< file descriptor for hugepage dir */
};

struct internal_config {
    ...
	unsigned num_hugepage_sizes;      /**< how many sizes on this system, <= MAX_HUGEPAGE_SIZES*/
	struct hugepage_info hugepage_info[MAX_HUGEPAGE_SIZES];
};
```

现在巨页的信息已经有了，下面就是想办法申请这些巨页并映射到进程的虚拟地址空间，这样应用才可以使用。这部分操作是在 `rte_eal_memory_init()` 中实现的。这里要区分主进程和从进程，主进程负责把巨页申请和映射，从进程只需要读取主进程的映射信息，并且将巨页映射到相同的虚拟地址即可。这样就可以保证**在使用一个 DPDK instance 的所有进程中，同样的虚拟地址一定指向相同的物理内存。**
```
const int retval = rte_eal_process_type() == RTE_PROC_PRIMARY ?
			rte_eal_hugepage_init() :
			rte_eal_hugepage_attach();
```

#### PRIMARY PROC
先从堆上分配若干内存用来零时存放巨页的具体信息，其大小和巨页数量和大小有关，为 `nr_hugepages * sizeof(struct hugepage_file)`，该零时内存用数组`tmp_hp[*]` 管理。
```
struct hugepage_file {
	void *orig_va;      /**< virtual addr of first mmap() */
	void *final_va;     /**< virtual addr of 2nd mmap() */
	uint64_t physaddr;  /**< physical addr */
	size_t size;        /**< the page size */
	int socket_id;      /**< NUMA socket ID */
	int file_id;        /**< the '%d' in HUGEFILE_FMT */
	int memseg_id;      /**< the memory segment to which page belongs */
	char filepath[MAX_HUGEPAGE_PATH]; /**< path to backing file on filesystem */
};
```

然后对 `internal_config` 中保存的巨页信息按照大小进行遍历，通过**两次映射**，将同一种大小的巨页尽可能地映射到一段连续的虚拟内存，称为一个内存段 `memseg`。如果没有这么大连续虚拟地址空间，那么将不连续的虚拟地址段划分到另外的内存段做管理。不同的大小的巨页，总是处在不同的内存段。

第一次映射，简单地将所有巨页映射，虚拟地址由系统自动分配。第一次映射之后，通过 `/proc/self/pagemap` 文件通过虚拟地址找到巨页实际的物理地址，保存到 `tmp_hp` 中。同时从 `/proc/self/numa_maps` 读取 NUMA 节点的 socket 信息，也保存到`tmp_hp` 中。接着根据前面找到的物理地址，对 `tmp_hp` 数组进行排序。

在第二次映射时，将排序好的巨页依次映射到一大块空闲的虚拟地址中，如果期望的虚拟地址映射失败，则交由系统随机分配一个新的地址。一般情况下，DPDK 的初始化都放在进程的开始阶段，所以一般来说，我们总能找到大块空闲的虚拟地址给 DPDK。映射完成后，我们在主进程中就得到了一块或多块虚拟地址连续，且物理地址连续的内存段。然后只需要将第一次映射的虚拟地址解除映射即可。

映射结束，创建一块共享内存用来保存 `tmp_hp` 的巨页信息，以便从进程使用。最后根据巨页的物理地址和虚拟地址信息, 以及 NUMA 节点信息，划分并配置好内存段，保存到全局变量 `rte_config->mem_config.memseg[i]` 中。

```
map_all_hugepages(..., 1);

find_physaddrs(...);

find_numasocket(...);

qsort(&tmp_hp[hp_offset], hpi->num_pages[0], sizeof(struct hugepage_file), cmp_physaddr);

map_all_hugepages(..., 0);

unmap_all_hugepages_orig(...);

//对于 NUMA 架构来说，还需要算出每个 socket 合适的不同大小的巨页数量。
calc_num_pages_per_socket(...);

copy_hugepages_to_shared_mem(...);

//配置内存段（memseg[0], memseg[1], ...）
...
```
![](maphugepage.png)

#### SECONDARY PROC
映射主进程中保存 `tmp_hp` 信息的共享内存，根据巨页的具体信息和 `rte_config->mem_config.memseg[i]` 中的信息，将巨页挂在到相同的虚拟地址。

**注意以下几种情况下的巨页将不能放在一个内存段（memseg）管理:**
* socket id 不同；
* 巨页大小不同；
* 虚拟地址不连续；
* 物理地址不连续；

#### 重要的结构体
```
struct rte_mem_config {
	volatile uint32_t magic;   /**< Magic number - Sanity check. */

	/* memory topology */
	uint32_t nchannel;    /**< Number of channels (0 if unknown). */
	uint32_t nrank;       /**< Number of ranks (0 if unknown). */

	/**
	 * current lock nest order
	 *  - qlock->mlock (ring/hash/lpm)
	 *  - mplock->qlock->mlock (mempool)
	 * Notice:
	 *  *ALWAYS* obtain qlock first if having to obtain both qlock and mlock
	 */
	rte_rwlock_t mlock;   /**< only used by memzone LIB for thread-safe. */
	rte_rwlock_t qlock;   /**< used for tailq operation for thread safe. */
	rte_rwlock_t mplock;  /**< only used by mempool LIB for thread-safe. */

	uint32_t memzone_cnt; /**< Number of allocated memzones */

	/* memory segments and zones */
	struct rte_memseg memseg[RTE_MAX_MEMSEG];    /**< Physmem descriptors. */
	struct rte_memzone memzone[RTE_MAX_MEMZONE]; /**< Memzone descriptors. */

	struct rte_tailq_head tailq_head[RTE_MAX_TAILQ]; /**< Tailqs for objects */

	/* Heaps of Malloc per socket */
	struct malloc_heap malloc_heaps[RTE_MAX_NUMA_NODES];

	/* address of mem_config in primary process. used to map shared config into
	 * exact same address the primary process maps it.
	 */
	uint64_t mem_cfg_addr;
} __attribute__((__packed__));

struct rte_config {
	uint32_t master_lcore;       /**< Id of the master lcore */
	uint32_t lcore_count;        /**< Number of available logical cores. */
	enum rte_lcore_role_t lcore_role[RTE_MAX_LCORE]; /**< State of cores. */
	enum rte_proc_type_t process_type; /** Primary or secondary configuration */

	/**
	 * Pointer to memory configuration, which may be shared across multiple
	 * DPDK instances
	 */
	struct rte_mem_config *mem_config;
} __attribute__((__packed__));
```