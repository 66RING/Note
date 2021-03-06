---
title: QEMU的内存模拟
date: 2021-04-13
tags: 
- qemu
- softmmu
mathjax: true
---

# QEMU softmmu

虚拟机(Guest)内存模拟的首要任务是将虚拟机虚拟地址(GVA)转换成实际存储的宿主机(Host)的物理地址(HPA)。

虚拟机提供了虚拟的硬件环境，GVA到到虚拟机物理地址(GPA)的过程可以由Guest OS完成，而QEMU要做的就是提供一个GPA到HPA的转换的实现。

```
             GVA
              |
              | Guest OS
              |
              v
             GPA             
Guest         |
--------------|---------------
Host          V
             HVA
              |
              | Host OS
              |
              v
             HPA
```


## MemoryRegion

MemoryRegion基本信息及含义：

- MemoryRegion表示虚拟机的一段内存区域，即GPA。用于管理虚拟机的内存，是GPA与RAMBlock(管理HVA的结构)联系的桥梁
- 树状结构维护，**每个MemoryRegion树代表一类作用的内存**，如qemu中的两个全局MemoryRegion：系统内存空间(`system_memory`)或IO内存空间(`system_io`)
- 叶子节点表示实际分配给虚拟机的物理内存或者MMIO(即实体MemoryRegion)，中间节点表示内存总线，内存控制器是其他MemoryRegion的别名

MemoryRegion可以分为如下几类

- 根级MemoryRegion
    * 没有自己的内存，用于管理subregion，如`system_memory`
    * 通过`memory_region_init`初始化
    * **别名mr都是根级mr的subregion**
- **实体MemoryRegion**
    * **有具体的内存**，从QEMU进程地址空间分配内存
    * 通过`memory_region_init_ram`初始化，实际由`qemu_ram_alloc`分配实际内存。如RAM、ROM、ROM device等
    * 分配内存返回HVA，保存在RAMBlock成员的host域
    * MemoryRegion通过对应的RAMBlock管理HVA(RAMBlock的`host`域)
- 别名MemoryRegion
    * 表示实体mr的不同分段，没有自己的内存，是实体mr的一部分
    * 通过`memory_region_init_alias`初始化
    * 通过`alias`成员 **指向实体MemoryRegion** ，`alias_offset`表示该别名mr在实体内存中的偏移量

```c
struct MemoryRegion {
    ...
    bool ram;   // 是否是ram
    bool terminates;  // 是否是叶子节点
    bool enabled;   // 是否已经通知KVM使用这段内存
    ...
    RAMBlock *ram_block; //指向对应的RAMBlock
    ...
    const MemoryRegionOps *ops;  // 回调函数集合
    void *opaque;
    MemoryRegion *container; //指向父MR，主要用于将mr合并
    Int128 size; // 区域大小
    hwaddr addr; // 在父MR中的偏移量，若父类是`address_spaces_memory`可以理解为GPA
    ...
    MemoryRegion *alias; // 指向实体MR
    hwaddr alias_offset; // 起始地址(GPA)在实体MemoryRegion中的偏移量
    ...
    QTAILQ_HEAD(subregions, MemoryRegion) subregions; //子区域链表头
    QTAILQ_ENTRY(MemoryRegion) subregions_link; //子区域链表结点
    ...
};
```

- `addr`表示在父mr(即`container`的指向)中的偏移
- `alias_offset`表示在实体mr中的偏移
    * 注意：实体mr并不是别名mr的父mr或者说container，因为别名mr就是实体mr的一部分

虚拟机申请ram时一次性申请完成，然后再在该ram的基础上按照size划分出若干subregion。每个subregion又可以通过`alias`找到原始的mr，`alias_offset`记录其在原始mr中的偏移。


## RAMBlock

RAMBlock结构体用来关联GPA和HVA，其记录实际分配的内存地址信息，`host`域指向申请到的qemu进程地址(HVA)，以此作为虚拟机的物理地址。

通过offset+`ram_block->host`即可完成逻辑GPA到HVA的转换。

```c
struct RAMBlock {
    struct rcu_head rcu;    // 用于保护 Read-Copy-Update
    struct MemoryRegion *mr; // RAMBlock所在的MemoryRegion
    uint8_t *host;         // 对应的HVA
    ram_addr_t offset;     // 在ram_list地址空间中的偏移 (要把前面block的size都加起来)
    ram_addr_t used_length;  // 当前使用的长度
    ram_addr_t max_length;  // 总长度
    ...
    QLIST_ENTRY(RAMBlock) next;    // 指向下一个RAMBlock
    int fd;       // 映射文件的文件描述符
    size_t page_size; // page大小，一般和host保持一致
};
```

- RAMBlock中几个重要的域
    * `host`，表示虚拟机物理内存对应的QEMU进程地址空间的虚拟内存，即HVA
    * `offset`，表示在`ram_list`中的偏移

使用全局变量`ram_list`以链表形式维护所有RAMBlock，新分配的RAMBlock会被插入`ram_list`头部。要查找地址对应的RAMBlock则遍历`ram_list`链表。

通过`qemu_ram_alloc`创建一个新的RAMBlock，`qemu_ram_alloc`是对`qemu_ram_alloc_internal`的封装，其给`new_block`的一些成员赋值，然后用`ram_block_add`申请HVA赋给`host`域，并添加到全局`ram_list`中。

```c
RAMBlock *qemu_ram_alloc_internal(ram_addr_t size, ram_addr_t max_size,
                                  void (*resized)(const char*,
                                                  uint64_t length,
                                                  void *host),
                                  void *host, bool resizeable, bool share,
                                  MemoryRegion *mr, Error **errp)
{
    RAMBlock *new_block;
    Error *local_err = NULL;
    size = HOST_PAGE_ALIGN(size);
    max_size = HOST_PAGE_ALIGN(max_size);
    new_block = g_malloc0(sizeof(*new_block));
    new_block->mr = mr;
    new_block->resized = resized;
    new_block->used_length = size;
    new_block->max_length = max_size;
    assert(max_size >= size);
    new_block->fd = -1;
    new_block->page_size = qemu_real_host_page_size;
    new_block->host = host;                 // 此时host还是空的
    ...
    ram_block_add(new_block, &local_err, share);   // 这里再计算host应该是多少
    ...
    return new_block;
}
```

如果没启用xen，则`ram_block_add`通过调用`phys_mem_alloc()`分配实际的内存，其内部调用的是`mmap`，于是`host`域就得到了HVA值

```c
new_block->host = phys_mem_alloc(new_block->max_length,
                                 &new_block->mr->align, shared);
```

其他设备，如rom、flash等都会申请一个实体MemoryRegion，然后通过`ram_block_add`添加到全局的`ram_list`

通过如下命令启动虚拟机，则`ram_list`的内容是这样的

```
qemu-system-x86_64 --enable-kvm -m 1G -hda Resery.img -vnv :0 -smp4
```

![ram list](https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/universe/qemu/qemu_softmmu/ram_list.png)

总结一下大概关系：根级mr可找到所有别名mr，别名mr通过其`alias`域找到实体MemoryRegion。实体mr对应一个RAMBlock，可以找到其对应的HVA

![几种结构的关系](https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/universe/qemu/qemu_softmmu/mr_ramblock_subregion.png)


## AddressSpace

如果说RAMBlock维护了实际内存HVA。那么`AddressSpace`就维护了虚拟机的一片地址空间。

AddressSpace用来表示Guest侧CPU/设备视角的地址空间，不同设备使用的地址空间不同，内存地址空间`address_spaces_memory`和IO地址空间`address_spaces_io`。其`root`域指向根级MemoryRegion，从而可以找到一系列subregion

```
/**
 * AddressSpace: describes a mapping of addresses to #MemoryRegion objects
 */
struct AddressSpace {
    /* All fields are private. */
    struct rcu_head rcu;
    char *name;
    MemoryRegion *root; //指向根MR
    ...
    struct FlatView *current_map;    // 对应的FlatView
    ...
    struct MemoryRegionIoeventfd *ioeventfds;
    struct AddressSpaceDispatch *dispatch;   // 负责根据GPA找到HVA
    struct AddressSpaceDispatch *next_dispatch;
    MemoryListener dispatch_listener;
    QTAILQ_HEAD(memory_listeners_as, MemoryListener) listeners;
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;
};
```

AddressSpace还有个作用，就是是把MemoryRegion和FlatView联系起来，当mr发生变化时，对应的FlatView也应发生变化。可以通过`dispatch_listener`在mr发生变化时做的一系列更新。

`AddressSpaceDispatch`结构可以根据GPA找到HVA

```c
struct AddressSpaceDispatch {
    MemoryRegionSection *mru_section;
    /* This is a multi-level map on the physical address space.
     * The bottom level has pointers to MemoryRegionSections.
     */
    PhysPageEntry phys_map;  // 指向一级页表
    PhysPageMap map;
};

typedef struct PhysPageMap {
    struct rcu_head rcu;

    unsigned sections_nb;          // sections动态数组中有效元素个数
    unsigned sections_nb_alloc;    // sections数组的可用长度
    unsigned nodes_nb; 			   // nodes动态数组中有效元素的个数
    unsigned nodes_nb_alloc;       // nodes数组的可用长度
    Node *nodes;        // 指向Node数组，多级页表
    MemoryRegionSection *sections;   // 指向所有MemoryRegionSection，用于寻找GPA
} PhysPageMap;

struct PhysPageEntry {
    /* How many bits skip to next level (in units of L2_SIZE). 0 for a leaf. */
    uint32_t skip : 6;
     /* index into phys_sections (!skip) or phys_map_nodes (skip) */
    uint32_t ptr : 26;
};
```

- Node类型就是一个大小为512的PhysPageEntry数组
- 最后一级页表的index就是sections数组的索引，于是找到了MemoryRegionSection

> 如通过`iotlb_to_section()`可以在AddressSpace中根据addr找到对应section


## MemoryListener

当AddressSpace中的MemoryRegion发生变化，则触发注册的listener，处理region变更的事件


通过`listener_add_address_space()`将listener注册到其对应的AddressSpace上

```c
FOR_EACH_FLAT_RANGE(fr, view) {
    MemoryRegionSection section = section_from_flat_range(fr, view);

    if (listener->region_add) {
        listener->region_add(listener, &section);
    }
    if (fr->dirty_log_mask && listener->log_start) {
        listener->log_start(listener, &section, 0, fr->dirty_log_mask);
    }
}
if (listener->commit) {
    listener->commit(listener);
}
```


## FlatView

FlatView是MemoryRegion的平坦化表示，将树状的MemoryRegion展开成线性的FlatView，**是QEMU管理内存的另一种方式**。

因为要使用`KVM_SET_USER_MEMORY_REGION`向KVM注册，将HVA和GPA的对应关系注册到KVM模块才能生成EPT。


```c
/*
 * Note that signed integers are needed for negative offsetting in aliases
 * (large MemoryRegion::alias_offset).
 */
struct AddrRange {
    Int128 start; //起始
    Int128 size; //大小
};

/* Range of memory in the global map.  Addresses are absolute. */
struct FlatRange {
    MemoryRegion *mr; //指向所属的MR
    hwaddr offset_in_region; //在MR中的offset
    AddrRange addr; //本FR代表的区间
    uint8_t dirty_log_mask;
    bool romd_mode;
    bool readonly;
};

/* Flattened global view of current active memory hierarchy.  Kept in sorted
 * order.
 */
struct FlatView {
    struct rcu_head rcu;
    unsigned ref; //引用计数，为0就销毁
    FlatRange *ranges; //对应的flatrange数组
    unsigned nr; //flatrange数目
    unsigned nr_allocated;
};
```

FlatView的`range`域是一个FlatRange数组，每个FlatRange对应一段虚拟机物理地址区间，即一段GPA。可以通过AddrRange的start域拿到GPA的首地址


### 平坦化方法

通过调用`static FlatView *generate_memory_topology(MemoryRegion *mr)`生成一个根mr的FlatView，其中的核心函数是`render_memory_region()`用于填充FlatRange。

```c
static FlatView *generate_memory_topology(MemoryRegion *mr)
{
    FlatView *view;

    view = g_new(FlatView, 1);
    flatview_init(view);

    if (mr) {
        render_memory_region(view, mr, int128_zero(),
                             addrrange_make(int128_zero(), int128_2_64()), false);
        // 从根级region开始，递归将region映射到线性地址空间中，产生一个个FlatRange，构成 FlatView. addrrange_make创建起始地址为0，结束地址为 2^64 的地址空间，作为 guest 的线性地址空间
    }
    flatview_simplify(view);//将 FlatView 中连续的 FlatRange 进行合并为一个

    view->dispatch = address_space_dispatch_new(view);
    for (i = 0; i < view->nr; i++) {
        MemoryRegionSection mrs =
            section_from_flat_range(&view->ranges[i], view);
        flatview_add_to_dispatch(view, &mrs);
    }

    ...
    return view;
}
```

#### `render_memory_region()`

```c
render_memory_region(view, mr, int128_zero(),
                     addrrange_make(int128_zero(), int128_2_64()), false);
// 从根级 region 开始，递归将 region 映射到线性地址空间中，产生一个个 FlatRange，构成 FlatView. addrrange_make创建起始地址为 0，结束地址为 2^64 的地址空间，作为 guest 的线性地址空间
```

了解几个变量的含义就可以尝试读源码了：

- `clip`表示映射区间的范围
- `base`当前映射的小区间
- `remain`表示clip还没映射的大小
- `offset_in_region`需要映射的部分在其所属mr中的偏移
- `int128_sub()`128位减法，同理`ini128_xxx()`


## MemoryRegionSection

表示MemoryRegion中的片段。将MemoryRegion平坦化后，由于可能重叠，本来完整的mr可能就被分成了数片MemoryRegionSection。

```c
/**
 * MemoryRegionSection: describes a fragment of a #MemoryRegion
 *
 * @mr: the region, or %NULL if empty
 * @address_space: the address space the region is mapped in
 * @offset_within_region: the beginning of the section, relative to @mr's start
 * @size: the size of the section; will not exceed @mr's boundaries
 * @offset_within_address_space: the address of the first byte of the section
 *     relative to the region's address space
 * @readonly: writes to this section are ignored
 */
    struct MemoryRegionSection {
    MemoryRegion *mr;                           // 指向所属 MemoryRegion
    AddressSpace *address_space;                // 所属 AddressSpace
    hwaddr offset_within_region;                // Section在MemoryRegion内的偏移
    Int128 size;
    hwaddr offset_within_address_space;         // 在AddressSpace内的偏移量，如果该AddressSpace为系统内存，则为GPA起始地址
    bool readonly;
};
```

其中偏移`offset_within_region`描述的是该section在其所属的MR中的偏移，因为一个`MemoryRegion`可能有多个`MemoryRegionSection`构成。而`offset_within_address_space`是在整个地址空间中的偏移，是全局的offset，如果AddressSpace为系统内存，则该偏移则为GPA的起始地址

通过MemoryRegionSection，根据其在AddressSpace中的偏移`offset_within_address_space`，加上修正就得到了GPA

通过该section所属mr的RAMBlock得到HVA，再加上section在所属的mr中的偏移`offset_in_region`，再加上对齐修正，就得到了HVA

- **HVA** = `memory_region_get_ram_ptr(mr)` + `section->offset_within_region` + 页面对齐delta
	* 通过`memory_region_get_ram_ptr(mr)`得到当前Section对应的Mr的HVA起始地址。如果mr是alias则向上追溯，offset叠加，知道找到实体mr。`ram_block->host+offset`
- **GPA** = `offset_within_address_space` + 页对齐修正delta


# 实例说明

可以通过以下调用找到MemoreySection
```
address_space_translate_internal
	address_space_lookup_region
		phys_page_find(AddressSpaceDispatch *d, hwaddr addr) 
```

使用找到的MemoryRegionSection找到ram，从而完成GPA到HVA的转换

```
memory_region_get_ram_ptr(section->mr)
```


# Reference

- [[cnblog] qemu对虚拟机的内存管理（一）](https://www.cnblogs.com/ccxikka/p/9477530.html)
- [[cnblog] qemu对虚拟机的内存管理（二）](https://www.cnblogs.com/ccxikka/p/9488357.html)
- [QEMU内存](https://www.binss.me/blog/qemu-note-of-memory/)
- [【系列分享】QEMU内存虚拟化源码分析](https://www.anquanke.com/post/id/86412)












