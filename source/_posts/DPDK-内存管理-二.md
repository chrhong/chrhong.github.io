---
title: DPDK 内存管理(二)
date: 2018-05-23 21:47:13
tag:
categories:
    - DPDK
---
申请到了巨页，也划分好了内存段，那该如何使用呢？这节详细介绍 DPDK 的内存堆管理。

DPDK 的堆就是一种管理巨页的方式，而 DPDK 中用到的绝大多数内存也通过特殊的 `rte_malloc` 类接口从自己的堆中申请。DPDK 将根据 NUMA socket 来进行内存管理，在 `rte_eal_malloc_heap_init()` 中，我们可以看到，通过遍历内存段，我们将同属于一个 socket 的内存段划分到一个内存堆来管理。堆信息将保存在全局变量 `rte_config->mem_config.malloc_heaps[i]` 中。<!-- more -->
```
int rte_eal_malloc_heap_init(void)
{
    ...
    for (ms = &mcfg->memseg[0], ms_cnt = 0;
			(ms_cnt < RTE_MAX_MEMSEG) && (ms->len > 0);
			ms_cnt++, ms++) {
		malloc_heap_add_memseg(&mcfg->malloc_heaps[ms->socket_id], ms);
	}

    return 0;
}
```

### DPDK 堆内存初始化

初始化时，一个内存段（ memseg ）被视为一大块空闲的内存，称为一个分配单元（malloc_elem），每个分配单元需要进行 Cache Line 对其。如下所示，先获得这个分配单元的起始地址，然后减去管理的头（ sizeof(malloc_elem) ）和尾（ RTE_CACHE_LINE_SIZE , 只在开了 DEBUG 的情况下生效，一般为 0），并对获得的结束地址进行对其操作得到最终的结束地址和该单元有效内存大小（包含了头和尾）。
```
struct malloc_elem *start_elem = (struct malloc_elem *)ms->addr;
struct malloc_elem *end_elem = RTE_PTR_ADD(ms->addr,
        ms->len - MALLOC_ELEM_OVERHEAD);
end_elem = RTE_PTR_ALIGN_FLOOR(end_elem, RTE_CACHE_LINE_SIZE);
const size_t elem_size = (uintptr_t)end_elem - (uintptr_t)start_elem;
```

对有效的分配单元和尾部无效部分进行初始化。这块尾部的也被认为是一个内存单元，但是从一开始就会被标记为 `ELEM_BUSY` 状态，防止被使用。
```
malloc_elem_init(start_elem, heap, ms, elem_size);
malloc_elem_mkend(end_elem, start_elem);
```

最终得到的结果如下图所示：
![](memseg.png)

有效的内存单元将被插入到对应的 NUMA 节点的堆管理链表中 `rte_config->mem_config.malloc_heaps[socketid]`，并增加相应的堆大小。
```
malloc_elem_free_list_insert(start_elem);
heap->total_size += elem_size;
```

DPDK 的堆内存管理采用一种类似哈希表的方式，使用数组下挂链表的方式。首先以具体分配单元的大小来确定数组索引 `free_head[index]`，然后用双向链表将同一个范围的分配单元串联起来。

![](heaptable.png)

```
//堆内存的管理头
struct malloc_heap {
	rte_spinlock_t lock;
	LIST_HEAD(, malloc_elem) free_head[RTE_HEAP_NUM_FREELISTS];
	unsigned alloc_count;
	size_t total_size;
} __rte_cache_aligned;
```

需要注意的是，每个内存单元（malloc_elem）有个 `prev` 指针将所有的内存单元串成单向链表。可以简单的理解为 `prev` 串联所有内存单元，而 `free_list` 串联所有空闲的内存单元。

![](insertheap.png)

由上可知，堆内存刚初始化时，一个内存段只有一块空闲内存和一块不可用内存，但是一个堆可能有多个内存段。

### DPDK 堆内存分配
申请堆内存的基础接口是 `malloc_elem_alloc`，提供从一个内存单元中分配指定大小的内存。所以应用申请时需要先知道具体到哪个堆或者哪个 NUMA 节点上申请，而且还得找到具体的满足要求的内存单元，而上层应用其实并不关心这些，所以 DPDK 封装了更外层的接口提供给应用使用。
```
/* Please refer lib/librte_eal/common/rte_malloc.c */

rte_realloc
    |
    +-->rte_malloc
            |
            +-->rte_malloc_socket(SOCKET_ID_ANY)
                      ^    |
rte_calloc            |    +-------------->malloc_heap_alloc---->malloc_elem_alloc
    |                 |
    +-->ret_zmalloc   |
            |         +
            +-->rte_zmalloc_socket(SOCKET_ID_ANY)
```

其中 `malloc_heap_alloc` 接口会根据用户申请的内存大小，得到 `free_head[]` 的索引 `i`，然后遍历挂在 `free_head[i]` 下的链表，找到一个可以满足用户需求的内存单元。
```
rte_spinlock_lock(&heap->lock);
elem = find_suitable_element(heap, size, flags, align, bound);
if (elem != NULL) {
	elem = malloc_elem_alloc(elem, size, align, bound);
	/* increase heap's count of allocated elements */
	heap->alloc_count++;
}
rte_spinlock_unlock(&heap->lock);
```

`malloc_elem_alloc` 则将对具体的内存单元进行分割，以将合适大小的内存分给用户。以第一次分配为例，根据前面的知识可知，第一个内存单元将是与一个内存段相当的一块内存，而显然这将超出用户申请的大小。此时，DPDK 会对该内存单元进行分割，分成两块较小的内存单元，一块刚好满足用户需求，分配给用户，并标记为 `ELEM_BUSY` 状态。另一块作为空闲内存单元初始化，如果该内存单元有效，将重新根据其大小，插入到新的 `free_head[j]` 下挂在的链表中。

![](heapalloc.png)

>注意：一块有效的内存单元需要 **大于或等于** 以下最小内存单元大小，所有小于最小内存单元的单元都将被标记为 `ELEM_BUSY` 状态。

![](minelem.png)

### DPDK 堆内存释放
堆内存的释放比较简单，通过指针偏移，获得内存单元的指针传给 `malloc_elem_free` 即可。
```
void rte_free(void *addr)
{
	if (addr == NULL) return;
	if (malloc_elem_free(malloc_elem_from_data(addr)) < 0)
		rte_panic("Fatal error: Invalid memory\n");
}
```

堆内存释放的过程和申请时相反，涉及到内存块的合并。
* 如果待释放内存单元（ELEM_A）前一个和后一个内存单元都是 `ELEM_BUSY` 状态, 则只根据 ELEM_A 大小插入到对应的数组下挂在的链表；
* 如果 ELEM_A 前一个内存单元（ELEM_P）是 `ELEM_FREE` 状态， 则将 ELEM_A 和 ELEM_P 合并成一个大的内存单元，插入新的链表；
* 如果 ELEM_A 后一个内存单元（ELEM_N）是 `ELEM_FREE` 状态， 则将 ELEM_A 和 ELEM_N 合并成一个大的内存单元，插入新的链表；
* 如果 ELEM_A 前后两个内存单元都是 `ELEM_FREE` 状态， 则将 ELEM_A ，ELEM_P 和 ELEM_N 合并成一个大的内存单元，插入新的链表；

![](heapfree.png)
