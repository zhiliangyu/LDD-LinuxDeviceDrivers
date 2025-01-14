2   **内存管理子系统(memory management)**
=====================


# 0 内存管理子系统(开篇)
-------



## 0.1 内存管理子系统概述
------

**概述: **内存管理子系统**, 作为 kernel 核心中的核心, 是承接所有系统活动的舞台, 也是 Linux kernel 中最为庞杂的子系统, 没有之一．截止 4.2 版本, 内存管理子系统(下简称 MM)所有平台独立的核心代码(C文件和头文件)达到11万6千多行, 这还不包括平台相关的 C 代码, 及一些汇编代码; 与之相比, 调度子系统的平台独立的核心代码才2万８千多行．


现代操作系统的 MM 提供的一个重要功能就是为每个进程提供独立的虚拟地址空间抽象, 为用户呈现一个平坦的进程地址空间, 提供安全高效的进程隔离, 隐藏所有细节, 使得用户可以简单可移植的库接口访问／管理内存, 大大解放程序员生产力．



在继续往下之前, 先介绍一些 Linux 内核中内存管理的基本原理和术语, 方便后文讨论．





**- 物理地址(Physical Address):** 这就是内存 DIMM 上的一个一个存储区间的物理编址, 以字节为单位．



**- 虚拟地址(Virtual Address):** 技术上来讲, 用户或内核用到的地址就是虚拟地址, 需要 MMU (内存管理单元, 一个用于支持虚拟内存的 CPU 片内机构) 翻译为物理地址．在 CPU 的技术规范中, 可能还有虚拟地址和线性地址的区别, 但在这不重要．



**- NUMA(Non-Uniform Memory Access):** 非一致性内存访问．NUMA 概念的引入是为了解决随着 CPU 个数的增长而出现的内存访问瓶颈问题, 非一致性内存意为每个 NUMA 节点都有本地内存, 提供高访问速度; 也可以访问跨节点的内存, 但要遭受较大的性能损耗．所以尽管整个系统的内存对任何进程来说都是可见的, 但却存在访问速度差异, 这一点对内存分配/内存回收都有着非常大的影响．Linux 内核于2.5版本引入对 NUMA的支持[<sup>7</sup>](#refer-anchor-7).



**- NUMA node(NUMA节点):**  NUMA 体系下, 一个 node 一般是一个CPU socket(一个 socket 里可能有多个核)及它可访问的本地内存的整体．



**- zone(内存区):** 一个 NUMA node 里的物理内存又被分为几个内存区(zone), 一个典型的 node 的内存区划分如下:

![zone 区域](https://pic3.zhimg.com/50/b53313b9ef1f062460f90f56bcf6d0b7_hd.jpg)


可以看到每个node里, 随着**物理内存地址**的增加, 典型地分为三个区:

> **1\. ZONE\_DMA**: 这个区的存在有历史原因, 古老的 ISA 总线外设, 它们进行 DMA操作[<sup>10</sup>](#refer-anchor-10) 时, 只能访问内存物理空间低 16MB 的范围．所以故有这一区, 用于给这些设备分配内存时使用．
> **2\. ZONE\_NORMAL**: 这是 32位 CPU时代产物, 很多内核态的内存分配都是在这个区间(用户态内存也可以在这部分分配, 但优先在ZONE\_HIGH中分配), 但这部分的大小一般就只有 896 MiB, 所以比较局限． 64位 CPU 情况下, 内存的访问空间增大, 这部分空间就增大了很多．关于为何这部分区间这么局限, 且内核态内存分配在这个区间, 感兴趣的可以看我之间一个回答[<sup>11</sup>](#refer-anchor-11).
> **3\. ZONE\_HIGH**: 典型情况下, 这个区间覆盖系统所有剩余物理内存．这个区间叫做高端内存区(不是高级的意思, 是地址区间高的意思). 这部分主要是用户态和部分内核态内存分配所处的区间．





**- 内存页/页面(page):** 现代虚拟内存管理／分配的单位是一个物理内存页, 大小是 4096(4KB) 字节. 当然, 很多 CPU 提供多种尺寸的物理内存页支持(如 X86, 除了4KB, 还有 2MB, 1GB页支持), 但 Linux 内核中的默认页尺寸就是 4KB．内核初始化过程中, 会对每个物理内存页分配一个描述符(struct page), 后文描述中可能多次提到这个描述符, 它是 MM 内部, 也是 MM 与其他子系统交互的一个接口描述符．



**- 页表(page table):** 从一个虚拟地址翻译为物理地址时, 其实就是从一个稀疏哈希表中查找的过程, 这个哈希表就是页表．



**- 交换(swap):** 内存紧缺时, MM 可能会把一些暂时不用的内存页转移到访问速度较慢的次级存储设备中(如磁盘, SSD), 以腾出空间,这个操作叫交换, 相应的存储设备叫交换设备或交换空间.



**- 文件缓存页(PageCache Page):** 内核会利用空闲的内存, 事先读入一些文件页, 以期不久的将来会用到, 从而避免在要使用时再去启动缓慢的外设(如磁盘)读入操作. 这些有后备存储介质的页面, 可以在内存紧缺时轻松丢弃, 等到需要时再次从外设读入. **典型的代表有可执行代码, 文件系统里的文件.**

**- 匿名页(Anonymous Page):** 这种页面的内容都是在内存中建立的,没有后备的外设, 这些页面在回收时不能简单的丢弃, 需要写入到交换设备中. **典型的代表有进程的栈, 使用 _malloc()_ 分配的内存所在的页等 .**



## 0.2 重要功能和时间点
-------


内存管理的目标是提供一种方法, 为实现各种目的而在各个用户之间实现内存共享. 内存管理方法应该实现以下两个功能:

*   最小化管理内存所需的时间

*   最大化用于一般应用的可用内存(最小化管理开销)

内存管理实际上是一种关于权衡的零和游戏. 您可以开发一种使用少量内存进行管理的算法, 但是要花费更多时间来管理可用内存. 也可以开发一个算法来有效地管理内存, 但却要使用更多的内存. 最终, 特定应用程序的需求将促使对这种权衡作出选择.


| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----------:|:---:|
| 1991/01/01 | [示例 sched: Improve the scheduler]() | 此处填写描述【示例】 | ☑ ☒☐ v3/5.4-rc1 | [LWN](), [PatchWork](), [lkml]() |
| 2016/06/27 | [mm: add page cache limit and reclaim feature](http://lore.kernel.org/patchwork/patch/473535) | Page Cache Limit | RFC v2 ☐  | [PatchWork](https://lore.kernel.org/patchwork/patch/473535) |
| 2016/06/27 | [mm: mirrored memory support for page buddy allocations](http://lore.kernel.org/patchwork/patch/574230) | 内存镜像的功能 | RFC v2 ☐  | [PatchWork](https://lore.kernel.org/patchwork/patch/574230) |
| 2018/05/17 | [Speculative page faults](http://lore.kernel.org/patchwork/patch/906210) | SPF | v11 ☐  | [PatchWork](https://lore.kernel.org/patchwork/patch/906210) |
| 2018/07/09 | [Improve shrink_slab() scalability (old complexity was O(n^2), new is O(n))](http://lore.kernel.org/patchwork/patch/960597) | 内存镜像的功能 | RFC v2 ☐  | [PatchWork](https://lore.kernel.org/patchwork/patch/960597) |
| 2020/02/24 | [Fine grained MM locking](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) | MM lockless | RFC ☐ | [PatchWork](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com), [fine_grained_mm.pdf](https://linuxplumbersconf.org/event/4/contributions/556/attachments/304/509/fine_grained_mm.pdf) |


## 0.3 主线内存管理分支合并窗口
-------

Mainline Merge Window - Merge branch 'akpm' (patches from Andrew)

| 版本 | 发布时间 | 合并链接 |
|:---:|:-------:|:-------:|
| 5.13 | 2021/06/28 | [5.13-rc1 step 1 2021/04/30](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d42f323a7df0b298c07313db00b44b78555ca8e6)<br>[5.13-rc1 step 2 2021/05/05](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8404c9fbc84b741f66cff7d4934a25dd2c344452)<br>[5.13-rc2 2021/05/15](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a4147415bdf152748416e391dd5d6958ad0a96da)<br>[5.13-rc3 2021/05/22](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=34c5c89890d6295621b6f09b18e7ead9046634bc)<br>[5.13-rc5 2021/06/05](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e5220dd16778fe21d234a64e36cf50b54110025f)<br>[5.13-rc7 2021/06/16](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=70585216fe7730d9fb5453d3e2804e149d0fe201)<br>[5.13 2021/06/25](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7ce32ac6fb2fc73584b567c73ae0c47528954ec6) |
| 5.14 | NA | [5.14 2021/06/29](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=65090f30ab791810a3dc840317e57df05018559c) |



## 0.4 社区的内存管理领域的开发者
-------

| developer | git |
|:---------:|:---:|
| [David Rientjes <rientjes@google.com>](https://lore.kernel.org/patchwork/project/lkml/list/?submitter=6580&state=*&archive=both&param=4&page=1) | NA |
| [Mel Gorman <mgorman@techsingularity.net>](https://lore.kernel.org/patchwork/project/lkml/list/?submitter=19167&state=*&archive=both&param=3&page=1) | [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/mel/linux.git) |
| [Minchan Kim <minchan@kernel.org>](https://lore.kernel.org/patchwork/project/lkml/list/?series=&submitter=13305&state=*&q=&archive=both&delegate=) | NA |
| [Joonsoo Kim](https://lore.kernel.org/patchwork/project/lkml/list/?submitter=13703&state=%2A&archive=both) | NA |
| [Kamezawa Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com>](https://lore.kernel.org/patchwork/project/lkml/list/?submitter=4430&state=%2A&archive=both) | NA |


## 0.5 社区地址
-------


| 描述 | 地址 |
|:---:|:----:|
| WebSite | [linux-mm](https://www.linux-mm.org) |
| PatchWork | [PatchWork](https://patchwork.kernel.org/project/linux-mm/list/?archive=both&state=*) |
| 邮件列表 | linux-mm@kvack.org |
| maintainer branch | [Github](https://github.com/hnaz/linux-mm)

## 0.5 目录
-------

下文将按此**目录**分析 Linux 内核中 MM 的重要功能和引入版本:




- [x] 1. 页表管理

- [x] 2. 内存分配

- [x] 3. 内存去碎片化

- [x] 4. 页面回收

- [x] 5. 页面写回

- [x] 6. 页面预读

- [x] 7. 大内存页支持

- [x] 8. 内存控制组(Memory Cgroup)支持

- [x] 9. 内存热插拔支持

- [x] 10. 超然内存(Transcendent Memory)支持

- [x] 11. 非易失性内存 (NVDIMM, Non-Volatile DIMM) 支持

- [x] 12. 内存管理调试支持

- [x] 13. 杂项



# 1 页表管理
-------


## 1.1 多级页表
-------

**2.6.11(2005年3月发布)**

页表实质上是一个虚拟地址到物理地址的映射表, 但由于程序的局部性, 某时刻程序的某一部分才需要被映射, 换句话说, 这个映射表是相当稀疏的, 因此在内存中维护一个一维的映射表太浪费空间, 也不现实. 因此, 硬件层次支持的页表就是一个多层次的映射表.

Linux 一开始是在一台i386上的机器开发的, i386 的硬件页表是2级的(页目录项 -> 页表项), 所以, 一开始 Linux 支持的软件页表也是2级的; 后来, 为了支持 PAE (Physical Address Extension), 扩展为3级; 后来, 64位 CPU 的引入, 3级也不够了, 于是, 2.6.11 引入了四级的通用页表.


关于四级页表是如何适配 i386 的两级页表的, 很简单, 就是虚设两级页表. 类比下, 北京市(省)北京市海淀区东升镇, 就是为了适配4级行政区规划而引入的一种表示法. 这种通用抽象的软件工程做法在内核中不乏例子.

关于四级页表演进的细节, 可看我以前文章: [Linux内核4级页表的演进](https://link.zhihu.com/?target=http%3A//larmbr.com/2014/01/19/the-evolution-of-4-level-page-talbe-in-linux)

https://lwn.net/Articles/717293/
generic code to 5-level paging
https://lore.kernel.org/patchwork/project/lkml/list/?submitter=13419&state=*&archive=both&param=6&page=7

## 1.2 延迟页表缓存冲刷 (Lazy-TLB flushing)
-------

**极早引入, 时间难考**


有个硬件机构叫 TLB, 用来缓存页表查寻结果, 根据程序局部性, 即将访问的数据或代码很可能与刚访问过的在一个页面, 有了 TLB 缓存, 页表查找很多时候就大大加快了. 但是, 内核在切换进程时, 需要切换页表, 同时 TLB 缓存也失效了, 需要冲刷掉. 内核引入的一个优化是, 当切换到内核线程时, 由于内核线程不使用用户态空间, 因此切换用户态的页表是不必要, 自然也不需要冲刷 TLB. 所以引入了 Lazy-TLB 模式, 以提高效率. 关于细节, 可参考[kernel 3.10内核源码分析--TLB相关--TLB概念、flush、TLB lazy模式](https://www.cnblogs.com/sky-heaven/p/5133747.html)


## 1.3 Clarifying memory management with page folios
-------


[LWN：利用page folio来明确内存操作！](https://blog.csdn.net/Linux_Everything/article/details/115388078)

[带有“memory folios”的 Linux：编译内核时性能提升了 7%](https://www.heikewan.com/item/27509944)

内存管理(memory management) 一般是以 page 为单位进行的, 一个 page 通常包含 4,096 个字节, 也可能更大. 内核已经将 page 的概念扩展到所谓的 compound page(复合页), 即一组组物理连续的单独 page 的组合. 这又使得 "page" 的定义变得有些模糊了. Matthew Wilcox 提出了 "page folio" 的概念, 它实际上仍然是一个 page structure, 只是保证了它一定不是 tail page. 任何接受 folio page 参数的函数都会是对整个 compound page 进行操作(如果传入的确实是一个 compound page 的话), 这样就不会有任何歧义. 从而可以使内核里的内存管理子系统更加清晰；也就是说, 如果某个函数被改为只接受 folio page 作为参数的话, 很明确, 它们不适用于对 tail page 的操作. 通过 folio 结构来管理内存. 它提供了一些具有自身价值的基础设施, 将内核的文本缩减了约 6kB.



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/06/22 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [Memory folios](https://lwn.net/Articles/849538) | NA | v13a ☐ | [PatchWork v13,000/137](https://patchwork.kernel.org/project/linux-mm/cover/20210712030701.4000097-1-willy@infradead.org/)<br>[PatchWork v13a](https://patchwork.kernel.org/project/linux-mm/cover/20210712190204.80979-1-willy@infradead.org) |
| 2021/06/22 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [Folio-enabling the page cache](https://lwn.net/Articles/1450196) | NA | v2 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/cover/1450196), [PatchWork v2](https://patchwork.kernel.org/project/linux-mm/cover/20210622121551.3398730-1-willy@infradead.org) |
| 2021/06/30 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [Folio conversion of memcg](https://lwn.net/Articles/1450196) | NA | v3 ☐ | [PatchWork v13b](https://patchwork.kernel.org/project/linux-mm/cover/20210712194551.91920-1-willy@infradead.org/) |




# 2 内存分配
-------



每个内存管理器都使用了一种基于堆的分配策略. 在这种方法中, 大块内存(称为 堆)用来为用户定义的目的提供内存. 当用户需要一块内存时, 就请求给自己分配一定大小的内存. 堆管理器会查看可用内存的情况(使用特定算法)并返回一块内存. 搜索过程中使用的一些算法有first-fit(在堆中搜索到的第一个满足请求的内存块) 和 best-fit(使用堆中满足请求的最合适的内存块). 当用户使用完内存后, 就将内存返回给堆.

这种基于堆的分配策略的根本问题是碎片(fragmentation). 当内存块被分配后, 它们会以不同的顺序在不同的时间返回. 这样会在堆中留下一些洞, 需要花一些时间才能有效地管理空闲内存. 这种算法通常具有较高的内存使用效率(分配需要的内存), 但是却需要花费更多时间来对堆进行管理.

另外一种方法称为 buddy memory allocation, 是一种更快的内存分配技术, 它将内存划分为 2 的幂次方个分区, 并使用 best-fit 方法来分配内存请求. 当用户释放内存时, 就会检查 buddy 块, 查看其相邻的内存块是否也已经被释放. 如果是的话, 将合并内存块以最小化内存碎片. 这个算法的时间效率更高, 但是由于使用 best-fit 方法的缘故, 会产生内存浪费.

## 2.1  页分配器: 伙伴分配器[<sup>12<sup>](#ref-anchor-12)
-------


### 2.1.1 BUDDY 伙伴系统
-------

古老, 具体时间难考 , 应该是生而有之. orz...

内存页分配器, 是 MM 的一个重大任务, 将内存页分配给内核或用户使用. 内核把内存页分配粒度定为 11 个层次, 叫做阶(order). 第0阶就是 2^0 个(即1个)连续物理页面, 第 1 阶就是 2^1 个(即2个)连续物理页面, ..., 以此类推, 所以最大是一次可以分配 2^10(= 1024) 个连续物理页面.



所以, MM 将所有空闲的物理页面以下列链表数组组织进来:

![](https://pic4.zhimg.com/50/1eddf633b4ec562e7bfc4b22fa5375e7_hd.jpg)


(图片来自[Physical Page Allocation](https://link.zhihu.com/?target=https%3A//www.kernel.org/doc/gorman/html/understand/understand009.html))


那伙伴(buddy)的概念从何体现?

体现在释放的时候, 当释放某个页面(组)时, MM 如果发现同一个阶中如果有某个页面(组) 跟这个要释放的页面(组) 是物理连续的, 那就把它们合并, 并升入下一阶(如: 两个 0 阶的页面, 合并后, 变为连续的2页面(组), 即一个1阶页面). 两个页面(组) 手拉手升阶, 所以叫伙伴.

关于 伙伴系统, 想了解的朋友可以参见我之前的博客.

|   日期   |   博文  |   链接   |
| ------- |:-------:|:-------:|
| 2016-06-14 | 伙伴系统之伙伴系统概述--Linux内存管理(十五) | [CSDN](https://kernel.blog.csdn.net/article/details/52420444), [GitHub](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/02-memory/04-buddy/01-buddy_system) |
| 2016-09-28 | 伙伴系统之避免碎片--Linux内存管理(十六) | [CSDN](https://blog.csdn.net/gatieme/article/details/52694362), [GitHub](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/02-memory/04-buddy/03-fragmentation) |



**关于 NUMA 支持:** Linux 内核中, 每个 zone 都有上述的链表数组, 从而提供精确到某个 node 的某个 zone 的伙伴分配需求.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/04/15 | Mel Gorman | [Remove zonelist cache and high-order watermark checking v4](https://lore.kernel.org/patchwork/cover/599755) | 优化调度器的路径, 减少对 rq->lock 的争抢, 实现 lockless. | v4 ☑ 4.4-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/599755) |
| 2016/04/15 | Mel Gorman | [Optimise page alloc/free fast paths v3](https://lore.kernel.org/patchwork/cover/668967) | 优化调度器的路径, 减少对 rq->lock 的争抢, 实现 lockless. | v3 ☑ 4.7-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/668967) |
| 2016/07/08 | Mel Gorman | [Move LRU page reclaim from zones to nodes v9](https://lore.kernel.org/patchwork/cover/696408) | 将 LRU 页面的回收从 ZONE 切换到 NODE. | v3 ☑ 4.7-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/696408) |
| 2016/07/15 | Mel Gorman | [Follow-up fixes to node-lru series v2](https://lore.kernel.org/patchwork/cover/698606) | node-lru 系列补丁的另一轮修复补丁, 防止 memcg 中警告被触发. | v3 ☑ 4.8-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/698606) |


一些核心的重构和修正


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/02/25 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [Rationalise `__alloc_pages` wrappers](https://patchwork.kernel.org/project/linux-mm/cover/20210225150642.2582252-1-willy@infradead.org) | 重构 alloc_pages 接口的调用逻辑使逻辑更清晰. | v3 ☑ 5.13-rc1 | [PatchWork v6](https://patchwork.kernel.org/project/linux-mm/cover/20210225150642.2582252-1-willy@infradead.org) |


### 2.1.2 fair allocation
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/08/02 | Mel Gorman <mgorman@techsingularity.net> | [mm: improve page aging fairness between zones/nodes](https://lore.kernel.org/patchwork/cover/397316) | NA | v2 ☑ 3.12-rc1 | [PatchWork v2,0/3](https://lore.kernel.org/patchwork/cover/397316) |
| 2013/12/18 | Mel Gorman <mgorman@techsingularity.net> | [Configurable fair allocation zone policy v4](https://lore.kernel.org/patchwork/cover/397316) | NA | v4 ☐ | [PatchWork RFC v4,0/6](https://lore.kernel.org/patchwork/cover/397316) |
| 2016/04/15 | Mel Gorman <mgorman@techsingularity.net> | [mm, page_alloc: Reduce cost of fair zone allocation policy retry](https://lore.kernel.org/patchwork/patch/668985) | [Optimise page alloc/free fast paths v3](https://lore.kernel.org/patchwork/cover/668967) 系列中的一个补丁. 降低了 fair zone 分配器的开销. | v3 ☑ 4.7-rc1 | [PatchWork v6 00/28](https://lore.kernel.org/patchwork/cover/668967) |
| 2016/07/08 | Mel Gorman <mgorman@techsingularity.net> |  | [Move LRU page reclaim from zones to nodes v9](https://lore.kernel.org/patchwork/cover/696437)系列中的一个补丁. 公平区域分配策略在区域之间交叉分配请求, 以避免年龄倒置问题, 即回收新页面来平衡区域. LRU 回收现在是基于节点的, 所以这应该不再是一个问题, 公平区域分配策略本身开销也不小, 因此这个补丁移除了它. | v9 ☑ [4.8-rc1](https://kernelnewbies.org/Linux_4.8#Memory_management) | [PatchWork v21](https://lore.kernel.org/patchwork/cover/696437) |


### 2.1.3 内存水线
-------

Linux 为每个 zone 都设置了独立的 min, low 和 high 三个档位的 watermark 值, 在代码中以struct zone中的 `_watermark[NR_WMARK]` 来表示.

*   在进行内存分配的时候, 如果伙伴系统发现当前空余内存的值低于"low"但高于"min", 说明现在内存面临一定的压力, 但是并不是非常紧张, 那么在此次内存分配完成后, kswapd将被唤醒, 以执行内存回收操作. 在这种情况下, 内存分配虽然会触发内存回收, 但不存在被内存回收所阻塞的问题, 两者的执行关系是异步的.

*   如果内存分配器发现空余内存的值低于了 "min", 说明现在内存严重不足. 那么这时候就有必要等待内存回收完成后, 再进行内存的分配了, 也就是 "direct reclaim". 但是这里面有个别特例, 内核提供了 PF_MEMALLOC 标记, 如果现在空余内存的大小可以满足本次内存分配的需求, 允许设置了 PF_MEMALLOC 标记的进程在内存紧张时, 先分配, 再回收. 比如 kswapd, 由于其本身就是负责回收内存的, 只需要满足它很小的需求, 它会回收大量的内存回来. 它就像公司濒临破产时抓到的一根救命稻草, 只需要小小的付出, 就会让公司起死回生.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/01/07 | Satoru Moriya <satoru.moriya@hds.com> | [Tunable watermark](https://lore.kernel.org/patchwork/cover/231713) | 引入可调的水线. 为 min/low/high 各个水线都引入了一个 sysctl 接口用于调节. | v1 ☐ | [PatchWork RFC,0/2](https://lore.kernel.org/patchwork/cover/231713) |
| 2013/02/17 | dormando <dormando@rydia.net><br>Rik van Riel <riel@redhat.com> | [add extra free kbytes tunable](https://lore.kernel.org/patchwork/cover/360274) | 默认内核中 min 和 low 之间的距离太短, 造成 kswapd 的作用空间太小, 从而导致频繁出现 direct reclaim. 这个补丁引入 extra_free_kbytes, 作为计算 low 时候的加权. 从而增大 min 和 low 之间的距离. | v1 ☐ | [PatchWork v5](https://lore.kernel.org/patchwork/cover/360274) |
| 2016/02/22 | Johannes Weiner <hannes@cmpxchg.org> | [mm: scale kswapd watermarks in proportion to memory](https://lore.kernel.org/patchwork/cover/649909) |  | v2 ☑ 4.6-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/360274) |
| 2018/11/23 | Mel Gorman | [Fragmentation avoidance improvements v5](https://lore.kernel.org/patchwork/cover/1016503) | 伙伴系统页面分配时的反碎片化 | v5 ☑ 5.0-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/1016503) |
| 2020/02/25 | Mel Gorman | [Limit runaway reclaim due to watermark boosting](https://lore.kernel.org/patchwork/cover/1200172) | 优化调度器的路径, 减少对 rq->lock 的争抢, 实现 lockless. | v4 ☑ 4.4-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/1200172) |
| 2020/06/11 |Charan Teja Kalla <charante@codeaurora.org> | [mm, page_alloc: skip ->waternark_boost for atomic order-0 allocations](https://lore.kernel.org/patchwork/cover/1254998) | NA | v1 ☑ 5.9-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/1244272), [](https://lore.kernel.org/patchwork/patch/1254998) |
| 2020/10/20 |Charan Teja Kalla <charante@codeaurora.org> | [mm: don't wake kswapd prematurely when watermark boosting is disabled](https://lore.kernel.org/patchwork/cover/1322999) | NA | v1 ☑ 5.11-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/1244272), [PatchWork](https://lore.kernel.org/patchwork/patch/1322999) |
| 2020/05/01 |Charan Teja Kalla <charante@codeaurora.org> | [mm: Limit boost_watermark on small zones.](https://lore.kernel.org/patchwork/cover/1234105) | NA | v1 ☑ 5.11-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/1234105) |


### 2.1.4 PCP(Per CPU Page) Allocation
-------

*   [Hot and cold pages](https://lwn.net/Articles/14768)

[Linux 中的冷热页机制概述](https://toutiao.io/posts/d4cz9u/preview)

[内存管理中的cold page和hot page,  冷页 vs 热页](https://blog.csdn.net/kickxxx/article/details/9306361)

v2.5.45, Martin Bligh 和 Andrew Morton 以及其他人提交了一个内核分配器 patch, 引入了 hot-n-cold pages 的概念, 这个概念本身是和现在处理器架构息息相关的. 尽量利用处理器 cache, 避免使用主存, 可以提升性能. hot-cold page 就是这样的思路. 处理器cache保存着最近访问的内存. kernel 认为最近访问的内存很有可能存在于cache之中. hot-cold page patch 为 每个 zone 建立了一个 [per-CPU 的页面缓存](https://elixir.bootlin.com/linux/v2.5.45/source/include/linux/mmzone.h#L125), 页面缓存中包含了[ cold 和 hot 两种页面](https://elixir.bootlin.com/linux/v2.5.45/source/include/linux/mmzone.h#L59) 每个内存zone). 当 kernel 释放的 page 可能是 hot page 时([page cache 的页面往往被认为是在 cache 中的](https://elixir.bootlin.com/linux/v2.5.45/source/mm/swap.c#L87), 是 hot 的), 那么就把它[放入hot链表](https://elixir.bootlin.com/linux/v2.5.45/source/mm/page_alloc.c#L558), 否则放入 cold 链表.

1.  当 kernel 需要分配一个page时, 新分配器通常会从 per-CPU 的 hot list 获取页面, 甚至我们获得的页面马上就要写入新数据的情况下, 仍然能获得较好的速度.

2.  当然也有些情况下, 申请 hot page 不会获得性能上的提高, 只要申请 cold page 就可以了. 比如 [DMA 读操作需要的内存分配](https://elixir.bootlin.com/linux/v2.5.45/C/ident/page_cache_alloc_cold), 设备会直接修改内存并且无效相应的 cache. 所以内核分配器提供了 [GFP_COLD分配标记](https://elixir.bootlin.com/linux/v2.5.45/source/mm/page_alloc.c#L412) 来显式从 cold page 链表分配内存.

此外使用 per-CPU page 链表也削减了锁竞争, 提高了性能. Andrew Morton 测试了这个patch, 在不同环境下获得了 `%1~%12` 不等的性能提升.


```cpp
8d6282a1cf8 [PATCH] hot-n-cold pages: free and allocate hints
5019ce29f74 [PATCH] hot-n-cold pages: use cold pages for readahead
a206231bbe6 [PATCH] hot-n-cold pages: page allocator core
1d2652dd2c3 [PATCH] hot-n-cold pages: bulk page freeing
38e419f5b01 [PATCH] hot-n-cold pages: bulk page allocator
```

后来经过测试后, 大家一致认为, 将热门页面和冷页面分开列出可能没有什么意义. 因为有一种方法可以连接这两个列表: 使用单一列表, 将冷页面放在末尾, 将热页面放在开头. 这样, 一个列表就可以为这两种类型的分配服务. [Page allocator: get rid of the list of cold pages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3dfa5721f12c3d5a441448086bee156887daa961).

这样 free_cold_page 函数已经没意义了, 被删除掉, [page-allocator: Remove dead function free_cold_page()](https://lore.kernel.org/patchwork/patch/166245), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=38a398572fa2d8124f7479e40db581b5b72719c9). 以及 [free_hot_page](https://lore.kernel.org/patchwork/patch/185034), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fc91668eaf9e7ba61e867fc2218b7e9fb67faa4f).

随后, 由于页面空闲路径没有区分缓存热页面和冷页面, 更是将 `__GFP_COLD` 标记删掉 [mm: remove `__GFP_COLD`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=453f85d43fa9ee243f0fc3ac4e1be45615301e3f). 当前空闲列表中没有真正有用的页面排序, 分配请求无法利用这些排序, 因此 `__GFP_COLD` 已经没有明确的意义. 删除 `__GFP_COLD` 参数还简化了页面分配器中的一些路径.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/10/17 | Jan Kara <jack@suse.cz> | [Speed up page cache truncation v2](https://patchwork.kernel.org/project/linux-fsdevel/patch/20171017162120.30990-2-jack@suse.cz) | NA | v2 ☐ | [PatchwWork v2](https://patchwork.kernel.org/project/linux-fsdevel/patch/20171017162120.30990-2-jack@suse.cz) |
| 2017/10/18 | Mel Gorman <mgorman@techsingularity.net> | [Follow-up for speed up page cache truncation v2](https://lore.kernel.org/patchwork/cover/842268) | NA | v2 ☑ 4.15-rc1 | [PatchwWork v2](https://lore.kernel.org/patchwork/cover/842268) |
| 2020/11/11 | Vlastimil Babka <vbabka@suse.cz> | [disable pcplists during memory offline](https://lore.kernel.org/patchwork/cover/1336780) | 当内存下线的时候, 禁用 PCP | v3 ☑ [5.11-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [v4](https://lore.kernel.org/patchwork/cover/1336780) |


*   [High-order per-cpu page allocator](https://lore.kernel.org/patchwork/patch/740275)

很长一段时间以来, SLUB 一直是默认的小型内核对象分配器, 但由于性能问题和对高阶页面的依赖, 它并不是普遍使用的. 高阶关注点有两个主要组件——高阶页面并不总是可用, 高阶页面分配可能会在 zone->lock 上发生冲突.

[mm: page_alloc: High-order per-cpu page allocator v4](https://lore.kernel.org/patchwork/patch/740275) 通过扩展 Per CPU Pages(PCP) 分配器来缓存高阶页面, 解决了一些关于区域锁争用的问题. 这个补丁做了以下修改

1.  添加了新的 Per CPU 列表来缓存高阶页面. 这会增加 Per CPU Allocation 的缓存空间和总体使用量, 但对于某些工作负载, 这将通过减少 zone->lock 上的争用而抵消. 列表中的第一个 MIGRATE_PCPTYPE 条目是每个 migratetype 的. 剩余的是高阶缓存, 直到并包括 PAGE_ALLOC_COSTLY_ORDER. 页面释放时, PCP 的计算现在被限制为 free_pcppages_bulk, 因为调用者不可能知道到底释放了多少页. 由于使用了高阶缓存, 请求耗尽的页面数量不再精确.

2.  增加 Per CPU Pages 的高水位, 以减少一次重新填充导致下一个空闲页面的溢出的概率. 这个改动的优化效果跟硬件环境和工作负载有较大的关系, 取决定因素的是当前有业务在 zone->lock 上的反弹和争用是否占主导.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/12/01 | Mel Gorman | [mm: page_alloc: High-order per-cpu page allocator v4](https://lore.kernel.org/patchwork/cover/740275) | 为高阶内存分配提供 Per CPU Pages 缓存  | v4 ☐ | [v4](https://lore.kernel.org/patchwork/cover/740275) |

*   [Bulk memory allocation](https://lwn.net/Articles/711075)

由于存储设备和网络接口等外围设备可以处理不断增加的数据速率, 因此内核面临许多可扩展性挑战. 通常, 提高吞吐量的关键是分批工作. 在许多情况下, 执行一系列相关操作的开销不会比执行单个操作的开销高很多. 内存分配是批处理可以显着提高性能的地方, 但是到目前为止, 关于如何进行批处理社区进行了多次激烈的讨论.

举例来说, 网络接口往往需要大量内存. 毕竟, 所有这些传入的数据包都必须放在某个地方. 但是分配该内存的开销很高, 以至于它可能会限制整个系统的最大吞吐量. 之前网络驱动开发人员的做法都是采用诸如先分配(大 order 的高阶内存后)再拆分(成小 order 的低阶内存块) 的折衷方案. 但是高阶页面分配会给整个系统带来压力. 参见 [Generic page-pool recycle facility ?](https://people.netfilter.org/hawk/presentations/MM-summit2016/generic_page_pool_mm_summit2016.pdf)

在 2016 年 [Linux 存储, 文件系统和内存管理峰会 (Linux Storage, Filesystem, and Memory-Management Summit)](https://lwn.net/Articles/lsfmm2016) 上, 网络开发人员 Jesper Dangaard Brouer 提议创建一个[新的内存分配器](https://lwn.net/Articles/684616), 该分配器从一开始就设计用于批处理操作. 驱动程序可以使用它在一个调用中分配许多页面, 从而最大程度地减少了每页的开销. 在这次会议上, 内存管理开发人员 Mel Gorman 了解了此问题, 但不同意他创建新分配器的想法. 因为这样做会降低内存管理子系统的可维护性. 另外, 随着新的分配器功能的不断完善, 新的分配器遇到现有分配器同样的问题, 比如 NUMA 上的一些处理, 并且当它想要完成具有所有必需的功能时, 它并不见得完成的更快. 参见 [Bulk memory allocation](https://lwn.net/Articles/711075). Mel Gorman 认为最好使用现有的 Per CPU Allocator 并针对这种情况进行优化. 那么内核中的所有用户都会受益. 他现在[有了一个补丁](https://lore.kernel.org/patchwork/cover/747351), 在特性的场景下, 它可以将页面分配器的开销减半, 且用户不必关心 NUMA 结构.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/01/4 | Mel Gorman | [Fast noirq bulk page allocator](https://lore.kernel.org/patchwork/cover/747351) | 中断安全的批量内存分配器, RFC 补丁, 最终 Mel Gorman 为了完成这组优化做了大量的重构和准备工作   | v5 ☐ | [RFC](https://lore.kernel.org/patchwork/cover/747351)<br>*-*-*-*-*-*-*-* <br>[RFC v2](https://lore.kernel.org/patchwork/cover/749110) |
| 2017/01/23 | Mel Gorman | [Use per-cpu allocator for !irq requests and prepare for a bulk allocator v5](https://lore.kernel.org/patchwork/cover/753645) | 重构了 Per CPU Pages 分配器, 使它独占 !irq 请求, 这将减少大约 30% 的分配/释放开销. 这是完成 Bulk memory allocation 工作的第一步  | v5 ☑ 4.11-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/753645) |
| 2017/01/25 | Mel Gorman | [mm, page_alloc: Use static global work_struct for draining per-cpu pages](https://lore.kernel.org/patchwork/cover/754235) | 正如 Vlastimil Babka 和 Tejun Heo 所建议的, 这个补丁使用一个静态 work_struct 来协调 Per CPU Pages 在工作队列上的排泄. 一次只能有一个任务耗尽, 但这比以前允许多个任务同时发送IPIs的方案要好. 一个需要考虑的问题是并行请求是否应该彼此同步. | v5 ☑ 4.11-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/754235) |
| 2017/01/25 | Mel Gorman | [Recalculate per-cpu page allocator batch and high limits after deferred meminit](https://lore.kernel.org/patchwork/cover/1141598) | 由于 PCP(Per CPU Page) Allocation 中不正确的高限制导致的高阶区域 zone->lock 的竞争, 在初始化阶段, 但是在初始化结束之前, PCP 分配器会计算分批分配/释放的页面数量, 以及 Per CPU 列表上允许的最大页面数量. 由于 zone->managed_pages 还不是最新的, pcp 初始化计算不适当的低批量和高值. 在某些情况下, 这会严重增加区域锁争用, 严重程度取决于共享一个本地区域的cpu数量和区域的大小. 这个问题导致了构建内核的时间比预期的长得多时, AMD epyc2 机器上的系统 CPU 消耗也过多. 这组补丁修复了这个问题 | v5 ☑ 4.11-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/1141598) |
| 2021/03/25 | Mel Gorman | [Introduce a bulk order-0 page allocator with two in-tree users](https://lore.kernel.org/patchwork/cover/1399888) | 批量 order-0 页面分配器, 目前 sunrpc 和 network 页面池是这个特性的第一个用户 | v6 ☑ 5.13-rc1 | [RFC](https://lore.kernel.org/patchwork/cover/1383906)<br>*-*-*-*-*-*-*-* <br>[v1](https://lore.kernel.org/patchwork/cover/1385629)<br>*-*-*-*-*-*-*-* <br>[v2](https://lore.kernel.org/patchwork/cover/1392670)<br>*-*-*-*-*-*-*-* <br>[v3](https://lore.kernel.org/patchwork/cover/1393519)<br>*-*-*-*-*-*-*-* <br>[v4](https://lore.kernel.org/patchwork/cover/1394347)<br>*-*-*-*-*-*-*-* <br>[v5](https://lore.kernel.org/patchwork/cover/1399888)<br>*-*-*-*-*-*-*-* <br>[v6](https://lore.kernel.org/patchwork/cover/1402140) |
| 2021/03/29 | Mel Gorman | [Use local_lock for pcp protection and reduce stat overhead](https://lore.kernel.org/patchwork/cover/1404513) | Bulk memory allocation 的第一组修复补丁, PCP 与 vmstat 共享锁定要求, 这很不方便, 并且会导致一些问题. 可能因为这个原因, PCP 链表和 vmstat 共享相同的 Per CPU 空间, 这意味着 vmstat 可能跨 CPU 更新包含 Per CPU 列表的脏缓存行, 除非使用填充. 该补丁集拆分该结构并分离了锁. | RFC ☐ | [RFC](https://lore.kernel.org/patchwork/cover/1404513) |
| 2020/03/20 | Mel Gorman | [mm/page_alloc: Add a bulk page allocator -fix -fix](https://lore.kernel.org/patchwork/cover/1405057) | Bulk memory allocation 的第二组修复补丁 | v1 ☐ | [RFC](https://lore.kernel.org/patchwork/cover/1405057) |

*   Adjust high and batch

percpu 页分配器(PCP)旨在减少对区域锁的争用, 但是 PCP 中的页面数量也要有限制. 因此 PCP 引入了 high 和 batch 来控制 pcplist 中的页面大小. 当 PCP 中的页面超过了 pcp->high 的时候, 则会释放 batch 的页面回到 BUDDY 中.

2.6.16 时, 内核通过 [commit Making high and batch sizes of per_cpu_pagelists configurable](https://lore.kernel.org/patchwork/cover/47659) 引入了一个参数 percpu_pagelist_fraction 用来设置 PCP->high 的大小. 而 PCP->batch 的大小将被设置为 min(high / 4, PAGE_SHIFT * 8).

时间来到 2021 年, batch 和 high 的大小已经过时了, 既不考虑 zone 大小, 也不考虑一个区域的本地节点上 CPU 的数量. PCP 的空间往往非常有限, 随着更大的 zone 和每个节点更多的 CPU, 将导致争用情况越来越糟, 特别是通过 `release_pages()` 批量释放页面时 zone->lock 争用尤为明显. 虽然可以使用 vm.percpu_pagelist_fraction 来增加 PCP->high 来减少争用, 但是由于 vm.percpu_pagelist_fraction 同时调整 high 和 batch 两个值, 虽然可以一定程度减少 zone->lock 锁的争用, 但也会增加分配延迟.

Mel Gorman 发现了这一问题, 开发了 [Calculate pcp->high based on zone sizes and active CPUs](https://lore.kernel.org/patchwork/cover/1435878) 将 pcp->high 和 pcp->batch 的设置分离, 然后根据 local zone 的大小扩展 pcp->high, 对活动 CPU 的回收和计算影响有限, 但 PCP->batch 保持静态. 它还根据最近的释放模式调整可以在 PCP 列表上的页面数量.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/12/09 | Mel Gorman <mgorman@techsingularity.net> | [Making high and batch sizes of per_cpu_pagelists configurable](https://lore.kernel.org/patchwork/cover/47659) | 引入了 percpu_pagelist_fraction 来调整各个 zone PCP 的 high, 同时将 batch 值设置为 min(high / 4, PAGE_SHIFT * 8).  | v1 ☑ 2.6.16-rc1 | [RFC](https://lore.kernel.org/patchwork/cover/47659), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8ad4b1fb8205340dba16b63467bb23efc27264d6) |
| 2021/05/25 | Mel Gorman <mgorman@techsingularity.net> | [Calculate pcp->high based on zone sizes and active CPUs](https://lore.kernel.org/patchwork/cover/1435878) | pcp->high 和 pcp->batch 根据 zone 内内存的大小进行调整. 移除了不适用的 vm.percpu_pagelist_fraction 参数. | v2 ☑ 5.14-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/1435878) |
| 2021/06/03 | Mel Gorman <mgorman@techsingularity.net> | [Allow high order pages to be stored on PCP v2](https://lore.kernel.org/patchwork/cover/1440776) | PCP 支持缓存高 order 的页面. | v2 ☑ 5.14-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/1440776) |


### 2.1.5 ALLOC_NOFRAGMENT 优化
-------

页面分配最容易出现的就是外碎片化问题, 因此主线进行了锲而不舍的优化, Mel Gorman 提出的 [Fragmentation avoidance improvements v5](https://lore.kernel.org/patchwork/cover/1016503) 是比较有特色的一组.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/11/23 | Mel Gorman | [Fragmentation avoidance improvements v5](https://lore.kernel.org/patchwork/cover/1016503) | 优化调度器的路径, 减少对 rq->lock 的争抢, 实现 lockless. | v5 ☑ 5.0-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/1016503) |



当发现某个 ZONE 上内存分配可能出现紧张的时候, 那么有两种选择:

| 方式 | 描述 | 注意 | 触发场景 |
|:---:|:----:|:---:|:-------:|
| 从当前 ZONE(比如 ZONE_NORMAL) 的低端 ZONE(比如 ZONE_DMA32) 中去分配, 这样可以防止分配内存时将当前 ZONE 做了分割, 久而久之就造成了碎片化 | 在 x86 上, 节点 0 可能有多个分区, 而其他节点只有一个分区. 这样造成的结果是, 运行在节点0上的任务可能会导致 ZONE_NORMAL 区域出现了碎片化, 但是其实 ZONE_DMA32 区域还有足够的空闲内存. 如果(此次分配采取其他方式)**将来会导致外部碎片问题**, 在分配器的快速路径, 这样它将尝试从较低的本地 ZONE (比如 ZONE_DMA32) 分配, 然后才会尝试去分割较高的区域(比如 ZONE_NORMAL). |  | 会导致外部碎片问题的事件 |
| 从同 ZONE 的其他迁移类型的空闲链表中窃取内存过来 | 而**当发生了外碎片化的时候**, 则更倾向于从 ZONE_NORMAL 区域中其他迁移类型的空闲链表中多挪用一些空闲内存过来, 这比过早地使用低端 ZONE 区域的内存要很好多. | 理想情况下, 总是希望从其他迁移类型的空闲链表中至少挪用 2^pageblock_order 个空闲页面, 参见 __rmqueue_fallback 函数 | 已经发生了外碎片化 |

怎么取舍和选择两种方式是非常微妙的.

- [x] 如果首选的 ZONE 是高端的(比如 ZONE_NORMAL), 那么过早使用较低的低层次的 ZONE(ZONE_DMA32) 可能会导致这些低层次的 ZONE 面临内存短缺的压力, 这通常比破碎化更严重. 特别的, 如果下一个区域是ZONE_DMA, 那么它可能太小了. 因此只有分散分配才能避免正常区域和DMA32区域之间的碎片化. 参见 alloc_flags_nofragment 及其函数注释.

- [x] 如果总是从当前 ZONE 的其他 MIGRATE_TYPE 窃取内存过来, 将导致两个 ZONE 的内存都被切分成多块. 长此以往, 系统中将满是零零碎碎的切片, 将导致碎片化.

之前在 `__alloc_pages_nodemas` 中. 页面分配器基于每个节点的 ZONE 迭代, 而没有考虑抗碎片化. 补丁 1 [mm, page_alloc: spread allocations across zones before introducing fragmentation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6bb154504f8b496780ec53ec81aba957a12981fa), 因此引入了一个新的选项 ALLOC_NOFRAGMENT, 来实现页面分配时候的抗外碎片化. 在特殊的场景下, 倾向于有限分配较低 ZONE 区域的内存, 而不是对较高的区域做切片. 补丁 1 增加了 `__alloc_pages_nodemas` 的 FRAGMENT AWARE 支持后, 页面分配的行为发生了变化. 如果发现当前 ZONE 中存在碎片化倾向(开始尝试执行 `__rmqueue_fallback` 去其他分组窃取内存)的时候:

*   如果分配内存的 order < pageblock_order 时, 就认为此操作即将导致外碎片化问题, 则倾向于从低端的 ZONE 中分配.

*   如果已经发生了碎片化, 或者分配的内存超过了 pageblock_order, 则倾向于从其他 MIGRATE_TYPE 中窃取内存过来分配.


补丁 2-4 [1c30844d2dfe mm: reclaim small amounts of memory when an external fragmentation event occurs
](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1c30844d2dfe) 则引入了 boost_watermark 机制, 在外部碎片事件发生时暂时提高水印. kswapd 唤醒以回收少量旧内存, 然后在完成时唤醒 kcompactd 以稍微恢复系统. 这在slowpath中引入了一些开销. 提升的级别可以根据对碎片和分配延迟的容忍程度进行调整或禁用.

补丁5暂停了一些可移动的分配请求, 以让补丁4的 kswapd 取得一些进展. 档位的持续时间很短, 但是如果可以容忍更大的档位, 则可以调整系统以避免碎片事件.

整个补丁在测试场景下将外碎片时间减少 94% 以上, 这收益大部分来自于补丁 1-4, 但补丁 5 可以处理罕见的特例, 并为THP分配成功率提供调整系统的选项, 以换取一些档位来控制碎片化.


### 2.1.6 页面窃取 page stealing
-------


在使用伙伴系统申请内存页面时, 如果所请求的 migratetype 的空闲页面列表中没有足够的内存, 伙伴系统尝试从其他不同的页面中窃取内存.

这会造成减少永久碎片化, 因此伙伴系统使用了各种各样启发式的方法, 尽可能的使这一事件不要那么频繁地触发,  最主要的思路是尝试从拥有最多免费页面的页面块中窃取, 并可能一次窃取尽量多的页面. 但是精确地搜索这样的页面块, 并且一次窃取整个页面块, 是昂贵的, 因此启发式方法是免费的列出从MAX_ORDER到请求的顺序, 并假设拥有最高次序空闲页面的块可能也拥有总数最多的空闲页面.

很有可能, 除了最高顺序的页面, 我们还从同一块中窃取低顺序的页面. 但我们还是分走了最高订单页面. 这是一种浪费, 会导致碎片化, 而不是避免碎片化.



因此, 这个补丁将__rmqueue_fallback()更改为仅仅窃取页面并将它们放到请求的migratetype的自由列表中, 并且只报告它是否成功. 然后我们使用__rmqueue_least()选择(并最终分割)最小的页面. 这一切都是在区域锁定下发生的, 所以在这个过程中没有人能从我们这里偷走它. 这应该可以减少由于回退造成的碎片. 在最坏的情况下, 我们只是窃取了一个最高顺序的页面, 并通过在列表之间移动它, 然后删除它而浪费了一些周期, 但后退并不是真正的热门路径, 所以这不应该是一个问题. 作为附带的好处, 该补丁通过重用__rmqueue_least()删除了一些重复的代码.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/01/23 | Vlastimil Babka <vbabka@suse.cz> | [page stealing tweaks](https://lore.kernel.org/patchwork/cover/535613) |  | v1 ☑ 4.13-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/cover/535613) |
| 2017/03/07 | Vlastimil Babka <vbabka@suse.cz> | [try to reduce fragmenting fallbacks](https://lore.kernel.org/patchwork/cover/766804) | 修复 [Regression in mobility grouping?](https://lkml.org/lkml/2016/9/28/94) 上报的碎片化问题, 通过修改 fallback 机制和 compaction 机制来减少永久随便化的可能性. 其中 fallback 修改时, 仅尝试从不同 migratetype 的 pageblock 中窃取的页面中挑选最小(但足够)的页面. | v3 ☑ [4.12-rc1](https://kernelnewbies.org/Linux_4.12#Memory_management) | [PatchWork v6](https://lore.kernel.org/patchwork/cover/766804), [KernelNewbies](https://kernelnewbies.org/Linux_4.12#Memory_management), [关键 commit 3bc48f96cf11 ("mm, page_alloc: split least stolen page in fallback")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3bc48f96cf11ce8699e419d5e47ae0d456403274) |
| 2017/05/29 | Vlastimil Babka <vbabka@suse.cz> | [mm, page_alloc: fallback to smallest page when not stealing whole pageblock](https://lore.kernel.org/patchwork/cover/793063) |  | v1 ☑ 4.13-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/cover/793063), [commit 7a8f58f39188](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7a8f58f3918869dda0d71b2e9245baedbbe7bc5e) |

commit fef903efcf0cb9721f3f2da719daec9bbc26f12b
Author: Srivatsa S. Bhat <srivatsa.bhat@linux.vnet.ibm.com>
Date:   Wed Sep 11 14:20:35 2013 -0700

    mm/page_allo.c: restructure free-page stealing code and fix a bug

### 2.1.7 优化
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/04/15 | Mel Gorman <mgorman@techsingularity.net> | [Optimise page alloc/free fast paths v3](https://lore.kernel.org/patchwork/cover/668967) | 优化 page 申请和释放的快速路径. 优化后<br>1. 在 free 路径中, 调试检查和页面区域/页面块仍然查找占主导地位, 目前仍没有明显的解决方案. 在 alloc 路径中, 主要的耗时操作是处理 zonelist、新页面准备和 fair zone 分配以及无数的统计更新. | v3 ☑ 4.7-rc1 | [PatchWork v6 00/28](https://lore.kernel.org/patchwork/cover/668967) |



### 2.1.7 重构
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/04/15 | Mel Gorman <mgorman@techsingularity.net> | [Rationalise `__alloc_pages` wrappers](https://patchwork.kernel.org/project/linux-mm/cover/20210225150642.2582252-1-willy@infradead.org) | NA | v3 ☑ 4.7-rc1 | [PatchWork v3,0/7](https://patchwork.kernel.org/project/linux-mm/cover/20210225150642.2582252-1-willy@infradead.org) |


## 2.2 内核级别的 malloc 分配器之-对象分配器(小内存分配)
-------

伙伴系统是每次分配内存最小都是以页(4KB)为单位的, 页面分配和回收起来很方便, 但是在实际使用过程中还是有一些问题的.

1.  系统运行的时候使用的绝大部分的数据结构都是很小的, 为一个小对象分配4KB显然是不划算了.

2.  内核中常见的是经常分配某种固定大小尺寸的对象, 并且对象都需要一定的初始化操作, 这个初始化操作有时比分配操作还要费时


因此 Linux 需要一个小块内存的快速分配方案:

1.      一个解决方法是用缓存池把这些对象管理起来, 把第一次分配作初始化; 释放时析构为这个初始状态, 这样能提高效率.

2.      此外, 增加一个缓存池, 把不同大小的对象分类管理起来, 这样能更高效地使用内存. 试想用固定尺寸的页分配器来分配给对象使用, 则不可避免会出现大量内部碎片.


因此 SLAB 等对象分配器应运而生.

BUDDY 提供了页面的分配和回收机制, 而在运行时, SLAB 等对象分配器向 BUDDY 一次性"批发"一些内存, 加工切块以后"散卖"出去. 等自己"批发"的库存耗尽了, 那么就再去向 BUDDY 申请批发一些. 在这个过程中, BUDDY 是一个内存的批发商, 接受一些大的内存订单, 而 SLAB 等对象分配器像是一个二级分销商, 把申请的内存零售出去, 一段时间以后如果客户不需要使用内存了, 那么 SLAB 这些分销商再零碎的回收回去, 当自己库存足够多的时候, 会再把库存退回给 BUDDY 这个批发商.


https://lore.kernel.org/patchwork/patch/47616/
https://lore.kernel.org/patchwork/patch/46669/
https://lore.kernel.org/patchwork/patch/46671/

https://lore.kernel.org/patchwork/cover/408914


### 2.2.1 SLAB
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/05/14 | Christoph Lameter <clameter@engr.sgi.com> | [NUMA aware slab allocator V3](https://lore.kernel.org/patchwork/cover/38309) | SLAB 分配器感知 NUMA | v3 ☑ 2.6.22-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/38309) |
| 2005/11/18 | Christoph Lameter <clameter@engr.sgi.com> | [NUMA policies in the slab allocator V2](https://lore.kernel.org/patchwork/cover/38309) | SLAB 分配器感知 NUMA | v3 ☑ 2.6.16-rc2 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/38309) |
| 2007/02/28 | Mel Gorman | [mm/slab: reduce lock contention in alloc path](https://lore.kernel.org/patchwork/cover/667440) | 优化 SLAB 分配的路径, 减少对 lock 的争抢, 实现 lockless. | v2 ☑ 2.6.22-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/667440) |


**2.0 版本时代(1996年引入)**

这是最早引入的对象分配器, Linux 所使用的 slab 分配器的基础是 Jeff Bonwick 为 SunOS 操作系统首次引入的一种算法. Jeff 的分配器是围绕对象缓存进行的. 在内核中, 会为有限的对象集(例如文件描述符和其他常见结构)分配大量内存. Jeff 发现对内核中普通对象进行初始化所需的时间超过了对其进行分配和释放所需的时间. 因此他的结论是不应该将内存释放回一个全局的内存池, 而是将内存保持为针对特定目而初始化的状态. 例如, 如果内存被分配给了一个互斥锁, 那么只需在为互斥锁首次分配内存时执行一次互斥锁初始化函数(mutex_init)即可. 后续的内存分配不需要执行这个初始化函数, 因为从上次释放和调用析构之后, 它已经处于所需的状态中了. 这里想说的主要是我们得到一块很原始的内存, 必须要经过一定的初始化之后才能用于特定的目的. 当我们用slab时, 内存不会被释放到全局内存池中, 所以还是处于特定的初始化状态的. 这样就能加速我们的处理过程.

Linux slab 分配器使用了这种思想和其他一些思想来构建一个在空间和时间上都具有高效性的内存分配器.

1.  小对象的申请和释放通过slab分配器来管理.
    SLAB 基于页分配器分配而来的页面(组), 实现自己的对象缓存管理. 它提供预定尺寸的对象缓存, 也支持用户自定义对象缓存

2.  slab分配器有一组高速缓存, 每个高速缓存保存同一种对象类型, 如i节点缓存、PCB缓存等.
    维护着每个 CPU , 每个 NUMA node 的缓存队列层级, 可以提供高效的对象分配

3.  内核从它们各自的缓存种分配和释放对象.

4.  每种对象的缓存区由一连串slab构成, 每个slab由一个或者多个连续的物理页面组成
    *   这些页面种包含了已分配的缓存对象, 也包含了空闲对象.
    *   还支持硬件缓存对齐和着色, 所谓着色, 就是把不同对象地址, 以缓存行对单元错开, 从而使不同对象占用不同的缓存行, 从而提高缓存的利用率并获得更好的性能

与传统的内存管理模式相比,  slab 缓存分配器提供了很多优点. 首先, 内核通常依赖于对小对象的分配, 它们会在系统生命周期内进行无数次分配. slab 缓存分配器通过对类似大小的对象进行缓存而提供这种功能, 从而避免了常见的碎片问题. slab 分配器还支持通用对象的初始化, 从而避免了为同一目而对一个对象重复进行初始化. 最后, slab 分配器还可以支持硬件缓存对齐和着色, 这允许不同缓存中的对象占用相同的缓存行, 从而提高缓存的利用率并获得更好的性能.


### 2.2.2 SLUB
-------

随着大规模多处理器系统和NUMA系统的广泛应用, slab终于暴露出不足:

1.  复杂的队列管理

2.  管理数据和队列存储开销较大

3.  长时间运行 partial 队列可能会非常长

4.  对 NUMA 支持非常复杂

为了解决这些高手们开发了slub: 改造 page 结构来削减 slab 管理结构的开销、每个 CPU 都有一个本地活动的 slab(kmem_cache_cpu)等. 对于小型的嵌入式系统存在一个 slab模拟层slob, 在这种系统中它更有优势.

**2.6.22(2007年7月发布)**


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/02/28 | Christoph Lameter <clameter@engr.sgi.com> | [SLUB The unqueued slab allocator V3](https://lore.kernel.org/patchwork/cover/75156) | 优化调度器的路径, 减少对 rq->lock 的争抢, 实现 lockless. | v3 ☑ 2.6.22-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/75156), [commit 81819f0fc828 SLUB core](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=81819f0fc8285a2a5a921c019e3e3d7b6169d225) |
| 2011/08/09 | Christoph Lameter <cl@linux.com> | [slub: per cpu partial lists V4](https://lore.kernel.org/patchwork/cover/262225) | slub 的 per-cpu 页面池, 有助于避免每个节点锁定开销, 使得 SLUB 的部分工作(比如fastpath和free slowpath)可以进一步在不禁用中断的情况下工作, 可以进一步减少分配器的延迟. | v3 ☑ 2.6.22-rc1 | [PatchWork v6](https://lore.kernel.org/patchwork/cover/262225) |


SLUB 这是第二个对象分配器实现. 引入这个新的实现的原因是 SLAB 存在的一些问题. 比如 NUMA 的支持, SLAB 引入时内核还没支持 NUMA, 因此, 一开始就没把 NUMA 的需求放在设计理念里, 结果导致后来的对 NUMA 的支持比较臃肿奇怪, 一个典型的问题是, SLAB 为追踪这些缓存, 在每个 CPU, 每个 node, 上都维护着对象队列. 同时, 为了满足 NUMA 分配的局部性, 每个 node 上还维护着所有其他 node 上的队列, 这样导致 SLAB 内部为维护这些队列就得花费大量的内存空间, 并且是O(n^2) 级别的. 这在大规模的 NUMA 机器上, 浪费的内存相当可观. 同时, 还有别的一些使用上的问题, 导致开发者对其不满, 因而引入了新的实现. 参见 [The SLUB allocator](https://lwn.net/Articles/229984).

SLUB 在解决了上述的问题之上, 提供与 SLAB 完全一样的接口, 所以用户可以无缝切换, 而且, 还提供了更好的调试支持. 早在几年前, 各大发行版中的对象分配器就已经切换为 SLUB 了.

关于性能, 前阵子为公司的系统切换 SLUB, 做过一些性能测试, 在一台两个 NUMA node, 32 个逻辑 CPU , 252 GB 内存的机器上, 在相同的 workload 测试下, SLUB 综合来说, 体现出了比 SLAB 更好的性能和吞吐量.



### 2.2.3 SLOB
-------

**2.6.16(2006年3月发布)**

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/11/03 | Matt Mackall <mpm@selenic.com> | [slob: introduce the SLOB allocator](https://lore.kernel.org/patchwork/cover/45623) | 实现 SLOB 分配器 | v2 ☑ 2.6.16-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/45623) |


这是第三个对象分配器, 提供同样的接口, 它是为适用于嵌入式小内存小机器的环境而引入的, 所以实现上很精简, 大大减小了内存 footprint, 能在小机器上提供很不错的性能.

### 2.2.4 SLQB
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2009/01/23 | Nick Piggin <npiggin@suse.de> | [SLQB slab allocator](https://lwn.net/Articles/311502) | 实现 SLQB 分配器 | v2 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/1385629)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/141837) |

### 2.2.5 改进与优化
-------

*   slab_merge

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2014/09/15 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [mm/slab_common: commonize slab merge logic](https://lore.kernel.org/patchwork/cover/493577) | 如果新创建的 SLAB 具有与现有 SLAB 相似的大小和属性, 该特性将重用它而不是创建一个新的. 这特性就是我们熟知的 `__kmem_cache_alias()`. SLUB 原生支持 merged. | v2 ☑ [3.18-rc1](https://kernelnewbies.org/Linux_3.18#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/cover/493577)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/499717) |
| 2017/06/20 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [mm: Allow slab_nomerge to be set at build time](https://lore.kernel.org/patchwork/cover/802038) | 新增 CONFIG_SLAB_MERGE_DEFAULT, 支持通过编译开关 slab_merge | v2 ☑ [3.18-rc1](https://kernelnewbies.org/Linux_3.18#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/cover/801532)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/802038) |
| 2021/03/29 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [mm/slab_common: provide "slab_merge" option for !IS_ENABLED(CONFIG_SLAB_MERGE_DEFAULT) builds](https://lore.kernel.org/patchwork/cover/1399240) | 如果编译未开启 CONFIG_SLAB_MERGE_DEFAULT, 可以通过 slab_merge 内核参数动态开启  | v2 ☐ | [PatchWork v1](https://lore.kernel.org/patchwork/cover/1399240)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/1399260) |


*   slab 抗碎片化

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/08/11 | Christoph Lameter <cl@linux-foundation.org> | [Slab Fragmentation Reduction V14](https://lore.kernel.org/patchwork/cover/125818) | SLAB 抗碎片化 | v14 ☐ | [PatchWork v5](https://lore.kernel.org/patchwork/cover/90742)<br>*-*-*-*-*-*-*-* <br>[PatchWork v14](https://lore.kernel.org/patchwork/cover/125818) |


*   kmalloc-reclaimable caches


内核的 dentry 缓存用于缓存文件系统查找的结果, 这些对象是可以立即被回收的. 但是使用 kmalloc() 完成的分配不能直接回收. 虽然这并不是一个大问题, 因为 dentry 收缩器可以回收这两块内存. 但是内核对真正可回收的内存的计算被这种分配模式打乱了. 内核经常会认为可回收的内存比实际的要少, 因此不必要地进入 OOM 状态.

[LSF/MM 2018](https://lwn.net/Articles/lsfmm2018) 讨论了一个有意思的话题: 可回收的 SLAB. 参见 [The slab and protected-memory allocators](https://lwn.net/Articles/753154), 这种可回收的 slab 用于分配那些可根据用户请求随时进行释放的内核对象, 比如内核的 dentry 缓存.

Roman Gushchin indirectly reclaimable memory](https://lore.kernel.org/patchwork/cover/922092) 可以认为是一种解决方案. 它创建一个新的计数器(nr_indirect_reclaimable)来跟踪可以通过收缩不同对象来释放的对象所使用的内存. 该特性于 [4.17-rc1](https://kernelnewbies.org/Linux_4.20#Memory_management) 合入主线.

不过, Vlastimil Babka 对补丁集并不完全满意. 计数器的名称迫使用户关注"间接"可回收内存, 这是他们不应该做的. Babka 认为更好的解决方案是为这些 kmalloc() 调用制作一套单独的[可回收 SLAB](https://lore.kernel.org/patchwork/cover/969264). 这将把可回收的对象放在一起, 从碎片化的角度来看, 这个方案是非常好的. 通过 GFP_RECLAIMABLE 标志, 可以从这些 slab 中分配内存. 经历了 v4 个版本后, 于 [4.20-rc1](https://kernelnewbies.org/Linux_4.20#Memory_management) 合入.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/03/05 | Roman Gushchin <guro@fb.com> | [indirectly reclaimable memory](https://lore.kernel.org/patchwork/cover/922092) | 引入间接回收内存的概念( `/proc/vmstat/nr_indirect _reclaimable`). 间接回收内存是任何类型的内存, 由内核使用(除了可回收的 slab), 这实际上是可回收的, 即将在内存压力下释放. 并[被当作 available 的内存对待](034ebf65c3c21d85b963d39f992258a64a85e3a9). | v1 ☑ [4.17-rc1](https://kernelnewbies.org/Linux_4.20#Memory_management) | [PatchWork](https://lore.kernel.org/patchwork/cover/922092)) |
| 2018/07/31 | Vlastimil Babka <vbabka@suse.cz> | [kmalloc-reclaimable caches](https://lore.kernel.org/patchwork/cover/969264) | 为 kmalloc 引入回收 SLAB, dcache external names 是 kmalloc-rcl-* 的第一个用户.  | v4 ☑ [4.20-rc1](https://kernelnewbies.org/Linux_4.20#Memory_management) | [PatchWork v4](https://lore.kernel.org/patchwork/cover/969264)) |


https://lore.kernel.org/patchwork/patch/76916
https://lore.kernel.org/patchwork/cover/78940
https://lore.kernel.org/patchwork/cover/72119
https://lore.kernel.org/patchwork/cover/63980
https://lore.kernel.org/patchwork/cover/91223
https://lore.kernel.org/patchwork/cover/145184
https://lore.kernel.org/patchwork/cover/668967



## 2.3 内核级别的 malloc 分配器之-大内存分配
-------

#### 2.3.1 VMALLOC 大内存分配器
-------

小内存的问题算是解决了, 但还有一个大内存的问题: 用伙伴系统分配 8 x 4KB 大小以上的的数据时, 只能从 16 x 4KB 的空闲列表里面去找(这样得到的物理内存是连续的), 但很有可能系统里面有内存, 但是伙伴系统分配不出来, 因为他们被分割成小的片段. 那么, vmalloc 就是要用这些碎片来拼凑出一个大内存, 相当于收集一些"边角料", 组装成一个成品后"出售".

2.6.28 时, Nick Piggin 重写了 vmalloc/vmap 分配器[mm: vmap rewrite](https://lwn.net/Articles/304188). 做了几项比较大的改动和优化.

1.  由于内核会频繁的进行 VMAP 区域的查找, 引入[红黑树表示的 vmap_area](https://elixir.bootlin.com/linux/v2.6.28/source/mm/vmalloc.c#L242) 来解决当查找数量非常多时效率低下的问题. 同时使用 vmap_area 的链表 vmap_area_list 替代原来的链表.

2.  由于要映射页表, 因此 TLB flush 就成了必需要做的操作. 这个过程在 vfree 流程中进行, 为了减少开销, 引入了 lazy 模式.

3.  整个 VMAP 子系统在一个全局读写锁 vmlist_lock 下工作. 虽然是 rwlock, 但是但它实际上是写在所有的快速路径上, 并且读锁可能永远不会并发运行, 所以这对于小型 VMAP 是毫无意义的. 因此实现了一个按 CPU 分配的分配器, 它可以平摊或避免全局锁定. 要使用 CPU 接口, 必须使用 vm_map_ram()/vm_unmap_ram() 接口来代替 vmap() 和 vunmap().  当前 vmalloc 目前没有使用这些接口, 所以它的可伸缩性不是很好(尽管它将使用惰性TLB刷新).



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| NA | Nick Piggin <npiggin@suse.de> | [Import 0.99.13k](https://elixir.bootlin.com/linux/0.99.13k/source/mm/vmalloc.c) | 引入 vmalloc 分配器. | ☑ 0.99.13k | [HISTORY commit](https://github.com/gatieme/linux-history/commit/537b6ff02ce317a747658a9f5dff96ed2733b28a) |
| 2002/08/06 | Nick Piggin <npiggin@suse.de> | [VM: Rework vmalloc code to support mapping of arbitray pages](https://github.com/gatieme/linux-history/commit/d24919a7fbc635bea6ecc267058dcdbadf03f565) | 引入 vmap/vunmap. 将 vmalloc 操作分为两部分: 分配支持物理地址不连续的页面(vmalloc)以及将这些页面映射到内核页表中, 以便进行实际的临时访问. 因此引入一组新的接口vmap/vunmap 允许将任意页映射到内核虚拟内存中. | ☑ 2.5.32~39 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/d24919a7fbc635bea6ecc267058dcdbadf03f565) |
| 2008/07/28 | Nick Piggin <npiggin@suse.de> | [mm: vmap rewrite](https://lwn.net/Articles/304188) | 重写 vmap 分配器 | RFC ☑ 2.6.28-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/118352))<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/patch/124065), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=db64fe02258f1507e13fe5212a989922323685ce) |
| 2009/02/18 | Tejun Heo <tj@kernel.org> | [implement new dynamic percpu allocator](https://lore.kernel.org/patchwork/cover/144750) | 实现了 [vm_area_register_early()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f0aa6617903648077dffe5cfcf7c4458f4610fa7) 以支持在启动阶段注册 vmap 区域. 基于此特性实现了可伸缩的动态 percpu 分配器(CONFIG_HAVE_DYNAMIC_PER_CPU_ARE), 可用于静态(pcpu_setup_static)和动态(percpu_modalloc) percpu 区域, 这将允许静态和动态区域共享更快的直接访问方法. | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/144750) |
| 2016/03/29 | Chris Wilson <chris@chris-wilson.co.uk> | [mm/vmap: Add a notifier for when we run out of vmap address space](https://lore.kernel.org/patchwork/cover/662338) | vmap是临时的内核映射, 可能持续时间很长. 对于驱动程序来说, 在对象上重用vmap是更好的选择, 因为在其他情况下, 设置vmap的成本可能会支配对象上的操作. 然而, 在32位系统上, vmap地址空间非常有限, 因此我们添加了一个vmap压力通知, 以便驱动程序释放任何缓存的 vmap 区域. 并为该通知链添加了首批用户. | v3 ☑ 4.7-rc1 | [PatchWork v3](https://lore.kernel.org/patchwork/cover/664579) |
| 2021/03/17 | Nicholas Piggin <npiggin@gmail.com> | [huge vmalloc mappings](https://lore.kernel.org/patchwork/cover/1397495) | vmalloc 支持透明大页 | v13 ☑ [5.13-rc1](https://kernelnewbies.org/Linux_5.13#Memory_management) | [PatchWork v13,00/14](https://patchwork.kernel.org/project/linux-mm/cover/20210317062402.533919-1-npiggin@gmail.com) |
| 2019/01/03 |  "Uladzislau Rezki (Sony)" <urezki@gmail.com> | [test driver to analyse vmalloc allocator](https://lore.kernel.org/patchwork/cover/1028793) | 实现一个驱动来帮助分析和测试 vmalloc | RFC v4 ☑ 5.1-rc1 | [PatchWork RFC v4](https://patchwork.kernel.org/project/linux-mm/cover/20190103142108.20744-1-urezki@gmail.com) |
| 2019/10/31 | Daniel Axtens <dja@axtens.net> | [kasan: support backing vmalloc space with real shadow memory](https://lore.kernel.org/patchwork/cover/1146684) | NA | v11 ☑ 5.5-rc1 | [PatchWork v11,0/4](https://lore.kernel.org/patchwork/cover/1146684) |
| 2021/03/24 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [vmalloc: Improve vmalloc(4MB) performance](https://lore.kernel.org/patchwork/cover/1401688) | 加速 4MB vmalloc 分配. | v2 ☑ 5.13-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/1401688) |
| 2006/04/21 | Nick Piggin <npiggin@suse.de> | [mm: introduce remap_vmalloc_range](https://lore.kernel.org/patchwork/cover/55978) | 添加 remap_vmalloc_range()、vmalloc_user() 和 vmalloc_32_user(), 这样驱动程序就可以有一个很好的接口来重新映射vmalloc内存. | v2 ☑ 2.6.18-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/55972)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/55978) |
| 2013/05/23 | Nick Piggin <npiggin@suse.de> | [kdump, vmcore: support mmap() on /proc/vmcore](https://lore.kernel.org/patchwork/cover/381217) | NA | v8 ☑ 3.11-rc1 | [PatchWork v8,0/9](https://lore.kernel.org/patchwork/cover/381217) |



#### 2.3.2 VMALLOC 中的 RBTREE 和 LIST 以及保护它们的锁
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/07/28 | Nick Piggin <npiggin@suse.de> | [mm: vmap rewrite](https://lwn.net/Articles/304188) | 重写 vmap 分配器, 为 vmap_area 首次引入了红黑树组织(使用 vmap_area_lock 来保护). | RFC ☑ 2.6.28-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/118352))<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/patch/124065), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=db64fe02258f1507e13fe5212a989922323685ce) |
| 2011/03/22 | Nick Piggin <npiggin@suse.de> | [mm: vmap area cache](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89699605fe7cfd8611900346f61cb6cbf179b10a) | 为 vmalloc 分配器提供一个空闲区域缓存 free_vmap_cache, 它缓存了上次搜索和分配的空闲区域的节点, 下次分配和释放时, 都将从这里开始搜索, 避免了每次从 VMALLOC_START 搜索. 之前的实现中, 如果 vmalloc/vfree 频繁, 将从 VMALLOC_START 地址建立一个区段链, 每次都必须对其进行迭代, 这将是 O(n) 的开销. 在这个补丁之后, 搜索将从它结束的地方开始, 给出更接近于 O(1) 的平摊结果. 这一定程度上减少了rbtree操作的数量. 这解决了在 [db64fe02 mm: rewrite vmap layer](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=db64fe02258f1507e13fe5212a989922323685ce) 中引入的惰性 vunmap TLB 清除导致的回归. 在 vunmapped 区段之后, 该补丁将在 vmap 分配器中保留区段, 直到可以在单个批处理中刷新的大量区段积累为止.  | v2 ☑ 3.10-rc1 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89699605fe7cfd8611900346f61cb6cbf179b10a) |
| 2013/03/13 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [remove vm_struct list management](https://lore.kernel.org/patchwork/cover/365284) | 在 vmalloc 初始化后, 不再使用旧的 vmlist 链表管理 vm_struct. 而是直接使用由 vmap_area 组织成的红黑树 vmap_area_root和链表 vmap_area_list 管理. 这个补丁之后 vmlist 被标记为 `__initdata`, 且只在 vmalloc_init 之前, 通过  vm_area_register_early-=>vm_area_add_early 来使用, 比如[注册 percpu 区域](https://elixir.bootlin.com/linux/v3.10/source/mm/percpu.c#L1785). 注意: 很多网络的博客上说, vm_struct 通过 next 字段组成链表, 这个链表之后, 这个说法在这个补丁之后已经不严谨了. | v2 ☑ 3.10-rc1 | [PatchWork v2,0/8](https://lore.kernel.org/patchwork/cover/365284)<br>*-*-*-*-*-*-*-* <br>[关注 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4341fa454796b8a37efd5db98112524e85e7114e) |
| 2016/04/15 | "Uladzislau Rezki (Sony)" <urezki@gmail.com> | [mm/vmalloc: Keep a separate lazy-free list](https://lore.kernel.org/patchwork/cover/669083) | 将惰性释放的 vmap_area 添加到一个单独的无锁空闲 vmap_purge_list, 这样我们就不必在每次清除时遍历整个列表.  | v2 ☑ 4.7-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/cover/667471)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/669083)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=80c4bd7a5e4368b680e0aeb57050a1b06eb573d8) |
| 2019/04/06 |  "Uladzislau Rezki (Sony)" <urezki@gmail.com> | [improve vmap allocation](https://lore.kernel.org/patchwork/cover/1059021) | [improve vmalloc allocation](https://patchwork.kernel.org/project/linux-mm/cover/20181019173538.590-1-urezki@gmail.com) 的 RESEND 和再版. 优化 alloc_vmap_area(), 优化查找复杂度从 O(N) 下降为 O(logN). 目前新的 VA 区域的分配是在繁忙列表(vmap_area_list/root)迭代中完成的, 直到在两个繁忙区域之间找到一个合适的空洞. 因此, 每次新的分配都会导致列表增长. 查找空洞的代价为 O(N). 而通过跟踪空闲的 vmap_area 区域, 并在空闲列表搜索空洞来完成分配, 可以将分配的开销降到 O(logN). 在初始化阶段 vmalloc_init() 中, vmalloc 内存布局将[对空闲区域进行跟踪和组织](https://elixir.bootlin.com/linux/v5.2/source/mm/vmalloc.c#L1880), 并用链表 free_vmap_area_list 和 红黑书 free_vmap_area_root 组织. 为了优化查找效率, vmap_area 中 subtree_max_size 存储了其节点为根的红黑树中最大的空洞大小. | v3 ☑ [5.2-rc7](https://kernelnewbies.org/Linux_5.2#Memory_management) | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/1002038)<br>*-*-*-*-*-*-*-* <br>[PatchWork v4](https://patchwork.kernel.org/project/linux-mm/patch/20190406183508.25273-2-urezki@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v3](https://lore.kernel.org/patchwork/cover/1057418) |
| 2019/07/06 | Pengfei Li <lpf.vector@gmail.com> | [mm/vmalloc.c: improve readability and rewrite vmap_area](https://lore.kernel.org/patchwork/cover/1101062) | 为了减少结构体 vmap_area 的大小.<br>1. "忙碌"树可能相当大, 即使区域被释放或未映射, 它仍然停留在那里, 直到"清除"逻辑删除它. 优化和减少"忙碌"树的大小, 在用户触发空闲路径时立即删除一个节点. 这样做是可能的, 因为分配是使用另一个扩展树完成的;这样忙碌的树将只包含分配的区域, 不会干扰惰性空闲的节点, 引入新的函数 show_purge_info(), 转储通过"/proc/vmallocinfo" 来显示 "unpurged" 区域.<br>2. 由于结构体 vmap_area 的成员不是同时使用的, 所以可以通过将几个不同时使用的成员放入联合中来减少其大小. | v6 ☑ 5.4-rc1 | [PatchWork v1,0/5](https://lore.kernel.org/patchwork/cover/1095787)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/5](https://lore.kernel.org/patchwork/cover/1096583)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6,0/2](https://patchwork.kernel.org/project/linux-mm/cover/20190716152656.12255-1-lpf.vector@gmail.com) |
| 2020/11/16 | "Uladzislau Rezki (Sony)" <urezki@gmail.com> | [mm/vmalloc: rework the drain logic](https://lore.kernel.org/patchwork/cover/1339347) | 当前的"惰性释放”模式至少存在两个问题: <br>1. 由于只有 vmap_purge_list 链表组织, 因此 vmap 区域的未排序的, 因此为了识别要耗尽的区域的 [min:max] 范围, 它需要扫描整个 vmap_purge_list 链表. 可能会耗费较多时间.<br>2. 作为下一步是关于合并所有片段与自由空间.这也是一种耗时的操作, 因为它必须遍历包含突出惰性区域的整个列表. 这造成了极高的延迟. | v1 ☑ 5.11-rc1 | [PatchWork](https://patchwork.kernel.org/project/linux-mm/patch/20201116220033.1837-2-urezki@gmail.com) |
| 2019/10/22 | Uladzislau Rezki (Sony) <urezki@gmail.com> | [mm/vmalloc: rework vmap_area_lock](https://lore.kernel.org/patchwork/cover/1143029) | 随着 5.2 版本 [improve vmap allocation](https://lore.kernel.org/patchwork/cover/1059021) 的合入, 全局的 vmap_area_lock 可以被拆分成两个锁: 一个用于分配部分(vmap_area_lock), 另一个用于回收(free_vmap_area_lock), 因为有两个不同的实体: "空闲数据结构"和"繁忙数据结构". 这和那后的减少锁争用, 允许在不同的 CPU 上并行执行"空闲"和"忙碌"树的操作. 但是需要注意的是, 分配/释放操作仍然存在依赖. 如果在不同的 CPU 上同时运行, 分配/释放操作仍然会相互干扰. | v1 ☑ 5.5-rc1 | [PatchWork v2,0/8](https://patchwork.kernel.org/project/linux-mm/patch/20191022155800.20468-1-urezki@gmail.com)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e36176be1c3920a487681e37158849b9f50189c4) |




#### 2.3.1.3 lazy drain
-------


vmalloc 分配的内存并不是内核预先分配好的, 而是需要动态分配的, 因此需要进行 TLB flush. 在 vmalloc 分配的内存释放(vfree)时会做 TLB flush. 这是一个全局内核 TLB flush, 在大多数架构上, 它是一个广播 IPI 到所有 CPU 来刷新缓存. 这都是在全局锁下完成的, 随着 CPU 数量的增加, 扩展后的工作负载需要执行的 vunmap() 数量也会增加, 全局 TLB 刷新的成本也会增加. 这导致了糟糕的二次可伸缩性问题.

2.6.28 时, Nick Piggin 重写了 vmalloc/vmap 分配器[mm: vmap rewrite](https://lwn.net/Articles/304188). 其中为了
为提升 TLB flush 效率. vfree 流程中, 对于 TLB flush 操作, 采用 lazy 模式, 即: 先收集, 不真正释放, 当达到[限制(lazy_max_pages)时](https://elixir.bootlin.com/linux/v2.6.28/source/mm/vmalloc.c#L552), 再一起释放.


并为小型 vmap 提供快速, 可伸缩的 percpu 前端.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/07/28 | Nick Piggin <npiggin@suse.de> | [mm: vmap rewrite](https://lwn.net/Articles/304188) | 重写 vmap 分配器. 引入减轻 TLB flush 的影响, 引入了 lazy 模式. | RFC ☑ 2.6.28-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/118352)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/patch/124065), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=db64fe02258f1507e13fe5212a989922323685ce) |
| 2016/04/15 | "Uladzislau Rezki (Sony)" <urezki@gmail.com> | [mm/vmalloc: Keep a separate lazy-free list](https://lore.kernel.org/patchwork/cover/669083) | 之前惰性释放的 vmap_area 也是放到 vmap_area_list 中, 但是使用 LAZY_FREE 标记. 每次处理时, 需要遍历整个 vmap_area_list, 然后将 LAZY_FREE 的节点添加到一个临时的 purge_list 中进行释放. 当混合使用大量 vmalloc 和 set_memory_*()(它调用vm_unmap_aliases())时, 触发了由于每次调用都要遍历整个 vmap_area_list 而导致性能严重下降的情况. 因此将惰性释放的 vmap_area 添加到一个单独的无锁空闲列表, 这样我们就不必在每次清除时遍历整个列表.  | v2 ☑ 4.7-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/cover/667471)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/669083)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=80c4bd7a5e4368b680e0aeb57050a1b06eb573d8) |
| 2018/08/23 | "Uladzislau Rezki (Sony)" <urezki@gmail.com> | [minor mmu_gather patches](https://lore.kernel.org/patchwork/cover/976960) | NA | v1 ☑ 5.11-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/976960) |

#### 2.3.1.4 per cpu vmap block
-------

[mm: vmap rewrite](https://lwn.net/Articles/304188) 新增了 per-cpu vmap block 缓存. 通过 vm_map_ram()/vm_unmap_ram() 显式调用. 当分配的空闲小于 VMAP_MAX_ALLOC 的时候, 将使用 per-cpu 缓存的 vmap block 来进行 vmap 映射.

vmap_block 是内核每次预先分配的 per-cpu vmap 缓存块, vmap_block_queue 链表维护了所有空闲 vmap_block 缓存.
每次分配的时候, 遍历 vmap_block_queue 查找到第一块满足要求(大小不小于待分配大小)的 vmap_block 块, 进行分配. 当链表中无空闲块或者所有的空闲块都无法满足当前分配需求的时候, 将通过 new_vmap_block 再缓存一块空闲块.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/07/28 | Nick Piggin <npiggin@suse.de> | [mm: vmap rewrite](https://lwn.net/Articles/304188) | 重写 vmap 分配器. 引入了 percpu 的 vmap_blocke_queue 来加速小于 VMAP_MAX_ALLOC 大小的 vmap 区域分配. 必须使用新增接口 vm_map_ram()/vm_unmap_ram() 来调用. | RFC ☑ 2.6.28-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/118352)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/patch/124065), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=db64fe02258f1507e13fe5212a989922323685ce) |
| 2009/01/07 | MinChan Kim <minchan.kim@gmail.com> | [Remove needless lock and list in vmap](https://lore.kernel.org/patchwork/cover/139998) | vmap 的 dirty_list 未使用. 这是为了优化 TLB flush. 但尼克还没写代码. 所以, 我们在需要的时候才需要它. 这个补丁删除了 vmap_block 的 dirty_list 和相关代码. | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/139998)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d086817dc0d42f1be8db4138233d33e1dd16a956) |
| 2010/02/01 | Nick Piggin <npiggin@suse.de> | [mm: purge fragmented percpu vmap blocks](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=02b709df817c0db174f249cc59e5f7fd01b64d92) | 引入 purge_fragmented_blocks_allcpus() 来释放 percpu 的 vmap block 缓存(借助了 lazy 模式). 在此之前, 内核不会释放 per-cpu 映射, 直到它的所有地址都被使用和释放. 所以碎片块可以填满 vmalloc 空间, 即使它们实际上内部没有活动的 vmap 区域. 这解决了 Christoph 报告的在 XFS 中使用 percpu vmap api 时出现的一些 vmap 分配失败的问题. | v1 ☑ 2.6.33-rc7 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=02b709df817c0db174f249cc59e5f7fd01b64d92) |
| 2020/08/06 | Matthew Wilcox (Oracle) <willy@infradead.org> | [vmalloc: convert to XArray](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0f14599c607d32512a1d37e6d2a2d1a867f16177) | 把 vmap_blocks 组织结构从 radix 切换到 XArray. | v1 ☑ 5.9-rc1 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0f14599c607d32512a1d37e6d2a2d1a867f16177) |
| 2008/07/28 | Nick Piggin <npiggin@suse.de> | [mm/vmalloc: fix possible exhaustion of vmalloc space](https://lore.kernel.org/patchwork/patch/553114) | 修复 vm_map_ram 分配器引发的高度碎片化: vmap_block 有空闲空间, 但仍然会出现新的块. 频繁的使用 vm_map_ram()/vm_unmap_ram() 去映射/解映射会非常快的耗尽 vmalloc 空间. 在小型 32 位系统上, 这不是一个大问题, 在第一次分配失败(alloc_vmap_area)时将很快调用 cause 清除, 但在 64 位机器上, 例如 x86_64 有 45 位 vmalloc 空间, 这可能是一场灾难.  | RFC ☑ 2.6.28-rc1 | [PatchWork RFC,0/3](https://lore.kernel.org/patchwork/cover/550979)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v2,0/3](https://lore.kernel.org/patchwork/patch/553114) |
| 2008/07/28 | Nick Piggin <npiggin@suse.de> | [mm, vmalloc: cleanup for vmap block](https://lore.kernel.org/patchwork/patch/384995) | 删除了 vmap block 中的一些死代码和不用代码. 其中 vb_alloc() 中 vmap_block 中如果有足够空间是必然分配成功的, 因此删除了 bitmap_find_free_region() 相关的 purge 代码. | RFC ☑ 2.6.28-rc1 | [PatchWork 0/3](https://lore.kernel.org/patchwork/cover/384995)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3fcd76e8028e0be37b02a2002b4f56755daeda06) |


https://lore.kernel.org/patchwork/cover/616606/
https://lore.kernel.org/patchwork/patch/164572/

commit f48d97f340cbb0c323fa7a7b36bd76a108a9f49f
Author: Joonsoo Kim <iamjoonsoo.kim@lge.com>
Date:   Thu Mar 17 14:17:49 2016 -0700

    mm/vmalloc: query dynamic DEBUG_PAGEALLOC setting



#### 2.3.1.5 other vmalloc interface
-------

vread 用于读取指定地址的内存数据.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 1993/12/28 | Linus Torvalds <torvalds@linuxfoundation.org> | [Linux-0.99.14 (November 28, 1993)](https://elixir.bootlin.com/linux/0.99.14/source/mm/vmalloc.c#L168) | 引入 vread. | ☑ 0.99.14 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/7e8425884852b83354ab090a07715c6c32918f37) |


vwrite 则用于对指定的 vmalloc 区域进行写操作.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| NA | Marc Boucher | [Support /dev/kmem access to vmalloc space (Marc Boucher)](https://elixir.bootlin.com/linux/2.4.17/source/mm/vmalloc.c#L168) | 引入 vwrite | ☑ 2.4.17 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/a3245879f664fb42b1903bc98af670da6d783db5) |
| 2021/05/06 | David Hildenbrand <david@redhat.com> | [mm/vmalloc: remove vwrite()](https://lore.kernel.org/patchwork/cover/1401594) | 删除 vwrite | v1 ☑ 5.13-rc1 | [PatchWork v1,3/3](https://lore.kernel.org/patchwork/cover/1401594)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f7c8ce44ebb113b83135ada6e496db33d8a535e3) |

vmalloc_to_page 则提供了通过 vmalloc 地址查找到对应 page 的操作.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/02/19 | Ingo Molnar <mingo@elte.hu> | [vmalloc_to_page helper](https://github.com/gatieme/linux-history/commit/094686d30d5f58780a1efda2ade5cb0d18e25f82) | NA | ☑ 3.11-rc1 | [commit1 HISTORY](https://github.com/gatieme/linux-history/commit/094686d30d5f58780a1efda2ade5cb0d18e25f82), [commit2 HISTORY](https://github.com/gatieme/linux-history/commit/e1f40fc0cf09f591b474a0c17fc4e5ca7438c7dd) |

对于 vmap_pfn

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/05/04 | Jeremy Fitzhardinge <jeremy@goop.org> | [xen: Xen implementation for paravirt_ops](https://lore.kernel.org/patchwork/patch/80500) | NA | ☑ 2.6.22-rc1 | [PatchWork 0/11](https://lore.kernel.org/patchwork/cover/80500), [关注 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aee16b3cee2746880e40945a9b5bff4f309cfbc4) |
| 2020/10/02 | Christoph Hellwig <hch@lst.de> | [mm: remove alloc_vm_area and add a vmap_pfn function](https://lore.kernel.org/patchwork/patch/1316291) | NA | ☑ 5.10-rc1 | [PatchWork 0/11](https://lore.kernel.org/patchwork/cover/1316291) |


#### 2.3.1.6 huge vmap
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/03/03 | Toshi Kani <toshi.kani@hp.com> | [Kernel huge I/O mapping support](https://lore.kernel.org/patchwork/cover/547056) | ioremap() 支持透明大页. 扩展了 ioremap() 接口, 尽可能透明地创建具有大页面的 I/O 映射. 当一个大页面不能满足请求范围时, ioremap() 继续使用 4KB 的普通页面映射. 使用 ioremap() 不需要改变驱动程序. 但是, 为了使用巨大的页面映射, 请求的物理地址必须以巨面大小(x86上为 2MB 或 1GB)对齐. 内核巨页的 I/O 映射将提高 NVME 和其他具有大内存的设备的性能, 并减少创建它们映射的时间. | v3 ☑ 4.1-rc1 | [PatchWork v3,0/6](https://lore.kernel.org/patchwork/cover/547056) |


### 2.3.2 连续内存分配器(CMA)
-------

**3.5(2012年7月发布)**

[Linux中的Memory Compaction [二] - CMA](https://zhuanlan.zhihu.com/p/105745299)

[Contiguous memory allocation for drivers](https://lwn.net/Articles/396702)

[A deep dive into CMA](https://lwn.net/Articles/486301)

[CMA and compaction](https://lwn.net/Articles/684611)

[A reworked contiguous memory allocator](https://lwn.net/Articles/447405)

顾名思义,这是一个分配连续物理内存页面的分配器. 也许你会疑惑伙伴分配器不是也能分配连续物理页面吗? 诚然, 但是一个系统在运行若干时间后, 可能很难再找到一片足够大的连续内存了, 伙伴系统在这种情况下会分配失败. 但连续物理内存的分配需求是刚需: 一些比较低端的 DMA 设备只能访问连续的物理内存; 还有下面会讲的透明大页的支持, 也需要连续的物理内存.

一个解决办法就是在系统启动时,在内存还很充足的时候, 先预留一部分连续物理内存页面, 留作后用. 但这有个代价, 这部分内存就无法被作其他使用了, 为了可能的分配需求, 预留这么一大块内存, 不是一个明智的方法.

CMA 的做法也是启动时预留, 但不同的是, 它允许这部分内存被正常使用, 在有连续内存分配需求时, 把这部分内存里的页面迁移走, 从而空出位置来作分配.

因此 DMA 设备对 CMA 区域有优先使用权, 被称为 primary client, 而其他模块的页面则是 secondary client.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/02/28 | Mel Gorman | [Introduce ZONE_CMA](https://lore.kernel.org/patchwork/cover/778794) | ZONE_CMA 的实现方案, 使用新的分区不仅可以有H/W 寻址限制, 还可以有 S/W 限制来保证页面迁移. | v7 ☐ | [PatchWork v7](https://lore.kernel.org/patchwork/cover/778794) |
| 2007/02/28 | Mel Gorman | [mm/cma: manage the memory of the CMA area by using the ZONE_MOVABLE](https://lore.kernel.org/patchwork/cover/857428) | 新增了 ZONE_CMA 区域, 使用新的分区不仅可以有H/W 寻址限制, 还可以有 S/W 限制来保证页面迁移.  | v2 ☐ | [PatchWork v7](https://lore.kernel.org/patchwork/cover/857428) |
| 2010/11/19 | Mel Gorman | [big chunk memory allocator v4](https://lore.kernel.org/patchwork/cover/224757) | 大块内存分配器 | v4 ☐ | [PatchWork v4](https://lore.kernel.org/patchwork/cover/224757) |
| 2012/04/03 | Michal Nazarewicz <m.nazarewicz@samsung.com><br>*-*-*-*-*-*-*-* <br>Marek Szyprowski <m.szyprowski@samsung.com> | [Contiguous Memory Allocator](https://lwn.net/Articles/486301) | 实现 CMA | v24 ☑ [3.5-rc1](https://kernelnewbies.org/Linux_3.5#Memory_Management) | [PatchWork v7](https://lore.kernel.org/patchwork/cover/229177)<br>*-*-*-*-*-*-*-* <br>[PatchWork v24](https://lore.kernel.org/patchwork/cover/295656) |
| 2015/02/12 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [mm/compaction: enhance compaction finish condition](https://lore.kernel.org/patchwork/patch/542063) | 同样的, 之前 NULL 指针和错误指针的输出也很混乱, 进行了归一化. | v1 ☑ 4.1-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/542063)<br>*-*-*-*-*-*-*-* <br>[关键 commit 2149cdaef6c0](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2149cdaef6c0eb59a9edf3b152027392cd66b41f) |
| 2015/02/23 | SeongJae Park <sj38.park@gmail.com> | [introduce gcma](https://lore.kernel.org/patchwork/patch/544555) | [GCMA(Guaranteed Contiguous Memory Allocator)方案](http://ceur-ws.org/Vol-1464/ewili15_12.pdf), 倾向于使用 writeback 的page cache 和 完成 swap out 的 anonymous pages 来做 seconday client, 进行迁移. 从而确保 primary client 的分配. | RFC v2 ☐ | [PatchWork](https://lore.kernel.org/patchwork/cover/544555), [GitHub](https://github.com/sjp38/linux.gcma/releases/tag/gcma/rfc/v2) |



# 3 内存去碎片化
-------

## 3.1 关于碎片化
-------

内存按 chunk 分配, 每个程序保留的 chunk 的大小和时间都不同. 一个程序可以多次请求和释放 `memory chunk`. 程序一开始时, 空闲内存有很多并且连续, 随后大的连续的内存区域碎片化, 变成更小的连续区域, 最终程序无法获取大的连续的 memory chunk.


| 碎片化类型 | 描述 | 示例 |
|:--------:|:----:|:---:|
| 内碎片化(Internal fragmentation) | 分给程序的内存比它实际需要的多, 多分的内存被浪费. | 比如chunk一般是4, 8或16的倍数, 请求23字节的程序实际可以获得24字节的chunk, 未被使用的内存无法再被分配, 这种分配叫fixed partitions. 一个程序无论多么小, 都要占据一个完整的partition. 通常最好的解决方法是改变设计, 比如使用动态内存分配, 把内存空间的开销分散到大量的objects上, 内存池可以大大减少internal fragmentation. |
| 外部碎片(External fragmentation) | 有足够的空闲内存, 但是没有足够的连续空闲内存供分配. | 因为都被分成了很小的pieces, 每个piece都不足以满足程序的要求. external指未使用的存储空间在已分配的区域外. 这种情况经常发生在频繁创建、更改(大小)、删除不同大小文件的文件系统中. 比起文件系统, 这种fragmentation在RAM上更是一个问题, 因为程序通常请求RAM分配一些连续的blocks, 而文件系统可以利用可用的blocks并使得文件逻辑上看上去是连续的. 所以对于文件系统来说, 有空闲空间就可以放新文件, 碎片化也没关系, 对内存来说, 程序请求连续blocks可能无法满足, 除非程序重新发出请求, 请求一些更小的分散的blocks. 解决方法一是compaction, 把所有已分配的内存blocks移到一块, 但是比较慢, 二是garbage collection, 收集所有无法访问的内存并把它们当作空闲内存, 三是paging, 把物理内存分成固定大小的frames, 用相同大小的逻辑内存pages填充, 逻辑地址不连续, 只要有可用内存, 进程就可以获得, 但是paging又会造成internal fragmentation. |
| Data fragmentation. | 当内存中的一组数据分解为不连续且彼此不紧密的许多碎片时, 会发生这种类型的碎片. 如果我们试图将一个大对象插入已经遭受的内存中, 则会发生外部碎片 .  | 通常发生在向一个已有external fragmentation的存储系统中插入一个大的object的时候, 操作系统找不到大的连续的内存区域, 就把一个文件不同的blocks分散放置, 放到可用的小的内存pieces中, 这样文件的物理存放就不连续, 读写就慢了, 这叫文件系统碎片. 消除碎片工具的主要工作就是重排block, 让每个文件的blocks都相邻. |

参考 [Fragmentation in Operating System](https://www.includehelp.com/operating-systems/fragmentation.aspx)



前面讲了运行较长时间的系统存在的内存碎片化问题, Linux 内核也不能幸免, 因此有开发者陆续提出若干种方法.



## 3.2 成块回收(Lumpy Reclaim)
-------


**2.6.23引入(2007年7月), 3.5移除(2012年7月)**

这不是一个完整的解决方案, 它只是缓解这一问题. 所谓回收是指 MM 在分配内存遇到内存紧张时, 会把一部分内存页面回收. 而[成块回收](https://lwn.net/Articles/211199), 就是尝试成块回收目标回收页相邻的页面, 以形成一块满足需求的高阶连续页块. 这种方法有其局限性, 就是成块回收时没有考虑被连带回收的页面可能是"热页", 即被高强度使用的页, 这对系统性能是损伤.



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/04/20 | Andy Whitcroft <apw@shadowen.org> | [Lumpy Reclaim V6](https://lore.kernel.org/patchwork/cover/78996) | 实现成块回收, 其中引入了 PAGE_ALLOC_COSTLY_ORDER, 该值虽然是经验值, 但是通常被认为是介于系统有/无回收页面压力的一个临界值, 一次分配低于这个 order 的页面, 通常是容易满足的. 而大于这个 order 的页面, 被认为是 costly 的. | v6 ☑ 2.6.23-rc1 | [Patchwork V5](https://lore.kernel.org/patchwork/cover/76206)<br>*-*-*-*-*-*-*-* <br>[PatchWork v8](https://lore.kernel.org/patchwork/cover/78996)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6 cleanup](https://lore.kernel.org/patchwork/cover/79316)<br>*-*-*-*-*-*-*-* <br>[COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5ad333eb66ff1e52a87639822ae088577669dcf9) |
| 2008/07/01 | Mel Gorman <mel@csn.ul.ie> | [Reclaim page capture v1](https://lore.kernel.org/patchwork/cover/121203) | 大 order 页面分配的有一次探索. 这种方法是捕获直接回收中释放的页面, 以便在空闲页面被竞争的分配程序重新分配之前增加它们被压缩的机会. | v1 ☐  | [Patchwork V5](https://lore.kernel.org/patchwork/cover/121203) |
| 2007/08/02 | Andy Whitcroft <apw@shadowen.org> | [Synchronous Lumpy Reclaim V3](https://lore.kernel.org/patchwork/cover/87667) | 当以较高的阶数应用回收时, 可能会启动大量IO. 这组补丁尝试修复这个问题, 用于在 VM 事件记录器中中将页面标记为非活动时修复, 并在直接回收连续区域时等待页面写回. | v3 ☑ 2.6.23-rc4 | [Patchwork V5](https://lore.kernel.org/patchwork/cover/87667) |


## 3.3 通过迁移类型分组来实现反碎片
-------

也称为: 基于页面可移动性的页面聚类(Page Clustering by Page Mobility)

为什么要引入迁移类型. 我们都知道伙伴系统是针对于解决外碎片的问题而提出的, 那么为什么还要引入这样一个概念来避免碎片呢?


在去碎片化时, 需要移动或回收页面, 以腾出连续的物理页面, 但可能一颗“老鼠屎就坏了整锅粥”——由于某个页面无法移动或回收, 导致整个区域无法组成一个足够大的连续页面块. 这种页面通常是内核使用的页面, 因为内核使用的页面的地址是直接映射(即物理地址加个偏移就映射到内核空间中), 这种做法不用经过页表翻译, 提高了效率, 却也在此时成了拦路虎.

我们注意到, 碎片一般是指散布在内存中的小块内存, 由于它们已经被分配并且插入在大块内存中, 而导致无法分配大块的连续内存. 而伙伴系统把内存分配出去后, 要再回收回来并且重新组成大块内存, 这样一个过程必须建立两个伙伴块必须都是空闲的这样一个基础之上, 如果其中有一个伙伴不是空闲的, 甚至是其中的一个页不是空闲的, 都会导致无法分配一块连续的大块内存. 我们引用一个例子来看这个问题:

![碎片化](1338638697_6064.png)

图中, 如果15对应的页是空闲的, 那么伙伴系统可以分配出连续的16个页框, 而由于15这一个页框被分配出去了, 导致最多只能分配出8个连续的页框. 假如这个页还会被回收回伙伴系统, 那么至少在这段时间内产生了碎片, 而如果更糟的, 如果这个页用来存储一些内核永久性的数据而不会被回收回来, 那么碎片将永远无法消除, 这意味着15这个页所处的最大内存块永远无法被连续的分配出去了. 假如上图中被分配出去的页都是不可移动的页, 那么就可以拿出一段内存, 专门用于分配不可移动页, 虽然在这段内存中有碎片, 但是避免了碎片散布到其他类型的内存中. 在系统中所有的内存都被标识为可移动的！也就是说一开始其他类型都没有属于自己的内存, 而当要分配这些类型的内存时, 就要从可移动类型的内存中夺取一部分过来, 这样可以根据实际情况来分配其他类型的内存.

长年致力于解决内存碎片化的内存领域黑客 Mel Gorman 观察到这个事实, 在经过28个版本的修改后, 他的解决方案 [Group pages of related mobility together to reduce external fragmentation v28](https://lore.kernel.org/patchwork/cover/75208) 进入内核.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/03/01 | Mel Gorman <mel@csn.ul.ie> | [Group pages of related mobility together to reduce external fragmentation v28](https://lore.kernel.org/patchwork/cover/75208) | 基于页面可移动性的页面聚类 | v28 ☑ 2.6.24-rc1 | [Patchwork v28,0/12](https://lore.kernel.org/patchwork/cover/75208) |

Mel Gorman 观察到, 所有使用的内存页有三种情形:

| 编号 | 类型 | 描述 |
|:---:|:---:|:----:|
| 1 | 容易回收的(easily reclaimable) | 这种页面可以在系统需要时回收, 比如文件缓存页, 们可以轻易的丢弃掉而不会有问题(有需要时再从后备文件系统中读取); 又比如一些生命周期短的内核使用的页, 如DMA缓存区. |
| 2 | 难回收的(non-reclaimable) | 这种页面得内核主动释放, 很难回收, 内核使用的很多内存页就归为此类, 比如为模块分配的区域, 比如一些常驻内存的重要内核结构所占的页面. |
| 3 | 可移动的(movable) | 用户空间分配的页面都属于这种类型, 因为用户态的页地址是由页表翻译的, 移动页后只要修改页表映射就可以(这也从另一面应证了内核态的页为什么不能移动, 因为它们采取直接映射). |

因此, 他修改了伙伴分配器和分配 API, 使得在分配时告知伙伴分配器页面的可移动性: 回收时, 把相同移动性的页面聚类; 分配时, 根据移动性, 从相应的聚类中分配.

聚类的好处是, 结合上述的**成块回收**方案, 回收页面时, 就能保证回收同一类型的; 或者在迁移页面时(migrate page), 就能移动可移动类型的页面, 从而腾出连续的页面块, 以满足高阶的连续物理页面分配.

关于细节, 可看我之前写的文章:[Linux内核中避免内存碎片的方法(1)](https://link.zhihu.com/?target=http%3A//larmbr.com/2014/04/08/avoiding-memory-fragmentation-in-linux-kernelp%281%29)


## 3.4 内存规整(Memory Compaction)
-------

### 3.4.1 内存规整
-------

**2.6.35(2010年8月发布)**

2.2中讲到页面迁移类型(聚类), 它把相当可移动性的页面聚集在一起: 可移动的在一起, 可回收的在一起, 不可移动的也在一起. **它作为去碎片化的基础.** 然后, 利用**成块回收**, 在回收时, 把可回收的一起回收, 把可移动的一起移动, 从而能空出大量连续物理页面. 这个**作为去碎片化的策略.**



2.6.35 里, Mel Gorman 又实现了一种新的**去碎片化的策略** 叫[**内存紧致化或者内存规整**](https://lwn.net/Articles/368869). 不同于**成块回收**回收相临页面, **内存规整**则是更彻底, 它在回收页面时被触发, 它会在一个 zone 里扫描, 把已分配的页记录下来, 然后把所有这些页移动到 zone 的一端, 这样这把一个可能已经七零八落的 zone 给紧致化成一段完全未分配的区间和一段已经分配的区间, 这样就又腾出大块连续的物理页面了.

它后来替代了成块回收, 使得后者在 3.5 中被移除.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2010/04/20 | Mel Gorman <mel@csn.ul.ie> | [Memory Compaction](https://lwn.net/Articles/368869) | 内存规整 | v8 ☑ 2.6.35-rc1 | [PatchWork v8](https://lore.kernel.org/patchwork/cover/196771), [LWN](https://lwn.net/Articles/368869) |
| 2010/11/22 | Mel Gorman <mel@csn.ul.ie> | [Use memory compaction instead of lumpy reclaim during high-order allocations V2](https://lore.kernel.org/patchwork/cover/196771) | 在分配大内存时, 不再使用成块回收(lumpy reclaim)策略, 而是使用内存规整(memory compaction) | v8 ☑ 2.6.35-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/196771) |
| 2011/02/25 | Mel Gorman <mel@csn.ul.ie> | [Reduce the amount of time compaction disables IRQs for V2](https://lore.kernel.org/patchwork/cover/238585) | 减少内存规整关中断的时间, 降低其开销. | v2 ☑ 2.6.39-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/238585) |
| 2012/04/11 | Mel Gorman <mel@csn.ul.ie> | [Removal of lumpy reclaim V2](https://lore.kernel.org/patchwork/cover/296609) | 移除成块回收(lumpy reclaim) 的代码. | v2 ☑ [3.5-rc1](https://kernelnewbies.org/Linux_3.5#Memory_Management) | [PatchWork v2](https://lore.kernel.org/patchwork/cover/296609) |
| 2012/09/21 | Mel Gorman <mel@csn.ul.ie> | [Reduce compaction scanning and lock contention](https://lore.kernel.org/patchwork/cover/327667) | 进一步优化内存规整的扫描耗时和锁开销. | v1 ☑ 3.7-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/cover/327667) |
| 2013/12/05 | Mel Gorman <mel@csn.ul.ie> | [Removal of lumpy reclaim V2](https://lore.kernel.org/patchwork/cover/296609) | 添加了 start 和 end 两个 tracepoint, 用于内存规整的开始和结束. 通过这两个 tracepoint 可以计算工作负载在规整过程中花费了多少时间, 并可能调试与用于扫描的缓存 pfns 相关的问题. 结合直接回收和 slab 跟踪点, 应该可以估计工作负载的大部分与分配相关的开销. | v2 ☑ 3.14-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/296609) |
| 2014/02/14 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [compaction related commits](https://lore.kernel.org/patchwork/cover/441817) | 内存规整相关清理和优化. 降低了内存规整 9% 的运行时间. | v2 ☑ 3.15-rc1 | [PatchWork v2 0/5](https://lore.kernel.org/patchwork/cover/441817) |
| 2015/07/02 | Mel Gorman <mel@csn.ul.ie> | [Outsourcing compaction for THP allocations to kcompactd](https://lore.kernel.org/patchwork/cover/575290) | 实现 per node 的 kcompactd 内核线程来定期触发内存规整. | RFC v2 ☑ 4.6-rc1 | [PatchWork RFC v2](https://lore.kernel.org/patchwork/cover/575290) |
| 2016/08/10 | Mel Gorman <mel@csn.ul.ie> | [make direct compaction more deterministic](https://lore.kernel.org/patchwork/cover/692460) | 更有效地直接压缩. 在内存分配的慢速路径 `__alloc_pages_slowpath` 中的一直会先尝试直接回收和压缩, 直到分配成功或返回失败.<br>1. 当回收先于压缩时更有可能成功, 因为压缩需要满足某些苛刻的条件和水线要求, 并且在有更多的空闲页面时会增加压缩成功的概率.<br>2. 另一方面, 从轻异步压缩(如果水线允许的话)开始也可能更有效, 特别是对于较小 order 的申请. 因此这个补丁慢速路径下的尝试流程修正为将先进行 MIGRATE_ASYNC 异步迁移(规整), 再尝试内存直接回收, 接着进行 MIGRATE_SYNC_LIGHT 轻度同步迁移(规整). 并引入了[直接规整的优先级](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a5508cd83f10f663e05d212cb81f600a3af46e40). | RFC v2 ☑ 4.8-rc1 & 4.9-rc1 | [PatchWork v3](https://lore.kernel.org/patchwork/cover/692460)<br>*-*-*-*-*-*-*-* <br>[PatchWork series 1 v5](https://lore.kernel.org/patchwork/cover/700017)<br>*-*-*-*-*-*-*-* <br>[PatchWork series 2 v6](https://lore.kernel.org/patchwork/cover/705827) |
| 2017/03/07 | Vlastimil Babka <vbabka@suse.cz> | [try to reduce fragmenting fallbacks](https://lore.kernel.org/patchwork/cover/766804) | 修复 [Regression in mobility grouping?](https://lkml.org/lkml/2016/9/28/94) 上报的碎片化问题, 通过修改 fallback 机制和 compaction 机制来减少永久随便化的可能性. 其中 fallback 修改时, 仅尝试从不同 migratetype 的 pageblock 中窃取的页面中挑选最小(但足够)的页面. | v3 ☑ [4.12-rc1](https://kernelnewbies.org/Linux_4.12#Memory_management) | [PatchWork v6](https://lore.kernel.org/patchwork/cover/766804), [KernelNewbies](https://kernelnewbies.org/Linux_4.12#Memory_management), [关键 commit 3bc48f96cf11 ("mm, page_alloc: split least stolen page in fallback")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3bc48f96cf11ce8699e419d5e47ae0d456403274) |



### 3.4.2 主动规整
-------

主动规整, 而不是按需规整.

对于正在进行的工作负载活动, 系统内存变得碎片化. 碎片可能会导致容量和性能问题. 在某些情况下, 程序错误也是可能的. 因此, 内核依赖于一种称为内存压缩的反应机制.

该机制的原始设计是保守的, 压缩活动是根据分配请求的要求发起的. 但是, 如果系统内存已经严重碎片化, 反应性行为往往会增加分配延迟. 通过在请求分配之前定期启动内存压缩工作, 主动压缩改进了设计. 这种增强增加了内存分配请求找到物理上连续的内存块的机会, 而不需要内存压缩来按需生成这些内存块. 因此, 特定内存分配请求的延迟降低了. 添加了一个新的 sysctl 接口 `vm.compaction_pro` 来调整内存规整的主动性, 它规定了 kcompactd 试图维护提交的外部碎片的界限.

> 警告 : 主动压缩可能导致压缩活动增加. 这可能会造成严重的系统范围的影响, 因为属于不同进程的内存页会被移动和重新映射. 因此, 启用主动压缩需要非常小心, 以避免应用程序中的延迟高峰.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/02/04 | SeongJae Park <sjpark@amazon.com> | [Proactive compaction for the kernel](https://lwn.net/Articles/817905) | 主动进行内存规整, 而不是之前的按需规整. 新的 sysctl 接口 `vm.compaction_pro` 来调整内存规整的主动性, 它规定了 kcompactd 试图维护提交的外部碎片的界限. | v8 ☑ [5.9](https://kernelnewbies.org/Linux_5.9#Memory_management) | [PatchWork v24](https://lore.kernel.org/patchwork/cover/1257280), [LWN](https://lwn.net/Articles/817905) |

# 4 页面回收
-------

[Page replacement for huge memory systems](https://lwn.net/Articles/257541)

[Short topics in memory management](https://lwn.net/Articles/224829)

从用户角度来看, 这一节对了解 Linux 内核发展帮助不大, 可跳过不读; 但对于技术人员来说, 本节可以展现教材理论模型到工程实现的一些思考与折衷, 还有软件工程实践中由简单粗糙到复杂精细的演变过程

当 MM 遭遇内存分配紧张时, 会回收页面. **页框替换算法(Page Frame Replacement Algorithm, 下称PFRA)** 的实现好坏对性能影响很大: 如果选中了频繁或将马上要用的页, 将会出现 **Swap Thrashing** 现象, 即刚换出的页又要换回来, 现象就是系统响应非常慢


[Linux内存回收机制](https://zhuanlan.zhihu.com/p/348873183)

内核之所以要进行内存回收, 主要原因有两个:

1.  内核需要为任何时刻突发到来的内存申请提供足够的内存, 以便 cache 的使用和其他相关内存的使用不至于让系统的剩余内存长期处于很少的状态.

2.  当真的有大于空闲内存的申请到来的时候, 会触发强制内存回收.

在不同的内存分配路径中, 会触发不同的内存回收方式, 内存回收针对的目标有两种, 一种是针对zone的, 另一种是针对一个memcg的, 把针对zone的内存回收方式分为三种, 分别是快速内存回收、直接内存回收、kswapd内存回收.

| 回收机制 | 描述 |
|:-------:|:---:|
| 快速内存回收机制 `node_reclaim()` | 处于get_page_from_freelist()函数中, 在遍历zonelist过程中, 对每个zone都在分配前进行判断, 如果分配后zone的空闲内存数量 < 阀值 + 保留页框数量, 那么此zone就会进行快速内存回收. 其中阀值可能是min/low/high的任何一种, 因为在快速内存分配, 慢速内存分配和oom分配过程中如果回收的页框足够, 都会调用到get_page_from_freelist()函数, 所以快速内存回收不仅仅发生在快速内存分配中, 在慢速内存分配过程中也会发生. |
| 直接内存回收 `__alloc_pages_direct_reclaim` | 处于慢速分配过程中, 直接内存回收只有一种情况下会使用, 在慢速分配中无法从zonelist的所有zone中以min阀值分配页框, 并且进行异步内存压缩后, 还是无法分配到页框的时候, 就对zonelist中的所有zone进行一次直接内存回收. 注意, 直接内存回收是针对zonelist中的所有zone的, 它并不像快速内存回收和kswapd内存回收, 只会对zonelist中空闲页框不达标的zone进行内存回收. 在直接内存回收中, 有可能唤醒flush内核线程. |
| kswapd 内存回收 | 发生在 kswapd 内核线程中, 当前每个 node 有一个 kswapd 内核线程, 也就是kswapd内核线程中的内存回收, 是只针对所在node的, 并且只会对分配了order页框数量后空闲页框数量 < 此zone的high阀值 + 保留页框数量的zone进行内存回收, 并不会对此node的所有zone进行内存回收. |

## 4.1 内存回收机制
-------

快速内存回收机制 `node_reclaim()` 在内存分配的快速路径进行, 因此一直是优化的重点.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/06/21 | Martin Hicks <mort@sgi.com> | [VM: early zone reclaim](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=753ee728964e5afb80c17659cc6c3a6fd0a42fe0) | 引入 zone_reclaim syscall, 用来回收 zone 的内存. | v1 ☑ 2.6.13-rc1 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=753ee728964e5afb80c17659cc6c3a6fd0a42fe0) |
| 2005/12/08 | Christoph Lameter <clameter@sgi.com> | [Zone reclaim V3: main patch](https://lore.kernel.org/patchwork/cover/47638) | 实现快速内存回收机制 `node_reclaim()`. 当前版本 LRU 是在 zone 上管理的, 因此引入快速页面内存回收的时候, 也是 Zone reclaim 的. 引入了 zone_reclaim_mode. | v14 ☐ | [PatchWork v5](https://lore.kernel.org/patchwork/cover/90742)<br>*-*-*-*-*-*-*-* <br>[PatchWork v14](https://lore.kernel.org/patchwork/cover/47638) |
| 2005/12/21 | Christoph Lameter <clameter@sgi.com> | [Zone reclaim Reclaim logic based on zoned counters](https://lore.kernel.org/patchwork/cover/48460) | NA | v4 ☐ | [PatchWork v4 0/3](https://lore.kernel.org/patchwork/cover/48460) |
| 2006/01/18 | Christoph Lameter <clameter@sgi.com> | [Zone reclaim: Reclaim logic](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9eeff2395e3cfd05c9b2e6074ff943a34b0c5c21) | 重构 zone reclaim. | v1 ☑ v2.6.16-rc2 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9eeff2395e3cfd05c9b2e6074ff943a34b0c5c21) |
| 2006/01/18 | Christoph Lameter <clameter@sgi.com> | [mm, numa: reclaim from all nodes within reclaim distance](https://lore.kernel.org/patchwork/patch/326889) | NA | v1 ☑ v2.6.16-rc2 | [PatchWork](https://lore.kernel.org/patchwork/cover/326889)<br>*-*-*-*-*-*-*-* <br>[PatchWork fix](https://lore.kernel.org/patchwork/cover/328345)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=957f822a0ab95e88b146638bad6209bbc315bedd) |
| 2009/06/11 | Mel Gorman <mel@csn.ul.ie> | [Fix malloc() stall in zone_reclaim() and bring behaviour more in line with expectations V3](https://lore.kernel.org/patchwork/patch/159963) | NA | v1 ☑ v2.6.16-rc2 | [PatchWork](https://lore.kernel.org/patchwork/cover/159963) |
| 2014/04/08 | Mel Gorman <mel@csn.ul.ie> | [Disable zone_reclaim_mode by default v2](https://lore.kernel.org/patchwork/patch/454625) | NA | v2 ☑ 3.16-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/454625) |


## 4.2 LRU
-------

### 4.2.1 经典地 LRU 算法
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/06/11 | Rik van Riel <riel@redhat.com> | [VM pageout scalability improvements (V12)](https://lore.kernel.org/patchwork/cover/118967) | 在大内存系统上, 扫描 LRU 中无法(或不应该)从内存中逐出的页面. 它不仅会占用CPU时间, 而且还会引发锁争用. 并且会使大型系统处于紧张的内存压力状态. 该补丁系列通过一系列措施提高了虚拟机的可扩展性:<br>1. 将文件系统支持的、交换支持的和不可收回的页放到它们自己的LRUs上, 这样系统只扫描它可以/应该从内存中收回的页<br>
2. 为匿名 LRUs 切换到双指针时钟替换, 因此系统开始交换时需要扫描的页面数量绑定到一个合理的数量<br>3. 将不可收回的页面完全远离LRU, 这样 VM 就不会浪费 CPU 时间扫描它们. ramfs、ramdisk、SHM_LOCKED共享内存段和mlock VMA页都保持在不可撤销列表中. | v12 ☑ [2.6.28-rc1](https://kernelnewbies.org/Linux_2_6_28#Various_core) | [PatchWork v2](https://lore.kernel.org/patchwork/cover/118967), [LWN](https://lwn.net/Articles/286472) |

### 4.2.2 二次机会法
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| NA | Daniel Phillips | [generic use-once optimization instead of drop-behind check_used_once](https://github.com/gatieme/linux-history/commit/6fbaac38b85e4bd3936b882392e3a9b45e8acb46) | NA | v2.4.7 -> v2.4.7.1 | NA |
| NA | Linus Torvalds <torvalds@athlon.transmeta.com> | [me: be sane about page reference bits](https://github.com/gatieme/linux-history/commit/c37fa164f793735b32aa3f53154ff1a7659e6442) | check_used_once -=> mark_page_accessed | v2.4.9.9 -> v2.4.9.10 | NA |
| 2014/05/13 | Mel Gorman <mgorman@suse.de> | [Misc page alloc, shmem, mark_page_accessed and page_waitqueue optimisations v3r33](https://lore.kernel.org/patchwork/cover/464309) | 在页缓存分配期间非原子标记页访问方案, 为了解决 dd to tmpfs 的性能退化问题. 问题的主要原因是Kconfig的不同, 但是 Mel 找到了 tmpfs、mark_page_accessible 和  page_alloc 中不必要的开销. | v3 ☑ [3.16-rc1](https://kernelnewbies.org/Linux_3.16#Memory_management) | [PatchWork v3](https://lore.kernel.org/patchwork/cover/464309) |

### 4.2.3 锁优化
-------

*   通过 pagevec 缓存来缓解 pagemap_lru_lock 锁竞争

2.5.32 合入了一组补丁优化全局 pagemap_lru_lock 锁的竞争, 做两件事:

1. 在几乎所有内核一次处理大量页面的地方, 将代码转换为一次执行16页的相同操作. 这把锁只开一次, 不要开十六次. 尽量缩短锁的使用时间.

2. 分页缓存回收函数的多线程: 在回收分页缓存页面时不要持有 pagemap_lru_lock. 这个功能非常昂贵.


这项工作的一个后果是, 我们在持有 pagemap_lru_lock 时从不使用任何其他锁. 因此, 这个锁从概念上从 VM 锁定层次结构中消失了.

所以. 这基本上都是为了提高内核可伸缩性而进行的代码调整. 它通过优化现有的设计来实现, 而不是重新设计. VM 的工作原理几乎没有概念上的改变.

```cpp
44260240ce0 [PATCH] deferred and batched addition of pages to the LRU
eed29d66442 [PATCH] pagemap_lru_lock wrapup
aaba9265318 [PATCH] make pagemap_lru_lock irq-safe
008f707cb94 [PATCH] batched removal of pages from the LRU
9eb76ee2a6f [PATCH] batched addition of pages to the LRU
823e0df87c0 [PATCH] batched movement of lru pages in writeback
3aa1dc77254 [PATCH] multithread page reclaim
6a952840483 [PATCH] pagevec infrastructure
```

| 编号 | 时间  | 作者 | 补丁 | 描述 |
|:---:|:----:|:----:|:---:|:---:|
| 1 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [6a952840483 ("pagevec infrastructure")](https://github.com/gatieme/linux-history/commit/6a95284048359fb4e1c96e02c6be0be9bdc71d6c) | 引入了 struct pagevec, 这是批处理工作的基本单元 |
| 2 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [3aa1dc77254 ("multithread page reclaim")](https://github.com/gatieme/linux-history/commit/3aa1dc772547672e6ff453117d169c47a5a7cbc5) | 借助 pagevec 完成页面回收 [shrink_cache()](https://elixir.bootlin.com/linux/v2.5.32/source/mm/vmscan.c#L278) 的并行化, 该操作需要持有 pagemap_lru_lock下运行. 借助 pagevec, 持锁后, 将 LRU 中的 32 个页面放入一个私有列表中, 然后解锁并尝试回收页面.<br>任何已成功回收的页面都将被批释放. 未回收的页面被重新添加到LRU.<br>这个补丁将 4-way 上的 pagemap_lru_lock 争用减少了 30 倍.<br>同时做的工作有: <br>对 shrink_cache() 代码进行了一些简化, 即使非活动列表比活动列表大得多, 它仍然会通过 [refill_inactive()](https://elixir.bootlin.com/linux/v2.5.32/source/mm/vmscan.c#L375) 渗透到活动列表中. |
| 3 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [823e0df87c0 ("batched movement of lru pages in writeback")](https://github.com/gatieme/linux-history/commit/823e0df87c01883c05b3ee0f1c1d109a56d22cd3) | 借助 struct pagevec 完成 [mpage_writepages()](https://elixir.bootlin.com/linux/v2.5.32/source/fs/mpage.c#L526) 的批处理, 在 LRU 上每次移动 16 个页面, 而不是每次移动一个页面. |
| 4 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [9eb76ee2a6f ("batched addition of pages to the LRU")](https://github.com/gatieme/linux-history/commit/9eb76ee2a6f64fe412bef315eccbb1dd63a203ae) | 通过对批量页面调用 lru_cache_add()的各个地方, 并对它们进行批量处理. 同时做了改进了系统在重回写负载下的行为. 页面分配失败减少了, 由于页面分配器在从 VM 回写时卡住而导致的交互性损失也有所减少. 原来 mpage_writepages() 无条件地将已写回的页面重新放到到非活动列表的头部. 现在对脏页做了优化. <br>1. 如果调用者(通常)是 balance_dirty_pages(), 则是脏页, 那么将页面留在它们在 LRU 上.<br>
如果调用者是 PF_MEMALLOC, 这些页面会 refiled 到 LRU 头. 因为只有 dirty_pages 才需要回写, 而且正在写的页面都位于 LRU 的尾部附近, 把它们留在那里, 页面分配器会过早地阻塞它们, 成为一个同步写操作. |
| 5 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [008f707cb94 ("batched removal of pages from the LRU")](https://github.com/gatieme/linux-history/commit/008f707cb94696398bac6e5b5050b3bfd0ddf054) | 这个补丁在一定程度上改变了截断锁定. 从 LRU 中删除现在发生在页面从地址空间中删除并解锁之后. 所以现在有一个窗口, shrink_cache代码可以通过LRU发现要释放的页面列表. <br>1. 将所有对 lru_cache_del() 的调用转换为使用批处理 pagevec_lru_del().<br>2. 更改 truncate_complete_page() 不从 LRU 中删除页面, 改用page_cache_release() |
| 6 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [aaba9265318 ("make pagemap_lru_lock irq-safe")](https://github.com/gatieme/linux-history/commit/aaba9265318483297267400fbfce1c399b3ac018) | 将 `spin_lock/unlock(&pagemap_lru_lock)` 替换为 `spin_lock/unlock_irq(&_pagemap_lru_lock)`. 对于 CPU 来说, 在保持页面 LRU 锁的同时进行中断是非常昂贵的, 因为在中断运行时, 其他 CPU 会在锁上堆积起来. 在持有锁时禁用中断将使 4-way 上的争用减少 30%. |
| 7 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [eed29d66442 ("pagemap_lru_lock wrapup")](https://github.com/gatieme/linux-history/commit/eed29d66442c0e6babcea33ab03f02cdf49e62af) | 第 6 个补丁之后的一些简单的 cleanup. |
| 8 | 2002/08/14 | Andrew Morton <akpm@zip.com.au> | [44260240ce0 ("deferred and batched addition of pages to the LRU")](https://github.com/gatieme/linux-history/commit/44260240ce0d1e19e84138ac775811574a9e1326) | 1. 通过 pagevec 做缓冲和延迟, lru_cache_add 分批地将页面添加到LRU.<br>2. 在页面回收代码中, 在开始之前清除本地CPU的缓冲区. 减少页面长时间不驻留在LRU上的可能性.<br>(可以有 15 * num_cpus 页不在LRU上) |

这些补丁引入了页向量(pagevec) 数据结构来完成页面的批处理, 借助一个数组来缓存特定树木的页面, 可以在不需要持有大锁 pagemap_lru_lock 的情况下, 对这些页面进行批量操作, 比单独处理一个个页面的效率要高.

首先通过 [`pagevec_add()`](https://elixir.bootlin.com/linux/v2.5.32/source/include/linux/pagevec.h#L43) 将页面插入到页向量(pvec->pages) 中, 如果页向量满了, 则通过 [`__pagevec_lru_add()`](https://elixir.bootlin.com/linux/v2.5.32/source/mm/swap.c#L197) 将页面[添加到 LRU 链表](https://elixir.bootlin.com/linux/v2.5.32/source/mm/swap.c#L61)中.


*   引入 local_locks 的概念, 它严格针对每个 CPU, 满足 PREEMPT_RT 所需的约束


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/12/05 | Alex Shi <alex.shi@linux.alibaba.com> | [Introduce local_lock()](https://lore.kernel.org/patchwork/cover/1248697) | 引入 local_locks, 在这之中进入了 lru_pvecs 结构. | v3 ☑ [5.8-rc1](https://kernelnewbies.org/Linux_5.8#Memory_management) | [PatchWork v3](https://lore.kernel.org/patchwork/cover/1248697) |

### 4.2.4 zone or node base LRU
-------

2.5.33 合入了基于 zone 的 LRU


| 编号 | 时间  | 作者 | 补丁 | 描述 |
|:---:|:----:|:----:|:---:|:---:|
| 1 | 2002/08/27 | Andrew Morton <akpm@zip.com.au> | [4fce9c6f187c ("rename zone_struct and zonelist_struct, kill zone_t and")](https://github.com/gatieme/linux-history/commit/4fce9c6f187c263e93b74c7db01b258ff77104b4) | NA |
| 2 | 2002/08/27 | Andrew Morton <akpm@zip.com.au> | [e6f0e61d9ed9 ("per-zone-LRU")](https://github.com/gatieme/linux-history/commit/e6f0e61d9ed94134f57bcf6c72b81848b9d3c2fe) | per zone 的 LRU 替换原来的全局 LRU 链表 |
| 3 | 2002/08/27 | Andrew Morton <akpm@zip.com.au> | [a8382cf11536 ("per-zone LRU locking")](https://github.com/gatieme/linux-history/commit/a8382cf1153689a1caac0e707e951e7869bb92e1) | per zone 的 lru_lock 替换原来的全局 `_pagemap_lru_lock` |

4.8 合入了基于 zone 的页面回收策略, 将 LRU 的页面回收从 zone 迁移到了 node 上.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/07/08 | Mel Gorman <mgorman@techsingularity.net> | [Move LRU page reclaim from zones to nodes v9](https://lore.kernel.org/patchwork/cover/696408) | 将 LRU 页面的回收从 ZONE 切换到 NODE. | v9 ☑ [4.8-rc1](https://kernelnewbies.org/Linux_4.8#Memory_management) | [PatchWork v21](https://lore.kernel.org/patchwork/cover/696408) |


[memcg lru lock 血泪史](https://blog.csdn.net/bjchenxu/article/details/112504932)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/12/05 | Alex Shi <alex.shi@linux.alibaba.com> | [per memcg lru lock](https://lore.kernel.org/patchwork/cover/1333353) | 实现 SLOB 分配器 | v21 ☑ [5.11](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork v21](https://lore.kernel.org/patchwork/cover/1333353) |


### 4.2.5 不同类型页面拆分管理
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/06/11 | Rik van Riel <riel@redhat.com> | [VM pageout scalability improvements (V12)](https://lore.kernel.org/patchwork/cover/118967) | 一系列完整的重构和优化补丁, 在大内存系统上, 扫描 LRU 中无法(或不应该)从内存中逐出的页面. 它不仅会占用CPU时间, 而且还会引发锁争用. 并且会使大型系统处于紧张的内存压力状态. 该补丁系列通过一系列措施提高了虚拟机的可扩展性:<br>1. 将文件系统支持的、交换支持的和不可收回的页放到它们自己的LRUs上, 这样系统只扫描它可以/应该从内存中收回的页<br>
2. 为匿名 LRUs 切换到双指针时钟替换, 因此系统开始交换时需要扫描的页面数量绑定到一个合理的数量<br>3. 将不可收回的页面完全远离LRU, 这样 VM 就不会浪费 CPU 时间扫描它们. ramfs、ramdisk、SHM_LOCKED共享内存段和mlock VMA页都保持在不可撤销列表中. | v12 ☑ [2.6.28-rc1](https://kernelnewbies.org/Linux_2_6_28#Various_core) | [PatchWork v2](https://lore.kernel.org/patchwork/cover/118966), [关键 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4f98a2fee8acdb4ac84545df98cccecfd130f8db) |

*   匿名页和文件页拆分

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/03/16 | Rik van Riel <riel@redhat.com> | [split file and anonymous page queues](https://lore.kernel.org/patchwork/cover/76770) | 将 LRU 中匿名页和文件页拆分的第一次尝试 | RFC v3 ☐ | [PatchWork RFC v1](https://lore.kernel.org/patchwork/cover/76719)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC v2](https://lore.kernel.org/patchwork/cover/76558)<br>*-*-*-*-*-*-*-*<br>[PatchWork RFC v3](https://lore.kernel.org/patchwork/cover/76558) |
| 2007/11/03 | Rik van Riel <riel@redhat.com> | [split anon and file LRUs](https://lore.kernel.org/patchwork/cover/96138) | 将 LRU 中匿名页和文件页分开管理的系列补丁 | RFC ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/96138) |
| 2008/06/11 | Rik van Riel <riel@redhat.com> | [VM pageout scalability improvements (V12)](https://lore.kernel.org/patchwork/cover/118967) | 这里我们关心的是它将 LRU 中匿名页和文件页分开成两个链表进行管理. | v12 ☑ [2.6.28-rc1](https://kernelnewbies.org/Linux_2_6_28#Various_core) | [PatchWork v2](https://lore.kernel.org/patchwork/cover/118966), [关键 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4f98a2fee8acdb4ac84545df98cccecfd130f8db) |

*   拆分出 unevictable page 的 LRU 链表

**2.6.28(2008年12月)**

[Unevictable快取的奇怪行為(Linux內核)](https://t.codebug.vip/questions-1595575.htm)

虽然现在拆分出 4 个链表了, 但还有一个问题, 有些页被**"钉"**在内存里(比如实时算法, 或出于安全考虑, 不想含有敏感信息的内存页被交换出去等原因, 用户通过 **_mlock()_**等系统调用把内存页锁住在内存里). 当这些页很多时, 扫描这些页同样是徒劳的. 内核将这些页面成为 [unevictable page](https://stackoverflow.com/questions/30891570/what-is-specific-to-an-unevictable-page). [Documentation/vm/unevictable-lru.txt](https://www.kernel.org/doc/Documentation/vm/unevictable-lru.txt)

有很多页面都被认为是 unevictable 的

[commit ba9ddf493916 ("Ramfs and Ram Disk pages are unevictable")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ba9ddf49391645e6bb93219131a40446538a5e76)

[commit 89e004ea55ab ("SHM_LOCKED pages are unevictable")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89e004ea55abe201b29e2d6e35124101f1288ef7)

[commit b291f000393f ("mlock: mlocked pages are unevictable")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b291f000393f5a0b679012b39d79fbc85c018233)

所以解决办法是把这些页独立出来, 放一个独立链表, [commit 894bc310419a ("Unevictable LRU Infrastructure")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=894bc310419ac95f4fa4142dc364401a7e607f65). 现在就有5个链表了, 不过有一个链表不会被扫描. 详情参见 [The state of the pageout scalability patches](https://lwn.net/Articles/286472).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/06/11 | Rik van Riel <riel@redhat.com> | [VM pageout scalability improvements (V12)](https://lore.kernel.org/patchwork/cover/118967) | 新增了 CONFIG_UNEVICTABLE_LRU, 将 unevictable page 用一个单独的 LRU 链表管理 | v12 ☑ [2.6.28-rc1](https://kernelnewbies.org/Linux_2_6_28#Various_core) | [PatchWork v2](https://lore.kernel.org/patchwork/cover/118967), [LWN](https://lwn.net/Articles/286472) |
| 2009/05/13 | KOSAKI Motohiro <kosaki.motohiro@jp.fujitsu.com> | [Kconfig: CONFIG_UNEVICTABLE_LRU move into EMBEDDED submenu](https://lore.kernel.org/patchwork/cover/155947) | NA | v1 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/cover/155947), [LWN](https://lwn.net/Articles/286472) |
| 2009/06/16 | KOSAKI Motohiro <kosaki.motohiro@jp.fujitsu.com> | [mm: remove CONFIG_UNEVICTABLE_LRU config option](https://lore.kernel.org/patchwork/cover/156055) | 已经没有人想要关闭 CONFIG_UNEVICTABLE_LRU 了, 因此将这个宏移除. 内核永久使能 UNEVICTABLE_LRU. | v1 ☑ v2.6.31-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/cover/156055), [commit ](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6837765963f1723e80ca97b1fae660f3a60d77df) |

*   让代码文件缓存页多待一会(Being nicer to executable pages)

**2.6.31(2009年9月发布)**

试想, 当你在拷贝一个非常大的文件时, 你发现突然电脑变得反应慢了, 那么可能发生的事情是:

突然涌入的大量文件缓存页让内存告急, 于是 MM 开始扫描前面说的链表, 如果系统的设置是倾向替换文件页的话(**_swappiness_** 靠近0), 那么很有可能, 某个 C 库代码所在代码要在这个内存吃紧的时间点(意味扫描 active list 会比较凶)被挑中, 给扔掉了, 那么程序执行到了该代码, 要把该页重新换入, 这就是发生了 **Swap Thrashing** 现象了. 这体验自然就差了.

解决方法是[对可执行页面做一些保护](https://lore.kernel.org/patchwork/cover/156462), 参见 [Being nicer to executable pages](https://lwn.net/Articles/333742), 在扫描这些在使用中的代码文件缓存页时, 跳过它, 让它有多一点时间[待在 active 链表](https://elixir.bootlin.com/linux/v2.6.31/source/mm/vmscan.c#L1301)上, 从而避免上述问题.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2009/05/17 | Wu Fengguang <fengguang.wu@intel.com> | [vmscan: make mapped executable pages the first class citizen](https://lwn.net/Articles/333742) | 将 VM_EXEC 的页面作为一等公民("the first class citizen") 区别对待. 在扫描这些在使用中的代码文件缓存页时, 跳过它, 让它有多一点时间待在 active 链表上, 从而防止抖动. | v2 ☑ 2.6.31-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/156462), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8cab4754d24a0f2e05920170c845bd84472814c6) |
| 2011/08/08 | Konstantin Khlebnikov <khlebnikov@openvz.org> | [vmscan: activate executable pages after first usage](https://lore.kernel.org/patchwork/patch/262018) | 上面补丁希望对 VM_EXEC 的页面区别对待, 但是添加的逻辑被明显削弱, 只有在第二次使用后才能成为一等公民("the first class citizen"), 由于二次机会法的存在, 这些页面将通过 page_check_references() 在第一次使用后才能进入 active LRU list, 然后才能触发前面补丁的优化, 保证 VM_EXEC 的页面有更好的机会留在内存中. 因此这个补丁 通过 page_check_references() 在页面在第一次被访问后就激活它们, 从而保护可执行代码将有更好的机会留在内存中. | v2 ☑ 3.3-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/262018), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c909e99364c8b6ca07864d752950b6b4ecf6bef4) |




### 4.2.6 页面老化(active 与 inactive 链表拆分)
-------


教科书式的 PFRA 会提到要用 LRU (Least-Recently-Used) 算法, 该算法思想基于 : 最近很少使用的页, 在紧接着的未来应该也很少使用, 因此, 它可以被当作替换掉的候选页.

但现实中, 要跟踪每个页的使用情况, 开销不是一般的大, 尤其是内存大的系统. 而且, 还有一个问题, LRU 考量的是近期的历史, 却没能体现页面的使用频率 - 假设有一个页面会被多次访问, 最近一次访问稍久点了, 这时涌入了很多只会使用一次的页(比如在播放电影), 那么按照 LRU 语义, 很可能被驱逐的是前者, 而不是后者这些不会再用的页面.

为此, Linux 引入了两个链表, 一个 active list, 一个 inactive list , 参见 [2.4.0-test9pre1 MM balancing (Rik Riel)](https://github.com/gatieme/linux-history/commit/1fc53b2209b58e786c102e55ee682c12ffb4c794). 这两个链表如此工作:

1.  inactive list 链表尾的页将作为候选页, 在需要时被替换出系统.

2.  对于文件缓存页, 当第一次被读入时, 将置于 inactive list 链表头. 如果它被再次访问, 就把它提升到 active list 链表尾; 否则, 随着新的页进入, 它会被慢慢推到 inactive list 尾巴; 如果它再一次访问, 就把它提升到 active list 链表头.

3.  对于匿名页, 当第一次被读入时, 将置于 active list 链表尾(对匿名页的优待是因为替换它出去要写入交换设备, 不能直接丢弃, 代价更大); 如果它被再次访问, 就把它提升到 active list 链表头.

4.  在需要换页时, MM 会从 active 链表尾开始扫描, 把足够量页面降级到 inactive 链表头, 同样, 默认文件缓存页会受到优待(用户可通过 **_swappiness_** 这个用户接口设置权重).

如上, 上述两个链表按照使用的热度构成了四个层级:

active 头(热烈使用中) > active 尾 > inactive 头 > inactive 尾(被驱逐者)

1.  这种增强版的 LRU 同时考虑了 LRU 的语义: 更近被使用的页在链表头;

2.  又考虑了使用频度: 还是考虑前面的例子, 一个频繁访问的页, 它极有可能在 active 链表头, 或者次一点, 在 active 链表尾, 此时涌入的大量一次性文件缓存页, 只会被放在 inactive 链表头, 因而它们会更优先被替换出去.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2.4.0-test9pre1 | Rik van Riel <riel@redhat.com> | [MM balancing (Rik Riel)](https://github.com/gatieme/linux-history/commit/1fc53b2209b58e786c102e55ee682c12ffb4c794) | 引入 MM balancing, 其中将 LRU 拆分成了 active_list 和 inactive_dirty_list 两条链表 | 2.4.0-test9pre1 | [1fc53b2209b](https://github.com/gatieme/linux-history/commit/1fc53b2209b58e786c102e55ee682c12ffb4c794) |



2.5.46 的时候, 设计了匿名页映射后, 优先加入到 active LRU list 中, 防止其被过早的释放. [commit 228c3d15a702 ("lru_add_active(): for starting pages on the active list")](https://github.com/gatieme/linux-history/commit/228c3d15a7020c3587a2c356657099c73f9eb76b), [commit a5bef68d6c85 ("start anon pages on the active list")](https://github.com/gatieme/linux-history/commit/a5bef68d6c85d26344dd31b4c342e5a365e68326).

但是这种改动又会导致另外一个问题, 这造成在某种场景下新申请的内存(即使被使用一次cold page)也会把在 active list 的 hot page 挤到 inactive list. 为了更好的解决这个问题, [workingset protection/detection on the anonymous LRU list](https://lwn.net/Articles/815342), 实现对匿名 LRU 页面列表的工作集保护和检测. 其中通过补丁 [commit b518154e59aa ("mm/vmscan: protect the workingset on anonymous LRU")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b518154e59aab3ad0780a169c5cc84bd4ee4357e) 将新创建或交换的匿名页面放到 inactive LRU list 中.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/10/31 | Andrew Morton <akpm@digeo.com> | [MM balancing (Rik Riel)](https://github.com/gatieme/linux-history/commit/1fc53b2209b58e786c102e55ee682c12ffb4c794) | 优化高压力 SWAP 下的性能, 为了防止匿名页过早的被释放, 在匿名页创建后先将他们加入到 active LRU list. | 2.5.46 | [HISTORY COMMIT1 228c3d15a70](https://github.com/gatieme/linux-history/commit/228c3d15a7020c3587a2c356657099c73f9eb76b)<br>*-*-*-*-*-*-*-*<br>[HISTORY COMMIT2 33709b5c802](https://github.com/gatieme/linux-history/commit/33709b5c8022486197ed2345eed18bbeb14a2251)<br>*-*-*-*-*-*-*-*<br>[HISTORY COMMIT3 a5bef68d6c8](https://github.com/gatieme/linux-history/commit/a5bef68d6c85d26344dd31b4c342e5a365e68326)<br>*-*-*-*-*-*-*-*<br> |
| 2020/04/03 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [workingset protection/detection on the anonymous LRU list](https://lwn.net/Articles/815342) | 实现对匿名 LRU 页面列表的工作集保护和检测. 在这里我们关心的是它将新创建或交换的匿名页面放到 inactive LRU list 中, 只有当它们被足够引用时才会被提升到活动列表. | v5 ☑ [5.9-rc1](https://kernelnewbies.org/Linux_5.9#Memory_management) | [Patchwork v7](https://lore.kernel.org/patchwork/patch/1278082)<br>*-*-*-*-*-*-*-*<br>[ZhiHu](https://zhuanlan.zhihu.com/p/113220105)<br>*-*-*-*-*-*-*-*<br>[关键 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b518154e59aab3ad0780a169c5cc84bd4ee4357e) |

2.6.31-rc1 合入了对只引用了一次的文件页面的处理, 极大地减少了一次性使用的映射文件页的生存期. 在此之前 VM 的实现中假设一个未激活的(处在 inactive LRU list 上)、映射的和引用的文件页面正在使用, 并将其提升到活动列表. 目前每个映射文件页面都是这样开始的, 因此当工作负载创建只在短时间内访问和使用的此类页面时, 就会出现问题. 这将造成 active list 中充斥大量这样的页面, 在进行页面回收时, VM 很快就会在寻找合格的回收候选对象时遇到麻烦. 结果是引起长时间的分配延迟和错误页面的回收. 补丁 [commit 645747462435 ("vmscan: detect mapped file pages used only once")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=645747462435d84c6c6a64269ed49cc3015f753d) 通过重用 PG_referenced 页面标志(用于未映射的文件页面)来实现一个使用检测, 来发现这些只引用了一次的页面. 该检测随着 LRU 列表循环的速度(即内存压力)而变化. 如果扫描程序遇到这些页面, 则设置该标志, 并在非活动列表上再次循环该页面. 只有当它返回另一个页表引用时, 它才能被激活. 否则它将被回收. 这有效地将使用过一次的映射文件页的最小生命周期从一个完整的内存周期更改为一个非活动的列表周期, 从而允许它在线性流(如文件读取)中发生, 而不会影响系统的整体性能.

但是这补丁也减少了所有共享映射文件页面的生存时间. 在这个补丁之后, 如果这个页面已经通过几个 ptes 被多次使用, page_check_references() 也会在第一次非活动列表扫描时激活这些文件映射页面.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2010/02/22 | Johannes Weiner <hannes@cmpxchg.org> | [vmscan: detect mapped file pages used only once](https://lore.kernel.org/patchwork/patch/189878) | 在文件页面线性流的场景, 源源不断的文件页面在被一次引用后, 被激活到 active LRU list 中, 将导致 active LRU list 中充斥着这样的页面. 为了解决这个问题, 通过重用 PG_referenced 页面标志检测和识别这类引用, 设置其只有在第二次引用后, 才能被激活. | v2 ☑ 2.6.31-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/189878), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=645747462435d84c6c6a64269ed49cc3015f753d) |
| 2011/08/08 | Konstantin Khlebnikov <khlebnikov@openvz.org> | [vmscan: promote shared file mapped pages](https://lore.kernel.org/patchwork/patch/262019) | 上面的补丁极大地减少了一次性使用的映射文件页的生存期. 不幸的是, 它还减少了所有共享映射文件页面的生存时间. 在这个补丁之后, 如果这个页面已经通过几个 ptes 被多次使用, page_check_references() 也会在第一次非活动列表扫描时激活文件映射页面. | v2 ☑ 3.3-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/262019), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=34dbc67a644f11ab3475d822d72e25409911e760) |

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/03/17 | Nicholas Piggin <npiggin@gmail.com> | [Multigenerational LRU Framework](https://lwn.net/Articles/851184) | 实现 SLOB 分配器 | v2 ☑ 2.6.16-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/1432175) |



## 4.2.7 LRU 中的内存水线
-------

**2.6.28(2008年12月)**

4.1 中描述过一个用户可配置的接口 : **_swappiness_**. 这是一个百分比数(取值 0-100, 默认60), 当值越靠近100, 表示更倾向替换匿名页; 当值越靠近0, 表示更倾向替换文件缓存页. 这在不同的工作负载下允许管理员手动配置.


4.1 中 规则 4)中说在需要换页时, MM 会从 active 链表尾开始扫描, 如果有一台机器的 **_swappiness_** 被设置为 0, 意为完全不替换匿名页, 则 MM 在扫描中要跳过很多匿名页, 如果有很多匿名页(在一台把 **_swappiness_** 设为0的机器上很可能是这样的), 则容易出现性能问题.


解决方法就是把链表拆分为匿名页链表和文件缓存页链表, 现在 active 链表分为 active 匿名页链表 和 active 文件缓存页链表了; inactive 链表也如此.  所以, 当 **_swappiness_** 为0时, 只要扫描 active 文件缓存页链表就够了.


## 4.2.8 工作集大小的探测(Better LRU list balancing)
-------

https://lore.kernel.org/patchwork/patch/222042
https://github.com/hakavlad/le9-patch

**3.15(2014年6月发布)**

LRU 链表被分为 inactive 和 active 链表:

*   在 active 的页面如果长时间未访问, 则会被降级到 inactive 链表, inactive 链表的页面一段时间未被访问后, 则会被释放.

*   处于 inactive 链表表头的页面, 如果它没被再次访问, 它将被慢慢推到 inactive 链表表尾, 最后在回收时被回收走; 而如果有再次访问, 它会被提升到 active 链表尾, 再再次访问, 提升到 active 链表头

但是(由于内存总量有限)如果 inactive 链表较长就意味着 active 链表相对较小; 这会引起大量的 "soft page fault", 最终导致整个系统的速度减慢. 因此, 作为内存管理决策的一部分, 两个链表的相对大小也需要采取一定的策略加以调节并保持适当的平衡.

因此, 可以定义一个概念: **访问距离, 它指该页面第一次进入内存到被踢出的间隔, 显然至少是 inactive 链表的长度.**

另外一方面, 非活动的列表应该足够小, 这样 LRU 扫描的时候不必做太多的工作. 但是要足够大, 使每个非活动页面在被淘汰之前有机会再次被引用.

**那么问题来了: 这个 inactive 链表的长度得多长? 才能既控制页面回收时扫描的工作量, 又保护该页面在第二次访问前尽量不被踢出, 以避免 Swap Thrashing 现象.**


如果进一步思考, 这个问题跟工作集大小相关. 所谓工作集, 就是维持系统所有活动的所需内存页面的最小量. 如果工作集小于等于 inactive 链表长度, 即访问距离, 则是安全的; 如果工作集大于 inactive 链表长度, 即访问距离, 则不可避免有些页要被踢出去.

当前内核采取的平衡策略相对简单: 仅仅控制 active list 的长度不要超过 inactive list 的长度一定比例, 参见 [inactive_list_is_low, v3.14, mm/vmscan.c, line 1799](https://elixir.bootlin.com/linux/v3.14/source/mm/vmscan.c#L1799), 具体的比例[与 inactive_ratio 有关](https://elixir.bootlin.com/linux/v3.14/source/mm/page_alloc.c#L5697). 其中对于文件页更是直接[要求 active list 的长度不要超过 inactive list](https://elixir.bootlin.com/linux/v3.14/source/mm/vmscan.c#L1788).

Johannes Weiner 认为这种经验公式过于简单且不够灵活, 为此他提出了一个替代方案 [Refault Distance 算法](https://lwn.net/Articles/495543). 该算法希望通过跟踪一个页框从被回收开始到(因为访问缺页)被再次载入所经历的时间长度来采取更灵活的平衡策略. 它通过估算访问距离, 来测定工作集的大小, 从而维持 inactive 链表在一个合适长度. 最早 v3.15 合入时, 只针对页面高速缓存类型的页面生效, [mm: thrash detection-based file cache sizing v9](https://lwn.net/Articles/495543). 随后被不断优化. 参见 [Better active/inactive list balancing](https://lwn.net/Articles/495543).

最开始, Refault Distance 算法只支持对文件高速缓存进行评估. [mm: thrash detection-based file cache sizing v9](https://lwn.net/Articles/495543).

后来 linux 5.9 引入了对匿名页的支持. [workingset protection/detection on the anonymous LRU list](https://lwn.net/Articles/815342).


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2014/04/03 | Johannes Weiner <hannes@cmpxchg.org> | [mm: thrash detection-based file cache sizing v9](https://lwn.net/Articles/495543) | 实现基于 Refault Distance 算法的文件高速缓存页的工作集探测, 引入 WORKINGSET_REFAULT, WORKINGSET_ACTIVATE, WORKINGSET_NODERECLAIM 三种工作集. | v9 ☑ [3.15](https://kernelnewbies.org/Linux_3.15#head-dbe2430cd9e5ed1d3f2362367758cd490aba4b9d) | [PatchWork v9](https://lore.kernel.org/patchwork/cover/437949) |
| 2015/08/03 | Johannes Weiner <hannes@cmpxchg.org> | [Make workingset detection logic memcg aware](https://lwn.net/Articles/586023) | 工作集探测感知 memcg. | v9 ☐ | [PatchWork v9](https://lore.kernel.org/patchwork/cover/586023) |
| 2016/04/04 | Johannes Weiner <hannes@cmpxchg.org> | [mm: support bigger cache workingsets and protect against writes](https://lore.kernel.org/patchwork/cover/664653) | NA | v1 ☑ [4.7-rc1](https://kernelnewbies.org/Linux_4.7#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/cover/664653) |
| 2016/07/08 | Mel Gorman <mgorman@techsingularity.net> | [Move LRU page reclaim from zones to nodes v9](https://lore.kernel.org/patchwork/cover/696408) | 将 LRU 页面的回收从 ZONE 切换到 NODE. 这里需要将 workingset 从 zone 切换到 node 上. | v9 ☑ [4.8](https://kernelnewbies.org/Linux_4.8#Memory_management) | [PatchWork v9](https://lore.kernel.org/patchwork/cover/696408), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1e6b10857f91685c60c341703ece4ae9bb775cf3) |
| 2018/08/28 | Johannes Weiner <hannes@cmpxchg.org> | [psi: pressure stall information for CPU, memory, and IO v4](https://lore.kernel.org/patchwork/cover/978495) | Refaults 发生在工作集转换和就地抖动期间. 在工作集转换期间, 非活动缓存发生 Refaults 并推出已建立的活动缓存. 但是, 如果活动缓存没有过期, 并且最终会出现 Refaults, 就会造成抖动. 引入一个新的页标志 WORKINGSET_RESTORE, 它在退出时告诉页面在其生命周期内是否处于活动状态. 然后将此位存储在影子条目中, 将故障分类为转换或抖动. | v1 ☑ [4.20-rc1](https://kernelnewbies.org/Linux_4.20#Memory_management) | [PatchWork](https://lore.kernel.org/patchwork/cover/978495), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1899ad18c6072d689896badafb81267b0a1092a4) |
| 2018/10/09 | Johannes Weiner <hannes@cmpxchg.org> | [mm: workingset & shrinker fixes](https://lore.kernel.org/patchwork/cover/997829) | 通过为循环中的影子节点添加一个计数器, 可以更容易地捕获影子节点收缩器中的 bug. | v1 ☑ [4.20-rc1](https://kernelnewbies.org/Linux_4.20#Memory_management) | [PatchWork](https://lore.kernel.org/patchwork/cover/997829), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=68d48e6a2df575b935edd420396c3cb8b6aa6ad3) |
| 2020/04/03 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [workingset protection/detection on the anonymous LRU list](https://lwn.net/Articles/815342) | 实现对匿名 LRU 页面列表的工作集保护和检测. 在之前的实现中, 新创建的或交换中的匿名页, 都是默认加入到 active LRU list, 然后逐渐降级到 inactive LRU list. 这造成在某种场景下新申请的内存(即使被使用一次cold page)也会把在a ctive list 的 hot page 挤到 inactive list. 为了解决这个的问题, 这组补丁, 将新创建或交换的匿名页面放到 inactive LRU list 中, 只有当它们被足够引用时才会被提升到活动列表. 另外,  因为这些更改可能导致新创建的匿名页面或交换中的匿名页面交换不活动列表中的现有页面, 所以工作集检测被扩展到处理匿名LRU列表. 以做出更优的决策. | v5 ☑ [5.9-rc1](https://kernelnewbies.org/Linux_5.9#Memory_management) | [PatchWork v5](https://lore.kernel.org/patchwork/cover/1219942), [Patchwork v7](https://lore.kernel.org/patchwork/patch/1278082), [ZhiHu](https://zhuanlan.zhihu.com/p/113220105) |
| 2020/05/20 | Johannes Weiner <hannes@cmpxchg.org> | [mm: balance LRU lists based on relative thrashing v2](https://lore.kernel.org/patchwork/cover/1245255) | 基于相对抖动平衡 LRU 列表(重新实现了页面缓存和匿名页面之间的 LRU 平衡, 以便更好地与快速随机 IO 交换设备一起工作). : 在交换和缓存回收之间平衡的回收代码试图仅基于内存引用模式预测可能的重用. 随着时间的推移, 平衡代码已经被调优到一个点, 即它主要用于页面缓存, 并推迟交换, 直到 VM 处于显著的内存压力之下. 因为 commit a528910e12ec Linux 有精确的故障 IO 跟踪-回收错误页面的最终代价. 这允许我们使用基于 IO 成本的平衡模型, 当缓存发生抖动时, 这种模型更积极地扫描匿名内存, 同时能够避免不必要的交换风暴. | v1 ☑ [5.8-rc1](https://kernelnewbies.org/Linux_5.8#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/cover/685701)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/1245255) |

## 4.2.9 其他优化
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2010/06/29 | Mel Gorman <mel@csn.ul.ie> | [Avoid overflowing of stack during page reclaim V3](https://lore.kernel.org/patchwork/cover/204944) | NA | v3 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/685701)<br>*-*-*-*-*-*-*-* <br>[PatchWork v3](https://lore.kernel.org/patchwork/cover/204944) |
| 2010/09/15 | Mel Gorman <mel@csn.ul.ie> | [Reduce latencies and improve overall reclaim efficiency v2](https://lore.kernel.org/patchwork/cover/215977) | NA | v2 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/cover/215977) |
| 2010/10/28 | Mel Gorman <mel@csn.ul.ie> | [Reduce the amount of time spent in watermark-related functions V4](https://lore.kernel.org/patchwork/cover/222014) | NA | v4 ☐ | [PatchWork v4](https://lore.kernel.org/patchwork/cover/222014) |
| 2010/07/30 | Mel Gorman <mel@csn.ul.ie> | [Reduce writeback from page reclaim context V6](https://lore.kernel.org/patchwork/cover/209074) | NA | v2 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/cover/209074) |

| 2021/05/27 | Muchun Song <songmuchun@bytedance.com> | [Optimize list lru memory consumption](https://lore.kernel.org/patchwork/cover/1436887) | NA | v2 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/cover/1436887) |


## 4.3 madvise MADV_FREE 页面延迟回收
-------



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/12/30 | Rik van Riel <riel@redhat.com> | [MM: implement MADV_FREE lazy freeing of anonymous memory](https://lore.kernel.org/patchwork/cover/79624) | madvise 支持页面延迟回收(MADV_FREE)的早期尝试  | v5 ☑ 4.5-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/79624) |
| 2014/07/18 | Minchan Kim | [MADV_FREE support](https://lore.kernel.org/patchwork/cover/484703) | madvise 可以用来设置页面的属性, MADV_FREE 则将这些页标识为延迟回收, 在页面用不着的时候, 可能并不会立即释放<br>1. 当内核内存紧张时, 这些页将会被优先回收, 如果应用程序在页回收后又再次访问, 内核将会返回一个新的并设置为 0 的页.<br>2. 而如果内核内存充裕时, 标识为 MADV_FREE 的页会仍然存在, 后续的访问会清掉延迟释放的标志位并正常读取原来的数据, 因此应用程序不检查页的数据, 就无法知道页的数据是否已经被丢弃. | v13 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/416962) |
| 2015/12/30 | Minchan Kim | [MADV_FREE support](https://lore.kernel.org/patchwork/cover/622178) | madvise 支持页面延迟回收(MADV_FREE)的再一次尝试  | v5 ☑ 4.5-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/622178), [KernelNewbies](https://kernelnewbies.org/Linux_4.5#Add_MADV_FREE_flag_to_madvise.282.29) |
| 2017/02/24 | Minchan Kim | [MADV_FREE support](https://lore.kernel.org/patchwork/cover/622178) | MADV_FREE 有几个问题, 使它不能在 jemalloc 这样的库中使用: 不支持系统没有交换启用, 增加了内存的压力, 另外统计也存在问题. 这个版本将 MADV_FREE 页面放到 LRU_INACTIVE_FILE 列表中, 为无交换系统启用 MADV_FREE, 并改进了统计计费.  | v5 ☑ [4.12-rc1](https://kernelnewbies.org/Linux_4.12#Memory_management) | [PatchWork v5](https://lore.kernel.org/patchwork/cover/622178) |


## 4.4 主动的页面回收
-------

### 4.4.1 [Proactively reclaiming idle memory](https://lwn.net/Articles/787611)
-------

[LWN：主动回收较少使用的内存页面](https://blog.csdn.net/Linux_Everything/article/details/96416633)

[[RFC,-V6,0/6] NUMA balancing: optimize memory placement for memory tiering system](https://lore.kernel.org/patchwork/cover/1393431)

[Software-defined far memory in warehouse scale computers](https://blog.acolyer.org/2019/05/22/sw-far-memory)

[系统软件工程师必备技能-进程内存的working set size(WSS)测量](https://blog.csdn.net/juS3Ve/article/details/85333717)

[LSF/MM 2019](https://lwn.net/Articles/lsfmm2019) 期间, 主动回收 IDLE 页面的议题引起了开发者的关注. 通过对业务持续一段时间的页面使用进行监测, 回收掉那些不常用的或者没必要的页面, 在满足业务需求的前提下, 可以节省大量的内存. 这可能比启发式的 kswapd 更有效. 这包括两部分的内容:

1.  冷热页区分:  为了能识别那些可以回收的页面, 必须对那些不常用的页面有效地进行跟踪, 即 idle page tracking.

2.  进程内存的 working set size(WSS) 估计: 为了在回收了内存之后还能满足业务的需求, 保障业务性能不下降, 需要能预测出业务运行所需要的实际最小内存. brendangregg 大神对此也有描述, [Working Set Size Estimation](https://www.brendangregg.com/wss.html), 并设计了 wss 工具 [Working Set Size (WSS) Tools for Linux](https://github.com/brendangregg/wss).


首先是 Google 的方案, 这个特性很好的诠释了上面量两部分的内容, 参见[V2: idle page tracking / working set estimation](https://lore.kernel.org/patchwork/cover/268228). 其主要实现思路如下:

1.  目前已经实现好的方案有一个 user-space 进程会频繁读取 sysfs 提供的 bitmap. 参见 [idle memory tracking](https://lore.kernel.org/patchwork/cover/580794), 引入了一个 `/sys/kernel/mm/page_idle/bitmap`, 不过这个方案CPU占用率偏高, 并且内存浪费的也不少.

2.  所以目前 Google 在试一个新方案, 基于一个名为 kstaled 的 kernel thread, 参见 [V2: idle page tracking / working set estimation](https://lore.kernel.org/patchwork/cover/268228). 这个 kernel thread 会利用 page 里的 flag 来跟踪 idle page, 所以不会再浪费更多系统内存了, 不过仍然需要挺多 CPU 时间的. 还有一个新加的 kreclaimd 线程来扫描内存, 回收那些空闲(没人读写访问)太久的页面. CPU 的开销并不小, 会随着需要跟踪的内存空间大小, 以及扫描的频率而线性增加. 在一个 512GB 内存的系统上, 可能会需要一个CPU完全用于做这部分工作. 大多数的时间都是在遍历 reverse-map 列表来查找 page mapping. 他们试过把 reverse-map 的查找去掉, 而是创建一个 PMD page table 链表, 可以改善这部分开销. 这样能减少 CPU 占用率到原来的 2/7. 还有另一个优化是把 kreclaimd 的扫描去掉而直接利用 kstaled 传入的一组页面, 也有明显的效果.

基于 kstaled 的方案, 没有合入主线, 但是 idle memory tracking 的方案在优化后, 于 4.3 合入了主线, 命名为 [IDLE_PAGE_TRACKING](https://www.kernel.org/doc/html/latest/admin-guide/mm/idle_page_tracking.html). 作者基于这个特性进程运行所需的实际内存预测(WSS), 并提供了一[系列工具 idle_page_tracking](https://github.com/sjp38/idle_page_tracking)来完成这个工作, 参见 [Idle Page Tracking Tools](https://sjp38.github.io/post/idle_page_tracking).

Facebook 指出他们也面临过同样的问题, 所有的 workload 都需要放到 container 里去执行, 用户需要明确申明需要使用多少内存, 不过其实没人知道自己真的会用到多少内存, 因此用户申请的内存数量都太多了, 也就有了类似的overcommit和reclaim问题. Facebook的方案是采用 [PSI(pressure-stall information)](https://lwn.net/Articles/759781), 根据这个来了解内存是否变得很紧张了, 相应的会把LRU list里最久未用的page砍掉. 假如这个举动导致更多的 refault 发生. 不过通过调整内存的回收就调整的激进程度可以缓和 refault. 从而达到较合理的结果, 同时占用的CPU时间也会小得多.

后来还有一些类似的特性也达到了很好的效果.

1.  intel 在对 NVDIMM/PMEM 进行支持的时候, 为了将热页面尽量使用快速的内存设备, 而冷页面尽量使用慢速的内存设备. 因此实现了冷热页跟踪机制. 完善了 idle page tracking 功能, 实现 per process 的粒度上跟踪内存的冷热. 在 reclaim 时将冷的匿名页面迁移到 PMEM 上(只能迁移匿名页). 同时利用一个 userspace 的 daemon 和 idle page tracking, 来将热内存(在PMEM上的)迁移到 DRA M中. [ept-idle](https://github.com/intel/memory-optimizer/tree/master/kernel_module).

2.  openEuler 实现的 etmem 内存分级扩展技术, 通过 DRAM + 内存压缩/高性能存储新介质形成多级内存存储, 对内存数据进行分级, 将分级后的内存冷数据从内存介质迁移到高性能存储介质中, 达到内存容量扩展的目的, 从而实现内存成本下降.

3.  Amazon 的开发人员 SeongJae Park 基于 DAMON 分析内存的访问, 在此基础上实现了[主动的内存回收机制 DAMON-based Reclamation](https://damonitor.github.io/doc/html/v29-darc-rfc-v2/admin-guide/mm/damon/reclaim.html). 使用 DAMON 监控数据访问, 以找出特定时间段内未访问的冷页, 优先将他们回收.



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/02/09 | David Rientjes <rientjes@google.com> | [smaps: extract pmd walker from smaps code](https://lore.kernel.org/patchwork/cover/73779) | Referenced Page flag, 通过在 /proc/PID/smap 中新增 pages referenced 计数的支持, 同时引入 `/proc/PID/clear_refs` 允许用户重置进程页面的 referenced 标记. 通过这种方式用户可以找出特定时间段内被访问过的页. 从而估计出进程的 WSS, 区分冷热页. | v1 ☑ 2.6.22-rc1 | [PatchWork v1 0/3](https://lore.kernel.org/patchwork/cover/73777) |
| 2007/10/09 | Matt Mackall <mpm@selenic.com> | [maps4: pagemap monitoring v4](https://lore.kernel.org/patchwork/cover/95279) | 引入 CONFIG_PROC_PAGE_MONITOR, 管理了 `/proc/pid/clear_refs`, `/proc/pid/smaps`, `/proc/pid/pagemap`, `/proc/kpagecount`, `/proc/kpageflags` 多个接口. | v1 ☑ 2.6.25-rc1 | [PatchWork v4 0/12](https://lore.kernel.org/patchwork/cover/95279) |
| 2011/09/28 | Rik van Riel <riel@redhat.com> | [V2: idle page tracking / working set estimation](https://lore.kernel.org/patchwork/cover/268228) | Google 的 kstaled 方案, 通过跟踪那些长期和回收未使用页面, 来减少内存使用, 同时不降低业务的性能. | v2 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/268228) |
| 2015/07/19 | Vladimir Davydov <vdavydov@parallels.com> | [idle memory tracking](https://lore.kernel.org/patchwork/cover/580794) | Google 的 idle page 跟踪技术, CONFIG_IDLE_PAGE_TRACKING 跟踪长期未使用的页面. | v9 ☑ 4.3-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/580794), [REDHAT Merge](https://lists.openvz.org/pipermail/devel/2015-October/067103.html), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=33c3fc71c8cfa3cc3a98beaa901c069c177dc295) |
| 2018/12/26 | Fengguang Wu <fengguang.wu@intel.com> | [PMEM NUMA node and hotness accounting/migration](https://lore.kernel.org/patchwork/cover/1027864) | 尝试使用 NVDIMM/PMEM 作为易失性 NUMA 内存, 使其对普通应用程序和虚拟内存透明. 其中引入 `/proc/PID/idle_pages` 接口, 用于用户空间驱动的热点页面进行统计. 实现冷热页扫描, 以及页面回收路径下的被动内核冷页面迁移, 改进了用于活动用户空间热/冷页面迁移的move_pages() | RFC v2 ☐ 4.20 | PatchWork RFC,v2,00/21](https://lore.kernel.org/patchwork/cover/1027864), [LKML](https://lkml.org/lkml/2018/12/26/138), [github/intel/memory-optimizer](http://github.com/intel/memory-optimizer) |
| 2021/03/18 | liubo <liubo254@huawei.com> | [etmem: swap and scan](https://gitee.com/openeuler/kernel/issues/I3W4XW) | openEuler 实现的内存分级扩展技术. | v1 ☐ 4.19 | [etmem tools](https://gitee.com/src-openeuler/etmem) |
| 2021/06/08 | SeongJae Park <sjpark@amazon.com> | [Introduce DAMON-based Proactive Reclamation](https://lore.kernel.org/patchwork/cover/1442732) | 该补丁集改进了用于生产质量的通用数据访问模式内存管理的引擎, 并在其之上实现了主动回收. | v2 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/1438747)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/1442732) |


# 5 页面写回
-------


从用户角度来看, 这一节对了解 Linux 内核发展帮助不大, 可跳过不读; 但对于技术人员来说, 本节可以展现教材理论模型到工程实现的一些思考与折衷, 还有软件工程实践中由简单粗糙到复杂精细的演变过程


当进程改写了文件缓存页, 此时内存中的内容与**后备存储设备(backing device)**的内容便处于不一致状态, 此时这种页面叫做**脏页(dirty page).** 内核会把这些脏页写回到后备设备中. 这里存在一个折衷: 写得太频繁(比如每有一个脏页就写一次)会影响吞吐量; 写得太迟(比如积累了很多个脏页才写回)又可能带来不一致的问题, 假设在写之前系统崩溃, 则这些数据将丢失, 此外, 太多的脏页会占据太多的可用内存. 因此, 内核采取了几种手段来写回:

1) 设一个后台门槛(background threshold), 当系统脏页数量超过这个数值, 用后台线程写回, 这是异步写回.

2) 设一个全局门槛(global threshold), 这个数值比后台门槛高. 这是以防系统突然生成大量脏页, 写回跟不上, 此时系统将扼制(throttle)生成脏页的进程, 让其开始同步写回.



## 5.1 由全局的脏页门槛到每设备脏页门槛
-------

**2.6.24(2008年1月发布)**


内核采取的第2个手段看起来很高明 - 扼制生成脏页的进程, 使其停止生成, 反而开始写回脏页, 这一进一退中, 就能把全局的脏页数量拉低到全局门槛下. 但是, 这存在几个微妙的问题:



> **1\.** 有可能这大量的脏页是要写回某个后备设备A的, 但被扼制的进程写的脏页则是要写回另一个后备设备B的. 这样, 一个不相干的设备的脏页影响到了另一个(可能很重要的)设备, 这是不公平的.
> **2\.** 还有一个更严重的问题出现在栈式设备上. 所谓的栈式设备(stacked device)是指多个物理设备组成的逻辑设备, 如 LVM 或 software RAID 设备上. 操作在这些逻辑设备上的进程只能感知到这些逻辑设备. 假设某个进程生成了大量脏页, 于是, 在逻辑设备这一层, 脏页到达门槛了, 进程被扼制并让其写回脏页, 由于进程只能感知到逻辑设备这一层, 所以它觉得脏页已经写下去了. 但是, 这些脏页分配到底下的物理设备这一层时, 可能每个物理设备都还没到达门槛, 那么在这一层, 是不会真正往下写脏页的. 于是, 这种极端局面造成了死锁: 逻辑设备这一层以为脏页写下去了; 而物理设备这一层还在等更多的脏页以到达写回的门槛.





2.6.24 引入了一个新的改进[Smarter write throttling](https://lwn.net/Articles/245600), 就是把全局的脏页门槛替换为每设备的门槛. 这样第1个问题自然就解决了. 第2个问题其实也解决了. 因为现在是每个设备一个门槛, 所以在物理设备这一层, 这个门槛是会比之前的全局门槛低很多的, 于是出现上述问题的可能性也不存在了.



那么问题来了, 每个设备的门槛怎么确定? 设备的写回能力有强有弱(SSD 的写回速度比硬盘快多了), 一个合理的做法是根据当前设备的写回速度分配给等比例的带宽(门槛). **这种动态根据速度调整的想法在数学上就是[指数衰减](https://zh.wikipedia.org/wiki/%E6%8C%87%E6%95%B0%E8%A1%B0%E5%87%8F)的理念:某个量的下降速度和它的值成比例.** 所以, 在这个改进里, 作者引入了一个叫"**浮动比例**"的库, 它的本质就是一个**根据写回速度进行指数衰减的级数**. (这个库跟内核具体的细节无关, 感兴趣的可以研究这个代码: [[PATCH 19/23] lib: floating proportions [LWN.net]](https://lwn.net/Articles/245603). 然后, 使用这个库, 就可以"实时地"计算出每个设备的带宽(门槛).



## 5.2 引入更具体扩展性的回写线程
-------

**2.6.32(2009年12月发布)**


Linux 内核在脏页数量到达一定门槛时, 或者用户在命令行输入 _sync_ 命令时, 会启用后台线程来写回脏页, 线程的数量会根据写回的工作量在2个到8个之间调整. 这些写回线程是面向脏页的, 而不是面向后备设备的. 换句话说, 每个回写线程都是在认领系统全局范围内的脏页来写回, 而这些脏页是可能属于不同后备设备的, 所以回写线程不是专注于某一个设备.



不过随着时间的推移, 这种看似灵巧的方案暴露出弊端.

> **1\.** 由于每个回写线程都是可以服务所有后备设备的, 在有多个后备设备, 且回写工作量大时, 线程间的冲突就变得明显了(毕竟, 一个设备同一时间内只允许一个线程写回), 当一个设备有线程占据, 别的线程就得等, 或者去先写别的设备. 这种冲突对性能是有影响的.
> **2\.** 线程写回时, 把脏页组成一个个写回请求, 挂在设备的请求队列上, 由设备去处理. 显然,每个设备的处理请求能力是有限的, 即队列长度是有限的. 当一个设备的队列被线程A占满, 新来的线程B就得不到请求位置了. 由于线程是负责多个设备的, 线程B不能在这设备上干等, 就先去忙别的, 以期这边尽快有请求空位空出来. 但如果线程A写回压力大, 一直占着请求队列不放, 那么A就及饥饿了, 时间太久就会出问题.



针对这种情况, 2.6.32为每个后备设备引入了专属的写回线程, 参见 [Flushing out pdflush](https://lwn.net/Articles/326552), 换言之, 现在的写回线程是面向设备的. 在写回脏页时, 系统会根据其所属的后备设备, 派发给专门的线程去写回, 从而避免上述问题.





## 5.3 动态的脏页生成扼制和写回扼制算法
-------

**3.1(2011年11月发布), 3.2(2012年1月发布)**


本节一开始说的写回扼制算法, 其核心就是**谁污染谁治理: 生成脏页多的进程会被惩罚, 让其停止生产, 责成其进行义务劳动, 把系统脏页写回.** 在5.1小节里, 已经解决了这个方案的针对于后备设备门槛的一些问题. 但还存在一些别的问题.



> **1.** 写回的脏页的**破碎性导致的性能问题**. 破碎性这个词是我造的, 大概的意思是, 由于被罚的线程是同步写回脏页到后备设备上的. 这些脏页在后备设备上的分布可能是很散乱的, 这就会造成频繁的磁盘磁头移动, 对性能影响非常大. 而 Linux 存在的一个块层(block layer, 倒数第2个子系统会讲)本来就是要解决这种问题, 而现在写回机制是相当于绕过它了.
>
> **2\.** 系统根据当前可用内存状况决定一个脏页数量门槛, 一到这个门槛值就开始扼制脏页生成. 这种做法太粗野了点. 有时启动一个占内存大的程序(比如一个 kvm), 脏页门槛就会被急剧降低, 就会导致粗暴的脏页生成扼制.
>
> **3\. 从长远看, 整个系统生成的脏页数量应该与所有后备设备的写回能力相一致.** 在这个方针指导下, 对于那些过度生产脏页的进程, 给它一些限制, 扼制其生成脏页的速度. 内核于是设置了一个**定点**, 在这个定点之下, 内核对进程生成脏页的速度不做限制; 但一超过定点就开始粗暴地限制.



3.1, 3.2 版本中, 来自 Intel 中国的吴峰光博士针对上述问题, 引入了动态的脏页生成扼制和[写回扼制算法 Dynamic writeback throttling[(https://lwn.net/Articles/405076). 其主要核心就是, 不再让受罚的进程同步写回脏页了, 而是罚它睡觉; 至于脏页写回, 会委派给专门的[写回线程](https://lwn.net/Articles/326552), 这样就能利用块层的合并机制, 尽可能把磁盘上连续的脏页合并再写回, 以减少磁头的移动时间.



至于标题中的"动态"的概念, 主要有下:

> **1\.** 决定受罚者睡多久. 他的算法中, 动态地去估算后备设备的写回速度, 再结合当前要写的脏页量, 从而动态地去决定被罚者要睡的时间.
> **2\.** 平缓地修改扼制的门槛. 之前进程被罚的门槛会随着一个重量级进程的启动而走人骤降, 在吴峰光的算法中, 增加了对全局内存压力的评估, 从而平滑地修改这一门槛.
> **3\.** 在进程生成脏页的扼制方面, 吴峰光同样采取反馈调节的做法, 针对写回工作量和写回速度, 平缓地(尽量)把系统的脏页生成控制在定点附近.



# 6 PageCache
-------

## 6.1 PAGE CACHE
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/06/11 | Matthew Wilcox <willy@infradead.org> | [Convert page cache to XArray](https://lore.kernel.org/patchwork/cover/951137) | 将 page cahce 的组织结构从 radix tree 切换到 [xarray](https://lwn.net/Articles/745073). | v13 ☑ 2.5.8 | [PatchWork](https://lore.kernel.org/patchwork/cover/951137) |
| 2020/06/10 | Matthew Wilcox <willy@infradead.org> | [Large pages in the page cache](https://lore.kernel.org/patchwork/cover/1254710) | NA | v6 ☐ | [PatchWork RFC,v6,00/51](https://patchwork.kernel.org/project/linux-mm/cover/20200610201345.13273-1-willy@infradead.org) |
| 2020/06/29 | Matthew Wilcox <willy@infradead.org> | [THP prep patches](https://patchwork.kernel.org/project/linux-mm/cover/20200629151959.15779-1-willy@infradead.org) | NA | v1 ☑ 2.5.8 | [PatchWork](https://patchwork.kernel.org/project/linux-mm/cover/20200629151959.15779-1-willy@infradead.org) |


https://lore.kernel.org/patchwork/cover/1324435/

## 6.2 页面预读(readahead)
-------

[linux文件预读发展过程](https://blog.csdn.net/jinking01/article/details/106541116)



从用户角度来看, 这一节对了解 Linux 内核发展帮助不大, 可跳过不读; 但对于技术人员来说, 本节可以展现教材理论模型到工程实现的一些思考与折衷, 还有软件工程实践中由简单粗糙到复杂精细的演变过程


系统在读取文件页时, 如果发现存在着顺序读取的模式时, 就会预先把后面的页也读进内存, 以期之后的访问可以快速地访问到这些页, 而不用再启动快速的磁盘读写.

### 6.2.1 原始的预读框架
-------

Linux内核的一大特色就是支持最多的文件系统, 并拥有一个虚拟文件系统(VFS)层. 早在2002年, 也就是2.5内核的开发过程中, Andrew Morton在VFS层引入了文件预读的基本框架, 以统一支持各个文件系统. 如图所示, Linux内核会将它最近访问过的文件页面缓存在内存中一段时间, 这个文件缓存被称为pagecache. 如下图所示. 一般的read()操作发生在应用程序提供的缓冲区与pagecache之间. 而预读算法则负责填充这个 pagecache. 应用程序的读缓存一般都比较小, 比如文件拷贝命令cp的读写粒度就是4KB；内核的预读算法则会以它认为更合适的大小进行预读I/O, 比比如16-128KB.


![以pagecache为中心的读和预读](./images/0002-1-readahead_page_cache.gif)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/04/09 | Andrew Morton <akpm@digeo.com> | [readahead](https://github.com/gatieme/linux-history/commit/8fa498462272fec2c16a92a9a7f67d005225b640) | 统一的预读框架, 预读算法的雏形 | v1 ☑ 2.5.8 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/8fa498462272fec2c16a92a9a7f67d005225b640) |
| 2003/02/03 | Andrew Morton <akpm@digeo.com> | [implement posix_fadvise64()](https://github.com/gatieme/linux-history/commit/fccbe3844c29beed4e665b1a5aafada44e133adc) | 引入 posix_fadvise64 | v1 ☑ 2.5.60 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/fccbe3844c29beed4e665b1a5aafada44e133adc) |


### 6.2.2 预读算法及其优化
-------

一开始, 内核的预读方案如你所想, 很简单. 就是在内核发觉可能在做顺序读操作时, 就把后面的 128 KB 的页面也读进来.

大约一年之后, Linus Torvalds 把 mmap 缺页 I/O 的预取算法单独列出, 从而形成了 read-around/read-ahead 两个独立算法(图4). read-around算法适用于那些以mmap方式访问的程序代码和数据, 它们具有很强的局域性(locality of reference)特征. 当有缺页事件发生时, 它以当前页面为中心, 往前往后预取共计128KB页面. 而readahead算法主要针对read()系统调用, 它们一般都具有很好的顺序特性. 但是随机和非典型的读取模式也大量存在, 因而readahead算法必须具有很好的智能和适应性.

![Linux中的read-around, read-ahead和direct read](./images/0002-2-readahead_algorithm.gif)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2003/07/03 | Linus Torvalds <torvalds@home.osdl.org> | [Simplify and speed up mmap read-around handling](https://github.com/gatieme/linux-history/commit/82a333fa1948869322f32a67223ea8d0ae9ad8ba) | 引入 posix_fadvise64 | v1 ☑ 2.5.75 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/82a333fa1948869322f32a67223ea8d0ae9ad8ba) |
| 2005/01/03 | Steven Pratt <slpratt@austin.ibm.com>, Ram Pai <linuxram@us.ibm.com> | [Simplified readahead](https://github.com/gatieme/linux-history/commit/6f734a1af323ab4690610ecd575198ae219b6fe8) | 引入读大小参数, 代码简化及优化; 支持随机读. | v1 ☑ 2.6.11 | [HISTORY commit 1](https://github.com/gatieme/linux-history/commit/6f734a1af323ab4690610ecd575198ae219b6fe8), [HISTORY commit 2](250c01d06ccb125519cc9958d938f41736868be9) |
| 2005/03/07 | Oleg Nesterov <oleg@tv-sign.ru> | [readahead: improve sequential read detection](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=671ccb4b50a6ef21e8c0ed0ef9070098295e1e61) | 支持非对齐顺序读. | v1 ☑ [2.6.12](https://kernelnewbies.org/Linux_2_6_12) | [commit 1](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=671ccb4b50a6ef21e8c0ed0ef9070098295e1e61), [commit 2](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=577a3dd8fd68d24056075fdf479a1627586f8c46) |



### 6.2.3 按需预读(On-demand Readahead)
-------

**2.6.23(2007年10月发布)**


这种固定的128 KB预读方案显然不是最优的. 它没有考虑系统内存使用状况和进程读取情况. 当内存紧张时, 过度的预读其实是浪费, 预读的页面可能还没被访问就被踢出去了. 还有, 进程如果访问得凶猛的话, 且内存也足够宽裕的话, 128KB又显得太小家子气了.

后续通过 Steven Pratt、Ram Pai 等人的大量工作, readahead算法进一步完善. 其中最重要的一点是实现了对随机读的完好支持. 随机读在数据库应用中处于非常突出的地位. 在此之前, 预读算法以离散的读页面位置作为输入, 一个多页面的随机读会触发“顺序预读”. 这导致了预读I/O数的增加和命中率的下降. 改进后的算法通过监控所有完整的 read( )调用, 同时得到读请求的页面偏移量和数量, 因而能够更好的区分顺序读和随机读.

2.6.23的内核引入了在这个领域耕耘许久的吴峰光的一个[按需预读的算法]((https://lwn.net/Articles/235164). 所谓的按需预读, 就是内核在读取某页不在内存时, 同步把页从外设读入内存, 并且, 如果发现是顺序读取的话, 还会把后续若干页一起读进来, 这些预读的页叫预读窗口; 当内核读到预读窗口里的某一页时, 如果发现还是顺序读取的模式, 会再次启动预读, 异步地读入下一个预读窗口.



该算法关键就在于适当地决定这个预读窗口的大小,和哪一页做为异步预读的开始. 它的启发式逻辑也非常简单, 但取得不了错的效果. 此外, 对于两个进程在同一个文件上的交替预读, 2.6.24 增强了该算法, 使其能很好地侦测这一行为.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/10/06 | WU Fengguang <wfg@mail.ustc.edu.cn> | [Adaptive file readahead](https://lwn.net/Articles/155510) | 自适应预读算法 | v1 ☐ | [LWN](https://lwn.net/Articles/155097) |
| 2011/05/17 | WU Fengguang <wfg@mail.ustc.edu.cn> | [512K readahead size with thrashing safe readahead](https://lwn.net/Articles/372384) | 将每次预读窗口大小的最大值从 128KB 增加到了 512KB, 其中增加了一个统计接口(tracepoint, stat 节点等). | v3 ☐ | [PatchWork](https://lore.kernel.org/patchwork/cover/190891), [LWN](https://lwn.net/Articles/234784) |
| 2011/05/17 | WU Fengguang <wfg@mail.ustc.edu.cn> | [on-demand readahead](https://lwn.net/Articles/235164) | on-demand 预读算法 | v1 ☑ [2.6.23-rc1](https://kernelnewbies.org/Linux_2_6_23#On-demand_read-ahead) | [LWN](https://lwn.net/Articles/234784) |


# 7 大内存页支持
-------


我们知道现代操作系统都是以页面(page)的方式管理内存的. 一开始, 页面的大小就是4K, 在那个时代, 这是一个相当大的数目了. 所以众多操作系统, 包括 Linux , 深深植根其中的就是一个页面是4K大小这种认知, 尽管现代的CPU已经支持更大尺寸的页面(X86体系能支持2MB, 1GB).



我们知道虚拟地址的翻译要经过页表的翻译, CPU为了支持快速的翻译操作, 引入了TLB的概念, 它本质就是一个页表翻译地址结果的缓存, 每次页表翻译后的结果会缓存其中, 下一次翻译时会优先查看TLB, 如果存在, 则称为TLB hit; 否则称为TLB miss, 就要从访问内存, 从页表中翻译. 由于这是一个CPU内机构, 决定了它的尺寸是有限的, 好在由于程序的局部性原理, TLB 中缓存的结果很大可能又会在近期使用.



但是, 过去几十年, 物理内存的大小翻了几番, 但 TLB 空间依然局限, 4KB大小的页面就显得捉襟见肘了. 当运行内存需求量大的程序时, 这样就存在更大的机率出现 TLB miss, 从而需要访问内存进入页表翻译. 此外, 访问更多的内存, 意味着更多的缺页中断. 这两方面, 都对程序性能有着显著的影响.

[[v2,0/4] x86, mm: Handle large PAT bit in pud/pmd interfaces](https://lore.kernel.org/patchwork/cover/579540)

[[v12,0/10] Support Write-Through mapping on x86](https://lore.kernel.org/patchwork/cover/566256/)

## 7.1 标准大页 HUGETLB 支持
-------


**(2.6前引入)**



如果能使用更大的页面, 则能很好地解决上述问题. 试想如果使用2MB的页(一个页相当于512个连续的4KB 页面), 则所需的 TLB 表项由原来的 512个变成1个, 这将大大提高 TLB hit 的机率; 缺页中断也由原来的512次变为1次, 这对性能的提升是不言而喻的.



然而, 前面也说了 Linux 对4KB大小的页面认知是根植其中的, 想要支持更大的页面, 需要对非常多的核心的代码进行大改动, 这是不现实的. 于是, 为了支持大页面, 有了一个所谓 HUGETLB 的实现.



它的实现是在系统启动时, 按照用户指定需求的最大大页个数, 每个页的大小. 预留如此多个数的大. . 用户在程序中可以使用 **mmap()** 系统调用或共享内存的方式访问这些大页, 例子网上很多, 或者参考官方文档:[hugetlbpage.txt [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/375098) . 当然, 现在也存在一些用户态工具, 可以帮助用户更便捷地使用. 具体可参考此文章: [Huge pages part 2: Interfaces [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/375096)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/05/10 | Muchun Song <songmuchun@bytedance.com> | [Free some vmemmap pages of HugeTLB page](https://lore.kernel.org/patchwork/cover/1422994) | madvise 支持页面延迟回收(MADV_FREE)的早期尝试  | v23 ☑ 4.5-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/1422994) |


这一功能的主要使用者是数据库程序.

## 7.2 透明大页的支持
-------


**2.6.38(2011年3月发布)**



7.1 介绍的这种使用大页的方式看起来是挺怪异的, 需要用户程序做出很多修改. 而且, 内部实现中, 也需要系统预留一大部分内存. 基于此, 2.6.38 引入了一种叫[透明大页 Transparent huge pages](https://lwn.net/Articles/423584) 的实现. 如其名字所示, 这种大页的支持对用户程序来说是透明的.



它的实现原理如下. 在缺页中断中, 内核会**尝试**分配一个大页, 如果失败(比如找不到这么大一片连续物理页面), 就还是回退到之前的做法: 分配一个小页. 在系统内存紧张需要交换出页面时, 由于前面所说的根植内核的4KB页面大小的因, MM 会透明地把大页分割成小页再交换出去.



用户态程序现在可以完成无修改就使用大页支持了. 用户还可以通过 **madvice()** 系统调用给予内核指示, 优化内核对大页的使用. 比如, 指示内核告知其你希望进程空间的某部分要使用大页支持, 内核会尽可能地满足你.

https://lore.kernel.org/patchwork/cover/1118785

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/05/10 | Muchun Song <songmuchun@bytedance.com> | [Overhaul multi-page lookups for THP](https://lore.kernel.org/patchwork/cover/1337675) | 提升大量页面查找时的效率 | v4 ☑ [5.12-rc1](https://kernelnewbies.org/Linux_5.12#Memory_management) | [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20201112212641.27837-1-willy@infradead.org) |


# 8 进程虚拟地址空间(VMA)
-------

## 8.1 VMA
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/12/19 | Colin Cross | [mm: add a field to store names for private anonymous memory](https://lore.kernel.org/patchwork/cover/416962) | 在二进制中通过 xxx_sched_class 地址顺序标记调度类的优先级, 从而可以通过直接比较两个 xxx_sched_class 地址的方式, 优化调度器中两个热点函数 pick_next_task()和check_preempt_curr(). | v2 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/416962)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/416962) |
| 2018/05/17 | Laurent Dufour <ldufour@linux.vnet.ibm.com> | [Speculative page faults](https://lore.kernel.org/patchwork/cover/906210) | 优化 MM 的 locking | v11 ☐ | [PatchWork](https://lore.kernel.org/patchwork/cover/906210)
| 2020/02/24 | Michel Lespinasse <walken@google.com> | [Fine grained MM locking](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) | 优化 MM 的 locking | RFC ☐ | [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) |



## 8.2 Mapping
-------

### 8.2.1 COW
-------

在 fork 进程的时候, 并不会为子进程直接分配物理页面, 而是使用 COW 的机制. 在 fork 之后, 父子进程都共享原来父进程的页面, 同时将父子进程的 COW mapping 都设置为只读. 这样当这两个进程触发写操作的时候, 触发缺页中断, 在处理缺页的过程中再为该进程分配出一个新的页面出来.

```cpp
copy_process
copy_mm
-=> dup_mm
    -=> dup_mmap
        -=> copy_page_range
            -=> copy_p4d_range
                -=> copy_pud_range
                    -=> copy_pmd_range
                        -=> copy_one_pte
                            -=> ptep_set_wrprotect
```


### 8.2.1 enhance
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/09/10 | Catalin Marinas <catalin.marinas@arm.com> | [arm64: Add support for hardware updates of the access and dirty pte bits](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1436545468-1549-1-git-send-email-catalin.marinas@arm.com) | ARMv8.1 体系结构扩展引入了对页表项中访问和脏信息的硬件更新的支持, 用于支持硬件自动原子地完成页表更新("读-修改-回写").<br>TCR_EL1.HA 为 1, 则使能硬件的自动更新访问位. 当处理器访问内存地址时, 硬件自动设置 PTE_AF 位, 而不是再触发访问位标志错误.<br>TCR_EL1.HD 为 1, 则使能硬件的脏位管理.  | v1 ☑ [4.3-rc1](https://kernelnewbies.org/Linux_4.3#Architectures) | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/344775)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/cover/344816), [commit 2f4b829c625e](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2f4b829c625ec36c2d80bef6395c7b74cea8aac0) |
| 2017/07/25 | Catalin Marinas <catalin.marinas@arm.com> | [arm64: Fix potential race with hardware DBM in ptep_set_access_flags()](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20170725135308.18173-2-catalin.marinas@arm.com) | 修复前面支持 TCP_EL1.HA/HD 引入的硬件和软件的竞争问题 | RFC ☐ | [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) |


### 8.2.3 MMAP locing
-------

[[LSF/MM TOPIC] mmap locking topics](https://www.spinics.net/lists/linux-mm/msg258803.html)

[The LRU lock and mmap_sem](https://lwn.net/Articles/753058)

https://events.static.linuxfound.org/sites/events/files/slides/mm.pdf

*   SPF(Speculative page faults)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/05/17 | Laurent Dufour <ldufour@linux.vnet.ibm.com> | [Speculative page faults](http://lore.kernel.org/patchwork/patch/906210) | SPF | v11 ☐  | [PatchWork](https://lore.kernel.org/patchwork/patch/906210) |
| 2021/04/20 | Michel Lespinasse <michel@lespinasse.org> | [Speculative page faults (anon vmas only)](http://lore.kernel.org/patchwork/patch/1420569) | SPF | v11 ☐  | [PatchWork RFC,00/37](https://lore.kernel.org/patchwork/cover/1408784)<br>*-*-*-*-*-*-*-* <br>[PatchWork v1](https://lore.kernel.org/patchwork/patch/1420569) |

* Fine grained MM locking


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/01/31 | Michel Lespinasse <walken@google.com> | [Mapping range lock](https://lore.kernel.org/patchwork/cover/356467) | 文件映射的 mapping lock | RFC ☐  | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/356467) |
| 2020/02/24 | Michel Lespinasse <walken@google.com> | [Fine grained MM locking](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) | 细粒度 MM MMAP lock | RFC ☐  | [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) |

## 8.3 反向映射 RMAP(Reverse Mapping)
-------

[郭健： Linux内存逆向映射(reverse mapping) 技术的前世今生](https://blog.51cto.com/u_15015138/2557286)

[Reverse-Mapping VM Performance](https://www.usenix.org/conference/2003-linux-kernel-developers-summit/presentation/reverse-mapping-vm-performance)

[Virtual Memory Management Techniques in 2.6 Linux kernel and challenges](https://www.researchgate.net/publication/348620478_Virtual_Memory_Management_Techniques_in_26_Linux_kernel_and_challenges)

[OLS2003](https://www.kernel.org/doc/ols/2003)

[linux内存管理-反向映射](https://blog.csdn.net/faxiang1230/article/details/106609834)

[如何知道物理内存中的某个页帧属于某个进程, 或者说进程的某个页表项?](https://www.zhihu.com/question/446137543)

RMAP 反向映射是一种物理地址反向映射虚拟地址的方法.

正向映射是通过虚拟地址根据页表找到物理内存, 而反向映射就是通过物理地址或者物理页面找到哪些进程或者哪些虚拟地址使用它.

那内核在哪些路径下需要进行反向映射呢?

反向映射的典型应用场景：

1.  kswapd 进行页面回收时, 需要断开所有映射了该匿名页面的PTE表项;

2.  页面迁移时, 需要断开所有映射了该匿名页面的PTE表项;

即一个物理页面是可以被多个进程映射到自己的虚拟地址空间. 在整个内存管理的过程中, 有些页面可能被迁移, 有些则可能要被释放、写回到磁盘或者 交换到 SWAP 空间(磁盘) 上, 那么这时候经常要找到哪些进程映射并使用了这个页面.

*   2.4 的 reverse mapping


在 Linux 2.4 的时代, 为了确定映射了某个页面的所以进程, 我们不得不遍历每个进程的页表, 这是一项费时费力的工作.

*   PTE-based reverse mapping

于是在 2002 年 [Linux 2.5.27](https://kernelnewbies.org/LinuxVersions) 内核开发者实现了一种低开销的[基于 PTE 的的反向映射方方案(low overhead pte-based reverse mapping scheme)](http://lastweek.io/notes/rmap) 来解决这一问题. 经过了不断的发展, 到 2.5.33 趋于完善.

这个版本的实现相当直接, 在每个物理页(包括匿名页和文件映射页)(的通过结构体类型 struct page ) 中维护了一个名为 [pte_chain 的链表](https://elixir.bootlin.com/linux/v2.5.27/source/include/linux/mm.h#L161), 里面包含所有引用该页的页表项, 链表的每一项都直接指向映射该物理页面的页表项 PTE 的指针.

该机制在大多数情况下都很高效, 但其自身也存在一些问题. 链表中保存反向映射信息的节点很多, 占用了大量内存, 并且 pte_chain 链表本身的维护的工作量也十分繁重, 并且处理速度随着物理页面的增多而变得缓慢. 以 fork() 系统调用为例, 由于需要为进程地址空间中的每个页面都添加新的反向映射项, 所以处理较慢, 影响了进程创建的速度. 关于这个方案的详细实现可以参考 [PATCH 2.5.26-rmap-1-core](http://loke.as.arizona.edu/~ckulesa/kernel/rmap-vm/2.5.26/2.5.26-rmap-1-core) 或者 [CODE mm/rmap.c, v2.5.27](https://elixir.bootlin.com/linux/v2.5.27/source/mm/rmap.c). 论文参见 [Object-based Reverse Mapping](https://landley.net/kdocs/ols/2004/ols2004v2-pages-71-74.pdf)

社区一直在努力试图优化 (PTE-based) RMAP 的整体效率.

*   object-based reverse mapping(objrmap)

在 2003 年, IBM 的 Dave McCracken 提出了一种新的解决方法 [**基于对象的反向映射**("object-based reverse mapping")](https://lwn.net/Articles/23732), 简称 objrmap. 虽然这组补丁当时未合入主线, 但是向社区证实了, 从 struct page 找到映射该物理页的页表项 PTE. 该方法虽然存在一些问题, 但是测试证明可以显著解决 RMAP 在部分场景的巨大开销问题. 参见 [Performance of partial object-based rmap](https://lkml.org/lkml/2003/2/19/235).

用户空间进程的页面主要有两种, 一种是 file mapped page, 另外一种是 anonymous mapped page. Dave McCracken 的 objrmap 方案虽然性能不错(保证逆向映射功能的基础上, 同时又能修正 rmap 带来的各种问题). 但是只是适用于 file mapped page, 对于匿名映射页面, 这个方案无能为力.

因此 Hugh Dickins 在 [2003 年](https://lore.kernel.org/patchwork/patch/14844) ~ [2004 年](https://lore.kernel.org/patchwork/patch/22938)的提交了[一系列补丁](https://lore.kernel.org/patchwork/patch/23565), 试图在 Dave McCracken 补丁的基础上, 进一步优化 RMAP 在匿名映射页面上的问题.

与此同时, SUSE 的 Andrea Arcangeli 当时正在做的工作就是让 32-bit 的 Linux 运行在配置超过 32G 内存的公司服务器上. 在这些服务器上往往启动大量的进程, 共享了大量的物理页帧, 消耗了大量的内存. 对于 Andrea Arcangeli 来说, 内存消耗的罪魁祸首是非常明确的 : 反向映射模块, 这个模块消耗了太多的 low memory, 从而导致了系统的各种 crash. 为了让自己的工作继续推进, 他必须解决 rmap 引入的内存扩展性 (memory scalability) 问题, 最直接有效的思路同样是在 Dave McCracken's objrmap 的基础上, 为匿名映射页面设计一种基于对象的逆向映射机制. 最后 Andrea Arcangeli [设计了 full objrmap 方案](https://lwn.net/Articles/77106). 并在与 Hugh Dickins 类似方案的 PK 中胜出. 并最终在 [v2.6.7](https://elixir.bootlin.com/linux/v2.6.7/source/mm/rmap.c#322) 合入主线.

> 小故事:
>
> 关于 32-bit 的 Linux 内核是否支持 4G 以上的 memory. 在 1999 年, Linus 的决定是 :
>
> 1999年, linus说:32位linux永远不会, 也不需要支持大于2GB的内存, 没啥好商量的.
>
> 不过历史的洪流不可阻挡, 处理器厂商设计了扩展模块以便寻址更多的内存, 高端的服务器也配置了越来越多的内存.
>
> 这也迫使 Linus 改变之前的思路, 让 Linux 内核支持更大的内存.


通过新增结构 anon_vma, 类似于重用 address_space 的想法, 即拥有一个数据结构 trampoline.
一个 VMA 中的所有页面只共享一个 anon_vma. vma->anon_vma 表示 vma 是否映射了页面. 相关的处理在 do_anonymous_fault()-=>anon_vma_prepare().

相对于基本的基于PTE-chain的解决方案, 基于对象的 RMAP :

| 比较 | 描述 |
|:---:|:----:|
| 优势 |  在页面错误期间, 我们只需要设置page->映射为指向anon_vma结构, 而不是分配一个新结构并插入.
| 不足 | 在rmap遍历期间, 我们需要额外的计算来遍历每个VMA的页表, 以确保该页实际上被映射到这个特定的VMA中. |


*    anon_vma_chain(AVC-per process anon_vma)

旧的机制下, 多个进程共享一个 AV(anon_vma), 这可能导致大量分支工作负载的可伸缩性问题. 具体来说, 由于每个 anon_vma 将在父进程和它的所有子进程之间共享.

1.  这样在进程 fork 子进程的场景下, 如果进行 COW, 父子进程原本共享的 page frame 已经不再共享, 然而, 这两个 page 却仍然指向同一个 anon_vma.

2.  即使一开始就没有在父子进程之间共享的页面, 当首次访问的时候(无论是父进程还是子进程), 通过 do_anonymous_page 函数分配的 page frame 也是同样的指向一个 anon_vma.


在有些网路服务器中, 系统非常依赖fork, 某个服务程序可能会fork巨大数量的子进程来处理服务请求, 在这种情况下, 系统性能严重下降. Rik van Riel给出了一个具体的示例：系统中有1000进程, 都是通过fork生成的, 每个进程的VMA有 1000个匿名页. 根据目前的软件架构, anon_vma链表中会有1000个vma 的节点, 而系统中有一百万个匿名页面属于同一个anon_vma.


这可能导致这样的系统 : 一个 CPU 遍历 page_referenced_one 中的 1000 个进程的页表, 而其他所有 CPU 都被 anon_vma 锁卡住. 这将导致像 AIM7 这样的基准测试出现灾难性的故障, 在那里进程的总数可能达到数万个. 实际工作量仍然比AIM7少10倍, 但它们正在迎头赶上.

这个补丁改变了 anon_vmas 和 VMA 的链接方式, 它允许我们将多个anon_vmas与一个VMA关联起来. 在分叉时, 每个子进程都获得自己的anon_vmas, 它的 COW 页面将在其中实例化. 父类的anon_vma也链接到VMA, 因为非cowed页面可以出现在任何子类中. 这将 1000 个子进程的页面的 RMAP 扫描复杂度降低到 O(1), 系统中最多 1/N 个页面的 RMAP 扫描复杂度为 O(N). 这将把繁重的工作负载从O(N)减少到很小的平均扫描成本.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/07/20 | Rik van Riel <riel@redhat.com> | [VM with reverse mappings](https://www.cs.helsinki.fi/linux/linux-kernel/2002-46/0281.html) | 反向映射 RMAP 机制 | ☑ 2.5.27 | [Patch Archive](http://loke.as.arizona.edu/~ckulesa/kernel/rmap-vm/2.5.31/readme.html), [LWN](https://lwn.net/Articles/2925) |
| 2003/02/20 | Dave McCracken <dmccr@us.ibm.com> | [The object-based reverse-mapping VM](https://lwn.net/Articles/23732) | 基于对象的反向映射技术 | ☐ 2.5.62 in Andrew Morton's tree | [LWN](https://lwn.net/Articles/23584) |
| 2003/04/02 | Dave McCracken <dmccr@us.ibm.com> | [Optimizaction for object-based rmap](https://lore.kernel.org/patchwork/patch/15415) | 优化基于对象的反向映射 | ☐ 2.5.66 | [PatchWork](https://lore.kernel.org/patchwork/patch/15415) |
| 2003/03/20 | Hugh Dickins <hugh@veritas.com> | [anobjrmap 0/6](https://lore.kernel.org/patchwork/patch/14844) | 扩展了Dave McCracken的objrmap来处理匿名内存 | ☐ 2.5.66 | [PatchWork for 6 patches 2003/03/20](https://lore.kernel.org/patchwork/patch/14844)<br>*-*-*-*-*-*-*-* <br>[PatchWork for 2.6.5-rc1 8 patches 2004/03/18](https://lore.kernel.org/patchwork/patch/22938)<br>*-*-*-*-*-*-*-* <br>[PatchWork for 2.6.5-mc2 40 patches 2004/04/08](https://lore.kernel.org/patchwork/patch/23565) |
| 2004/03/10 | Andrea Arcangeli <andrea@suse.de> | [Virtual Memory II: full objrmap](https://lwn.net/Articles/75198) | 基于对象的反向映射技术的有一次尝试 | v1 ☑ [2.6.7](https://elixir.bootlin.com/linux/v2.6.7/source/mm/rmap.c#322) | [230-objrmap fixes for 2.6.3-mjb2](https://lore.kernel.org/patchwork/patch/22524)<br>*-*-*-*-*-*-*-* <br>[objrmap-core-1](https://lore.kernel.org/patchwork/patch/22623)<br>*-*-*-*-*-*-*-* <br>[RFC anon_vma previous (i.e. full objrmap)](https://lore.kernel.org/patchwork/patch/22674)<br>*-*-*-*-*-*-*-* <br>[anon_vma RFC2](https://lore.kernel.org/patchwork/patch/22694)<br>*-*-*-*-*-*-*-* <br>[anon-vma](https://lore.kernel.org/patchwork/patch/23307) |
| 2009/11/24 | Hugh Dickins <hugh.dickins@tiscali.co.uk> | [ksm: swapping](https://lore.kernel.org/patchwork/patch/179157) | 内核Samepage归并(KSM)是Linux 2.6.32 中合并的一个特性, 它对虚拟客户机的内存进行重复数据删除. 但是, 该实现不允许交换共享的页面. 这个版本带来了对KSM页面的交换支持.<br>这个对 RMAP 的影响是, RMAP 支持 KASM 页面. 同时用 rmap_walk 的方式 | v1 ☑ [2.6.33-rc1](https://kernelnewbies.org/Linux_2_6_33#Swappable_KSM_pages) | [PatchWork v1](https://lore.kernel.org/patchwork/patch/179157) |
| 2010/01/14 | Rik van Riel <riel@redhat.com> | [change anon_vma linking to fix multi-process server scalability issue](https://lwn.net/Articles/75198) | 引入 AVC, 实现了 per process anon_vma, 每一个进程的 page 都指向自己特有的 anon_vma 对象. | v1 ☑ [2.6.34](https://kernelnewbies.org/Linux_2_6_34#Various_core_changes) | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/185210)<br>*-*-*-*-*-*-*-* <br>[PatchWork v1](https://lore.kernel.org/patchwork/patch/186566) |
| 2013/12/4 | Rik van Riel <riel@redhat.com> | [mm/rmap: unify rmap traversing functions through rmap_walk](https://lwn.net/Articles/424318) | 引入 AVC, 实现了 per process anon_vma, 每一个进程的 page 都指向自己特有的 anon_vma 对象. | v1 ☑ 3.4-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/424318) |
| 2012/12/01 | Davidlohr Bueso <davidlohr@hp.com> | [mm/rmap: Convert the struct anon_vma::mutex to an rwsem](https://lore.kernel.org/patchwork/cover/344816) | 将struct anon_vma::mutex 转换为rwsem, 这将有效地改善页面迁移过程中对 VMA 的访问的竞争问题. | v1 ☑ 3.8-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/344775)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/cover/344816) |
| 2014/05/23 | Davidlohr Bueso <davidlohr@hp.com> | [mm: i_mmap_mutex to rwsem](https://lore.kernel.org/patchwork/cover/466974) | 2012 年 Ingo 将 anon-vma lock 从 mutex 转换到了 rwsem 锁. i_mmap_mutex 与 anon-vma 有类似的职责, 保护文件支持的页面. 因此, 我们可以使用类似的锁定技术 : 将互斥锁转换为rwsem, 并在可能的情况下共享锁. | RFC ☐ | [PatchWork](https://lore.kernel.org/patchwork/cover/466974) |


## 8.4 Mremapping
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/10/14 | Kalesh Singh <kaleshsingh@google.com> | [Speed up mremap on large regions](https://lore.kernel.org/patchwork/cover/1321018) | mremap PUD 映射优化, 可以加速映射大块地址时的速度. 如果源地址和目的地址是 PMD/PUD 对齐和 PMD/PUD 大小, mremap 的耗时时间可以通过移动 PMD/PUD 级别的表项来优化. 允许在 arm64 和 x86 上的 PMD 和 PUD 级别移动. 其他支持这种类型移动且已知安全的体系结构也可以通过启用 HAVE_MOVE_PMD 和 HAVE_MOVE_PUD 来选择这些优化. | v4 ☑ | [PatchWork v4,0/5](https://patchwork.kernel.org/project/linux-mm/cover/20201014005320.2233162-1-kaleshsingh@google.com) |


## 8.5 filemapping
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/10/25 | Kalesh Singh <kaleshsingh@google.com> | [generic_file_buffered_read() improvements](https://lore.kernel.org/patchwork/cover/1324435) | filemap 的批量内存分配和拷贝. | v4 ☑ [5.11-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork v2,0/2](https://lore.kernel.org/patchwork/cover/1324435) |


## 8.6 ioremap
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2006/08/10 | Haavard Skinnemoen <hskinnemoen@atmel.com> | [Generic ioremap_page_range: introduction](https://lore.kernel.org/patchwork/cover/62430) | 基于i386实现的 ioremap_page_range() 的通用实现, 将 I/O 地址空间映射到内核虚拟地址空间. | v1 ☑ 2.6.19-rc1 | [PatchWork 0/14](https://lore.kernel.org/patchwork/cover/62430) |
| 2015/03/03 | Toshi Kani <toshi.kani@hp.com> | [Kernel huge I/O mapping support](https://lore.kernel.org/patchwork/cover/547056) | ioremap() 支持透明大页. 扩展了 ioremap() 接口, 尽可能透明地创建具有大页面的 I/O 映射. 当一个大页面不能满足请求范围时, ioremap() 继续使用 4KB 的普通页面映射. 使用 ioremap() 不需要改变驱动程序. 但是, 为了使用巨大的页面映射, 请求的物理地址必须以巨面大小(x86上为 2MB 或 1GB)对齐. 内核巨页的 I/O 映射将提高 NVME 和其他具有大内存的设备的性能, 并减少创建它们映射的时间. | v3 ☑ 4.1-rc1 | [PatchWork v3,0/6](https://lore.kernel.org/patchwork/cover/547056) |
| 2015/05/15 | Haavard Skinnemoen <hskinnemoen@atmel.com> | [mtrr, mm, x86: Enhance MTRR checks for huge I/O mapping](https://lore.kernel.org/patchwork/cover/943736) | 增强了对巨页 I/O 映射的 MTRR 检查.<br>1. 允许 pud_set_huge() 和 pmd_set_huge() 创建一个巨页映射, 当范围被任何内存类型的单个MTRR条目覆盖时. <br>2. 当指定的 PMD 映射范围超过一个 MTRR 条目时, 记录 pr_warn_once() 消息. 当这个范围被 MTRR 覆盖时, 驱动程序应该发出一个与单个 MTRR 条目对齐的映射请求. | v5 ☐ | [PatchWork v5,0/6](https://lore.kernel.org/patchwork/cover/943736) |




# 9 内存控制组 MEMCG (Memory Cgroup)支持
-------

[KS2012: The memcg/mm minisummit](https://lwn.net/Articles/516439)

**2.6.25(2008年4月发布)**

在Linux轻量级虚拟化的实现 container 中(比如现在挺火的Docker, 就是基于container), 一个重要的功能就是做资源隔离. Linux 在 2.6.24中引入了cgroup(control group, 控制组)的资源隔离基础框架(将在最后一个部分详述), 提供了资源隔离的基础.

在2.6.25 中, 内核在此基础上支持了内存资源隔离, 叫内存控制组. 它使用可以在不同的控制组中, 实施内存资源控制, 如分配, 内存用量, 交换等方面的控制.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2010/09/01 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg: towards I/O aware memcg v7.](https://lore.kernel.org/patchwork/cover/213968) | IO 感知的 MEMCG. | v7 ☐ | [PatchWork v7](https://lore.kernel.org/patchwork/cover/213968) |

## 9.1 PageCacheLimit
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/01/15 | Roy Huang <royhuang9@gmail.com> | [Provide an interface to limit total page cache.](https://lore.kernel.org/patchwork/cover/72078) | 限制 page cache 的内存占用. | RFC ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/72078) |
| 2007/01/24 | Christoph Lameter <clameter@sgi.com> | [Limit the size of the pagecache](https://lore.kernel.org/patchwork/cover/72581) | 限制 page cache 的内存占用. | RFC ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/72581) |
| 2014/06/16 | Xishi Qiu <qiuxishi@huawei.com> | [mm: add page cache limit and reclaim feature](https://lore.kernel.org/patchwork/cover/473535) | 限制 page cache 的内存占用. | v1 ☐ | [PatchWork](https://lore.kernel.org/patchwork/cover/416962)<br>*-*-*-*-*-*-*-* <br>[openEuler 4.19](https://gitee.com/openeuler/kernel/commit/6174ecb523613c8ed8dcdc889d46f4c02f65b9e4) |
| 2019/02/23 | Chunguang Xu <brookxu@tencent.com> | [pagecachelimit: limit the pagecache ratio of totalram](http://github.com/tencent/TencentOS-kernel/commit/6711b34671bc658c3a395d99aafedd04a4ebbd41) | 限制 page cache 的内存占用. | NA |  [pagecachelimit: limit the pagecache ratio of totalram](http://github.com/tencent/TencentOS-kernel/commit/6711b34671bc658c3a395d99aafedd04a4ebbd41) |

## 9.2 Cgroup-Aware OOM killer
-------

https://lwn.net/Articles/317814
https://lwn.net/Articles/761118

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/03/17 | Roman Gushchin <guro@fb.com> | [add support for reclaiming priorities per mem cgroup](https://lore.kernel.org/patchwork/cover/771278) | 设置 CGROUP 的内存优先级, Kswapd 等优先扫描低优先级 CGROUP 来回收内存. 用于优化 Android 上的内存回收和 OOM 等机制 | v13 ☐ | [PatchWork v7](https://lore.kernel.org/patchwork/cover/771278), [LKML](https://lkml.org/lkml/2017/3/17/658) |
| 2017/09/04 | Tim Murray <timmurray@google.com> | [cgroup-aware OOM killer](https://lore.kernel.org/patchwork/cover/828043) | Cgroup 感知的 OOM, 通过优先级限定 OOM 时杀进程的次序 | v13 ☐ | [PatchWork v7](https://lore.kernel.org/patchwork/cover/828043) 带 oom_priority<br>*-*-*-*-*-*-*-* <br>[PatchWork v13](https://lore.kernel.org/patchwork/cover/828043) |
| 2018/03/16 | David Rientjes <rientjes@google.com> | [rewrite cgroup aware oom killer for general use](https://lore.kernel.org/patchwork/cover/828043) | Cgroup 感知的 OOM, 通过优先级限定 OOM 时杀进程的次序 | v13 ☐ | [PatchWork v1](https://lore.kernel.org/patchwork/cover/934536) |
| 2021/04/14 | Yulei Zhang <yuleixzhang@tencent.com> | [introduce new attribute "priority" to control group](https://lore.kernel.org/patchwork/cover/828043) | Cgroup 感知的 OOM, 通过优先级限定 OOM 时杀进程的次序 | v13 ☐ | [PatchWork v1](https://lwn.net/Articles/851649)<br>*-*-*-*-*-*-*-* <br>[LWN](https://lwn.net/Articles/852378) |
| 2021/03/25 | Ybrookxu | [bfq: introduce bfq.ioprio for cgroup](https://lore.kernel.org/patchwork/cover/828043) | Cgroup 感知的 bfq.ioprio | v3 ☐ | [LKML](https://lkml.org/lkml/2021/3/25/93) |


## 9.3 kmemcg
-------

[memcg kernel memory tracking](https://lwn.net/Articles/482777)

```cpp
git://git.kernel.org/pub/scm/linux/kernel/git/glommer/memcg.git kmemcg-stac
```

2012-04-20 [slab+slub accounting for memcg 00/23](https://lore.kernel.org/patchwork/cover/298625)
2012-05-11 [kmem limitation for memcg (v2,00/29)](https://lore.kernel.org/patchwork/cover/302291)
2012-05-25 [kmem limitation for memcg (v3,00/28)](https://lore.kernel.org/patchwork/patch/304994)
2012-06-18 [kmem limitation for memcg (v4,00/25)](https://lore.kernel.org/patchwork/cover/308973)
2012-06-25 [kmem controller for memcg: stripped down version](https://lore.kernel.org/patchwork/cover/310278)
2012-09-18 [ kmem controller for memcg (v3,00/13)](https://lore.kernel.org/patchwork/cover/326975)
2012-10-08 [kmem controller for memcg (v4,00/14)](https://lore.kernel.org/patchwork/cover/331578)
2012-10-16 [kmem controller for memcg (v5,00/14](https://lore.kernel.org/patchwork/cover/333535)

```cpp
git://github.com/glommer/linux.git kmemcg-slab
```

2021-07-25 [memcg kmem limitation - slab.](https://lore.kernel.org/patchwork/cover/315827)
2012-08-09 [Request for Inclusion: kmem controller for memcg (v2, 00/11)](https://lore.kernel.org/patchwork/cover/318976)
2012-09-18 [slab accounting for memcg (v3,00/16)](https://lore.kernel.org/patchwork/cover/326976)
2012-10-19 [slab accounting for memcg (v5,00/18)](https://lore.kernel.org/patchwork/cover/334479)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2012/03/09 |  Suleiman Souhlal <ssouhlal@FreeBSD.org> | [Memcg Kernel Memory Tracking](https://lwn.net/Articles/485593) | memcg 支持对内核 STACK 进行统计 | v2 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/cover/291219) |
| 2012/10/16 | Glauber Costa <glommer@parallels.com> | [kmemcg-stack](https://lwn.net/Articles/485593) | memcg 支持对内核 STACK 进行统计 | v5 ☐ | [PatchWork v5](https://lore.kernel.org/patchwork/cover/333535) |
| 2012/10/19 | Glauber Costa <glommer@parallels.com> | [kmemcg-stack](https://lwn.net/Articles/485593) | memcg 支持对内核 SLAB 进行统计 | v5 ☐ | [PatchWork v5](https://lore.kernel.org/patchwork/cover/334479) |
| 2015/11/10 | Vladimir Davydov <vdavydov@virtuozzo.com> | [memcg/kmem: switch to white list policy](https://lore.kernel.org/patchwork/cover/616606/) | NA | v2 ☑ 4.5-rc1 | [PatchWork v2,0/6](https://lore.kernel.org/patchwork/cover/616606) |
| 2020/06/23 | Glauber Costa <glommer@parallels.com> | [kmem controller for memcg](https://lwn.net/Articles/516529) | memcg 支持对内核内存(kmem)进行统计, memcg 开始支持统计两种类型的内核内存使用 : 内核栈 和 slab. 这些限制对于防止 fork 炸弹(bombs)等事件很有用. | v6 ☑ 3.8-rc1 | [PatchWork v5 kmem(stack) controller for memcg](https://lore.kernel.org/patchwork/cover/333535), [PatchWork v5 slab accounting for memcg](https://lore.kernel.org/patchwork/cover/334479)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6](https://lore.kernel.org/patchwork/cover/337780) |
| 2020/06/23 |  Roman Gushchin <guro@fb.com> | [The new cgroup slab memory controller](https://lore.kernel.org/patchwork/cover/1261793) | 将 SLAB 的统计跟踪统计从页面级别更改为到对象级别. 它允许在 memory cgroup 之间共享 SLAB 页. 这一变化消除了每个 memory cgroup 每个重复的每个 cpu 和每个节点 slab 缓存集, 并为所有内存控制组建立了一个共同的每个cpu和每个节点slab缓存集. 这将显著提高 SLAB 利用率(最高可达45%), 并相应降低总的内核内存占用. 测试发现不可移动页面数量的减少也对内存碎片产生积极的影响. | v7 ☑ 5.9-rc1 | [PatchWork v7](https://lore.kernel.org/patchwork/cover/1261793) |
| 2015/11/10 | Roman Gushchin <guro@fb.com> | [memcg/kmem: switch to white list policy](https://lore.kernel.org/patchwork/cover/616606) | 所有的 kmem 分配(即每次 kmem_cache_alloc、kmalloc、alloc_kmem_pages调用)都会自动计入内存 cgroup. 如果出于某些原因, 呼叫者必须明确选择退出. 这样的设计决策会导致以下许多个问题, 因此该补丁切换为白名单策略. 现在 kmalloc 用户必须通过传递 __GFP_ACCOUNT 标志来显式标记. | v7 ☑ 5.9-rc1 | [PatchWork v7](https://lore.kernel.org/patchwork/cover/616606) |
| 2021/05/06 |  Waiman Long <longman@redhat.com> | [The new cgroup slab memory controller](https://lore.kernel.org/patchwork/cover/1422112) | [The new cgroup slab memory controller](https://lore.kernel.org/patchwork/cover/1261793) 合入后, 不再需要为每个 MEMCG 使用单独的 kmemcache, 从而减少了总体的内核内存使用. 但是, 我们还为每次调用 kmem_cache_alloc() 和 kmem_cache_free() 增加了额外的内存开销. 参见 [10befea91b:  hackbench.throughput -62.4% regression](https://lore.kernel.org/lkml/20210114025151.GA22932@xsang-OptiPlex-9020) 和 [memcg: performance degradation since v5.9](https://lore.kernel.org/linux-mm/20210408193948.vfktg3azh2wrt56t@gabell/T/#u). 这组补丁通过降低了 kmemcache 统计的开销. 缓解了这个问题. | v7 ☑ 5.9-rc1 | [PatchWork v7](https://lore.kernel.org/patchwork/cover/1422112) |


## 9.4 memcg LRU list
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/12/27 | Kamezawa Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [per-zone and reclaim enhancements for memory controller take 3](https://lore.kernel.org/patchwork/cover/98042) | per-zone LRU for memcg, 其中引入了 mem_cgroup_per_zone, mem_cgroup_per_node 等结构 | v7 ☑ 2.6.25-rc1 | [PatchWork v7](https://lore.kernel.org/patchwork/cover/98042) |
| 2011/12/08 | Johannes Weiner <jweiner@redhat.com> | [memcg naturalization -rc5](https://lore.kernel.org/patchwork/cover/273527) | 引入 per-memcg lru, 消除重复的 LRU 列表, 全球 LRU 不再存在, page 只存在于 per-memcg LRU list 中.<br>该补丁引入了 lruvec 结构 | v5 ☑ [3.3-rv1](https://kernelnewbies.org/Linux_3.3#Memory_management) | [PatchWork v5](https://lore.kernel.org/patchwork/cover/273527), [LWN](https://lwn.net/Articles/443241) |
| 2012/02/20 | Hugh Dickins <hughd@google.com> | [mm/memcg: per-memcg per-zone lru locking](https://lore.kernel.org/patchwork/cover/288055) | per-memcg lru lock | v1 ☐ | [PatchWork v1](https://lore.kernel.org/patchwork/cover/288055) |
| 2020/12/05 | Alex Shi <alex.shi@linux.alibaba.com> | [per memcg lru lock](https://lore.kernel.org/patchwork/cover/1333353) | per memcg LRU lock | v21 ☑ [5.11](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork v21](https://lore.kernel.org/patchwork/cover/1333353) |




## 9.5 MEMCG DRITY Page
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/04/17 | Greg Thelen <gthelen@google.com> | [memcg: per cgroup dirty page limiting](https://lore.kernel.org/patchwork/cover/263368) | per-memcg 的脏页带宽限制 | v9 ☐ | [PatchWork v9](https://lore.kernel.org/patchwork/cover/263368) |
| 2015/12/30 | Tejun Heo <tj@kernel.org> | [memcg: add per cgroup dirty page accounting](https://lore.kernel.org/patchwork/cover/558382) | madvise 支持页面延迟回收(MADV_FREE)的早期尝试  | v5 ☑ [4.2-rc1](https://kernelnewbies.org/Linux_4.2#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/cover/558382), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c4843a7593a9df3ff5b1806084cefdfa81dd7c79) |


## 9.6 memcg swapin
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/07/04 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg: shmem swap cache](https://lore.kernel.org/patchwork/cover/121475) | memcg swapin 的延迟统计, 对memcg进行了修改, 使其在交换时直接对交换页统计, 而不是在出错时统计, 这可能要晚得多, 或者根本不会发生. | v1 ☑ 2.6.26-rc8-mm1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/121270)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/patch/121475)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/https://lore.kernel.org/patchwork/patch/121475), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d13d144309d2e5a3e6ad978b16c1d0226ddc9231) |
| 2008/07/14 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg: handle tmpfs' swapcache](https://lore.kernel.org/patchwork/cover/122556) | memcg swapin 的延迟统计, 对memcg进行了修改, 使其在交换时直接对交换页统计, 而不是在出错时统计, 这可能要晚得多, 或者根本不会发生. | v1 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/patch/122556)) |
| 2008/11/14 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg : add swap controller](https://lore.kernel.org/patchwork/cover/134928) | NA | v1 ☑ 2.6.29-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/134928)) |
| 2009/06/02 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg fix swap accounting](https://lore.kernel.org/patchwork/cover/158520) | NA | v1 ☑ 2.6.29-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/157997)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/patch/158520)) |
| 2009/06/02 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [mm rss counting updates](https://lore.kernel.org/patchwork/cover/182191) | NA | v1 ☑ 2.6.29-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/patch/182191)) |
| 2015/12/17 | Vladimir Davydov <vdavydov@virtuozzo.com> | [Add swap accounting to cgroup2](https://lore.kernel.org/patchwork/cover/628754) | NA | v2 ☑ 2.6.29-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/628754)) |
| 2020/08/20 | Johannes Weiner <hannes@cmpxchg.org> | [mm: memcontrol: charge swapin pages on instantiation](https://lore.kernel.org/patchwork/cover/1239175) | memcg swapin 的延迟统计, 对memcg进行了修改, 使其在交换时直接对交换页统计, 而不是在出错时统计, 这可能要晚得多, 或者根本不会发生. | v2 ☑ [5.8-rc1](https://kernelnewbies.org/Linux_5.8#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/cover/1227833), [PatchWork v2](https://lore.kernel.org/patchwork/cover/1239175) |

# 10 内存热插拔支持
-------




内存热插拔, 也许对于第一次听说的人来说, 觉得不可思议: 一个系统的核心组件, 为何要支持热插拔? 用处有以下几点.



> 1\. 大规模集群中, 动态的物理容量增减, 可以实现更好地支持资源集约和均衡.
> 2\. 大规模集群中, 物理内存出错的机会大大增多, 内存热插拔技术对提高高可用性至关重要.
> 3\. 在虚拟化环境中, 客户机(Guest OS)间的高效内存使用也对热插拔技术提出要求



当然, 作为一个核心组件, 内存的热插拔对从系统固件,到软件(操作系统)的要求, 跟普通的外设热插拔的要求, 不可同日而语. 这也是为什么 Linux 内核对内存热插拔的完全支持一直到近两年才基本完成.



总的来说, 内存热插拔分为两个阶段, 即**物理热插拔阶段**和**逻辑热插拔阶段:**

> **物理热插拔阶段:** 这一阶段是内存条插入/拔出主板的过程. 这一过程必须要涉及到固件的支持(如 ACPI 的支持), 以及内核的相关支持, 如为新插入的内存分配管理元数据进行管理. 我们可以把这一阶段分别称为 hot-add / hot-remove.
> **逻辑热插拔阶段:** 这一阶段是从使用者视角, 启用/关闭这部分内存. 这部分的主要从内存分配器方面做的一些准备工作. 我们可以把这一阶段分别称为 online / offline.



逻辑上来讲, 内存插和拔是一个互为逆操作, 内核应该做的事是对称的, 但是, 拔的过程需要关注的技术难点却比插的过程多, 因为, 从无到有容易, 从有到无麻烦:在使用的内存页应该被妥善安置, 就如同安置拆迁户一样, 这是一件棘手的事情. 所以内核对 hot-remove 的完全支持一直推迟到 2013 年.



## 10.1 内存热插入支持
-------

**2.6.15(2006年1月发布)**


这提供了最基本的内存热插入支持(包括物理/逻辑阶段). 注意, **此时热拔除还不支持.**





## 10.2 初步的内存逻辑热拔除支持
-------

**2.6.24(2008年1月发布)**




此版本提供了**部分的逻辑热拔除阶段的支持, 即 offline.** Offline 时, 内核会把相关的部分内存隔离开来, 使得该部分内存不可被其他任何人使用, 然后再把此部分内存页, 用前面章节说过的内存页迁移功能转移到别的内存上. 之所以说**部分支持**, 是因为该工作只提供了一个 offline 的功能. 但是, 不是所有的内存页都可以迁移的. 考虑"迁移"二字的含义, 这意味着**物理内存地址**会变化, 而内存热拔除应该是对使用者透明的, 这意味着, 用户见到的**虚拟内存地址**不能变化, 所以这中间必须存在一种机制, 可以修改这种因迁移而引起的映射变化. 所有通过页表访问的内存页就没问题了, 只要修改页表映射即可; 但是, 内核自己用的内存, 由于是绕过寻常的逐级页表机制, 采用直接映射(提高了效率), 内核内存页的虚拟地址会随着物理内存地址变动, 因此, 这部分内存页是无法轻易迁移的. 所以说, 此版本的逻辑热拔除功能只是部分完成.



注意, 此版本中, **物理热拔除是还完全未实现. 一句话, 此版本的热拔除功能还不能用.**





## 10.3 完善的内存逻辑热拔除支持
-------

**3.8(2013年2月发布)**




针对9.2中的问题, 此版本引入了一个解决方案. 9.2 中的核心问题在于**不可迁移的页会导致内存无法被拔除.** 解决问题的思路是使可能被热拔除的内存不包含这种不可迁移的页. 这种信息应该在内存初始化/内存插入时被传达, 所以, 此版本中, 引入一个 movable\_node 的概念. 在此概念中, 一个被 movable\_node 节点的所有内存, 在初始化/插入后, 内核确保它们之上不会被分配有不可迁移的页, 所以当热拔除需求到来时, 上面的内存页都可以被迁移, 从而使内存可以被拔除.





## 10.4 物理热拔除的支持
-------


**3.9(2013年4月支持)**

此版本支持了**物理热拔除,** 这包括对内存管理元数据的删除, 跟固件(如ACPI) 相关功能的实现等比较底层琐碎的细节, 不详谈.



在完成这一步之后, 内核已经可以提供基本的内存热插拔支持. 值得一提的是, 内存热插拔的工作, Fujitsu 中国这边的内核开放者贡献了很多 patch. 谢谢他们!



# 11 超然内存(Transcendent Memory)支持
-------


## 11.1 tmem 简介
-------

[超然内存(Transcendent Memory)](https://lwn.net/Articles/338098), 对很多第一次听见这个概念的人来说, 是如此的奇怪和陌生. 这个概念是在 Linux 内核开发者社区中首次被提出的. 超然内存(后文一律简称为[**tmem**](https://lore.kernel.org/patchwork/cover/161314))之于普通内存, 不同之处在于以下几点:


1.  tmem 的大小对内核来说是未知的, 它甚至是可变的; 与之相比, 普通内存在系统初始化就可被内核探测并枚举, 并且大小是固定的(不考虑热插拔情形).

2.  tmem 可以是稳定的, 也可以是不稳定的, 对于后者, 这意味着, 放入其中的数据, 在之后的访问中可能发现不见了; 与之相比, 普通内存在系统加电状态下一直是稳定的, 你不用担心写入的数据之后访问不到(不考虑内存硬件故障问题).

3.  基于以上两个原因, tmem 无法被内核直接访问, 必须通过定义良好的 API 来访问 tmem 中的内存; 与之相比, 普通内存可以通过内存地址被内核直接访问.


初看之下, [tmem](https://lore.kernel.org/patchwork/cover/163284) 这三点奇异的特性, 似乎增加了不必要的复杂性, 尤其是第二条, 更是诡异无用处.

计算机界有句名言: 计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决. tmem 增加的这些间接特性正是为了解决某些问题.


考虑虚拟化环境下, 虚拟机管理器(hypervisor) 需要管理维护各个虚拟客户机(Guest OS)的内存使用. 在常见的使用环境中, 我们新建一台虚拟机时, 总要事先配置其可用的内存大小. 然而这并不十分明智, 我们很难事先确切知道每台虚拟机的内存需求情况, 这难免造成有些虚拟机内存富余, 而有些则捉襟见肘. 如果能平衡这些资源就好了. 事实上, 存在这样的技术, hypervisor 维护一个内存池子, 其中存放的就是这些所谓的 tmem, 它能动态地平衡各个虚拟机的盈余内存, 更高效地使用. 这是 tmem 概念提出的最初来由.



再考虑另一种情形, 当一台机器内存使用紧张时, 它会把一些精心选择的内存页面写到交换外设中以腾出空间. 然而外设与内存存在着显著的读写速度差异, 这对性能是一个不利的影响. 如果在一个超高速网络相连的集群中, 网络访问速度比外设访问速度快, 可不可以有别的想法? tmem 又可以扮演一个重要的中间角色, 它可以把集群中所有节点的内存管理在一个池子里, 从而动态平衡各节点的内存使用, 当某一节点内存紧张时, 写到交换设备页其实是被写到另一节点的内存里...



还有一种情形, 旨在提高内存使用效率.. 比如大量内容相同的页面(全0页面)在池子里可以只保留一份; 或者, tmem 可以考虑对这些页面进行压缩, 从而增加有效内存的使用.



Linux 内核 从 3.X 系列开始陆续加入 tmem 相关的基础设施支持, 并逐步加入了关于内存压缩的功能. 进一步讨论内核中的实现前, 需要对这一问题再进一步细化, 以方便讨论细节.



前文说了内核需要通过 API 访问 tmem, 那么进一步, 可以细化为两个问题.

1.  内核的哪些部分内存可以被 tmem 管理呢? 有两大类, 即前面提到过的文件缓存页和匿名页.

2.  tmem 如何管理其池子中的内存. 针对前述三种情形, 有不同的策略. Linux 现在的主要解决方案是针对内存压缩, 提高内存使用效率.



针对这两个问题, 可以把内核对于 tmem 的支持分别分为**前端**和**后端**. 前端是内核与 tmem 通讯的接口; 而后端则实现 tmem 的管理策略.


## 11.1 tmem 前端
-------

### 11.1.1 前端接口之 CLEANCACHE
-------

**3.0(2011年7月发布)**


前面章节提到过, 内核会利用空闲内存缓存后备外设中的内容, 以期在近期将要使用时不用从缓慢的外设读取. 这些内容叫文件缓存页. 它的特点就是只要是页是干净的(没有被写过), 那么在系统需要内存时, 随时可以直接丢弃这些页面以腾出空间(因为它随时可以从后备文件系统中读取).



然而, 下次内核需要这些文件缓存页时, 又得从外设读取. 这时, tmem 可以起作用了. 内核丢弃时,假如这些页面被 tmem 接收并管理起来, 等内核需要的时候, tmem 把它归还, 这样就省去了读写磁盘的操作, 提高了性能. 3.0 引入的 CLEANCACHE, 作为 tmem 前端接口, hook 进了内核丢弃这些干净文件页的地方, 把这些页面截获了, 放进 tmem 后端管理(后方讲). 从内核角度看, 这个 CLEANCACHE 就像一个魔盒的入口, 它丢弃的页面被吸入这个魔盒, 在它需要时, 内核尝试从这个魔盒中找, 如果找得到, 搞定; 否则, 它再去外设读取. 至于为何被吸入魔盒的页面为什么会找不到, 这跟前文说过的 tmem 的特性有关. 后文讲 tmem 后端时再说.



### 11.1.2 前端接口之 FRONTSWAP
-------

**3.5(2012年7月发布)**


除了文件缓存页, 另一大类内存页面就是匿名页. 在系统内存紧张时, 内核必须要把这些页面写出到外设的交换设备或交换分区中, 而不能简单丢弃(因为这些页面没有后备文件系统). 同样, 涉及到读写外设, 又有性能考量了, 此时又是 tmem 起作用的时候了.



同样, FRONTSWAP 这个前端接口, 正如其名字一样, 在 内核 swap 路径的前面, 截获了这些页面, 放入 tmem 后端管理. 如果后端还有空闲资源的话, 这些页面被接收, 在内核需要这些页面时, 再把它们吐出来; 如果后端没有空闲资源了, 那么内核还是会把这些页面按原来的走 swap 路径写到交换设备中.



## 11.2 后端
-------

### 11.2.1 后端之 ZCACHE
-------


**(没能进入内核主线)**

讲完了两个前端接口, 接下来说后端的管理策略. 对应于 CLEANCACHE, 一开始是有一个专门的后端叫 [zcache](https://lwn.net/Articles/397574), **不过最后被删除了.** 它的做法就是把这些被内核逐出的文件缓存页压缩, 并存放在内存中. 所以, zcache 相当于把内存页从内存中的一个地方移到另一个地方, 这乍一看, 感觉很奇怪, 但这正是 tmem 的灵活性所在. 它允许后端有不同的管理策略, 比如在这个情况下, 它把内存页压缩后仍然放在内存中, 这提高了内存的使用. 当然, 毕竟 zcache 会占用一部分物理内存, 导致可用的内存减小. 因此, 这需要有一个权衡. 高效(压缩比, 压缩时间)的压缩算法的使用, 从而使更多的文件页待在内存中, 使得其带来的避免磁盘读写的优势大于减少的这部分内存的代价. 不过, 也因为如此, 它的实现过于复杂, **以至最终没能进入内核主线.** 开发者在开始重新实现一个新的替代品, 不过截止至 4.2 , 还没有看到成果.



### 11.2.2 后端之 ZRAM
-------

**3.14(2014年3月发布)**



FRONTSWAP 对应的一个后端叫 ZRAM. 前身为 compcache, 它早在 2.6.33(2010年2月发布) 时就已经进入内核的 staging 分支经历了 4 年的 staging 后, 于内核 3.14 版本进入 mainline(2014年3月), 目前是谷歌 Android 系统(自4.4版本)和 Chrome OS(自2013年)使用的默认方案.

> Staging 分支是在内核源码中的一个子目录, 它是一个独立的分支, 主要维护着独立的 driver 或文件系统, 这些代码未来可能也可能不进入主线.


ZRAM 是一个在内存中的块设备(块设备相对于字符设备而言, 信息存放于固定大小的块中, 支持随机访问, 磁盘就是典型的块设备, 更多将在块层子系统中讲), 因此, 内核可以复用已有的 swap 设备设施, 把这个块设备格式化为 swap 设备. 因此, 被交换出去的页面, 将通过 FRONTSWAP 前端进入到 ZRAM 这个伪 swap 设备中, 并被压缩存放! 当然, 这个ZRAM 空间有限, 因此, 页面可能不被 ZRAM 接受. 如果这种情形发生, 内核就回退到用真正的磁盘交换设备.



### 11.2.3 后端之 ZSWAP
-------

**3.11(2013年9月发布)**


FRONTSWAP 对应的另一个后端叫 [ZSWAP](https://lwn.net/Articles/537422). ZSWAP 的做法其实也是尝试把内核交换出去的页面压缩存放到一个内存池子中. 当然, ZSWAP 空间也是有限的. 但同 ZRAM 不同的是, ZSWAP 会智能地把其中一些它认为近期不会使用的页面解压缩, 写回到真正的磁盘外设中. 因此, 大部分情况下, 它能避免磁盘写操作, 这比 ZRAM 不知高明到哪去了.



## 11.3 一些细节
-------


这一章基本说完了, 但牵涉到后端, 其实还有一些细节可以谈, 比如对于压缩的效率的考量, 会影响到后端实现的选择, 比如不同的内存页面的压缩效果不同(全0页和某种压缩文件占据的内存页的压缩效果显然差距很大)对压缩算法的选择; 压缩后页面的存放策略也很重要, 因为以上后端都存在特殊情况要把页面解压缩写回到磁盘外设, 写回页面的选择与页面的存放策略关系很大. 但从用户角度讲, 以上内容足以, 就不多写了.



关于 tmem, lwn 上的两篇文章值得关注技术细节的人一读:

[Transcendent memory in a nutshell [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/454795)

[In-kernel memory compression [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/545244)



# 12 非易失性内存 (NVDIMM, Non-Volatile DIMM) 支持
-------



[Linux Kernel中AEP的现状和发展](https://kernel.taobao.org/2019/05/NVDIMM-in-Linux-Kernel)


计算机的存储层级是一个金字塔体系, 从塔尖到塔基, 访问速度递减, 而存储容量递增. 从访问速度考量, 内存(DRAM)与磁盘(HHD)之间, 存在着[显著的差异(可达到10^5级别)](https://www.directionsmag.com/article/3794). 因此, 基于内存的缓存技术一直都是系统软件或数据库软件的重中之重. 即使近些年出现的新兴的最快的基于PCIe总线的SSD, 这中间依然存在着鸿沟.



![](https://pic4.zhimg.com/50/0c0850cde43c84764e65bc24942bc6d3_hd.jpg)




另一方面, 非可易失性内存也并不是新鲜产物. 然而实质要么是一块DRAM, 后端加上一块 NAND FLASH 闪存, 以及一个超级电容, 以在系统断电时的提供保护; 要么就是一块简单的 NAND FLASH, 提供类似 SSD 一样的存储特性. 所有这些, 从访问速度上看, 都谈不上真正的内存, 并且, NAND FLASH 的物理特性, 使其免不了磨损(wear out); 并且在长时间使用后, 存在写性能下降的问题.



2015 年算得上闪存技术革命年. 3D NAND FLASH 技术的创新, 以及 Intel 在酝酿的完全不同于NAND 闪存技术的 3D XPoint 内存, 都将预示着填充上图这个性能鸿沟的时刻的临近. 它们不仅能提供更在的容量(TB级别), 更快的访问速度(3D XPoint 按 Intel 说法能提供 `~1000` 倍快于传统的 NAND FLASH, 5 - 8 倍慢于 DRAM 的访问速度), 更持久的寿命.



相应的, Linux 内核也在进行相应的功能支持.



## 12.1 NVDIMM 支持框架
-------

** libnvdimm 4.2(2015年8月30日发布)**

2015 年 4 月发布的 [ACPI 6.0 规范](https://uefi.org/sites/default/files/resources/ACPI/_6.0.pdf](https://link.zhihu.com/?target=http%3A//www.uefi.org/sites/default/files/resources/ACPI_6.0.pdf), 定义了NVDIMM Firmware Interface Table (NFIT), 详细地规定了 NVDIMM 的访问模式, 接口数据规范等细节. 在 Linux 4.2 中, 内核开始支持一个叫 libnvdimm 的子系统, 它实现了 NFIT 的语义, 提供了对 NVDIMM 两种基本访问模式的支持, 一种即内核所称之的 PMEM 模式, 即把 NVDIMM 设备当作持久性的内存来访问; 另一种则提供了块设备模式的访问. 开始奠定 Linux 内核对这一新兴技术的支持.





## 12.2 DAX
-------


**4.0(2015年4月发布)**



与这一技术相关的还有另外一个特性值得一提, 那就是 DAX(Direct Access, 直接访问, X 无实义, 只是为了酷).



传统的基于磁盘的文件系统, 在被访问时, 内核总会把页面通过前面所提的文件缓存页(page cache)的缓存机制, 把文件系统页从磁盘中预先加载到内存中, 以提速访问. 然后, 对于新兴的 NVDIMM 设备, 基于它的非易失特性, 内核应该能直接访问基于此设备之上的文件系统的内容, 它使得这一拷贝到内存的操作变得不必要. 4.0 开始引入的 DAX 就是提供这一支持. 截至 4.3, 内核中已经有 XFS, EXT2, EXT4 这几个文件系统实现这一特性.

## 12.3 NUMA nodes for persistent-memory management
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/02/25 | Dave Hansen <dave.hansen@linux.intel.com><br>Huang Ying <ying.huang@intel.com> | [Allow persistent memory to be used like normal RAM](https://lore.kernel.org/patchwork/cover/1045596) | 通过memory hotplug的方式把PMEM添加到Linux的buddy allocator里面. 新添加的PMEM会以一个或多个NUMA node的形式出现, Linux Kernel就可以分配PMEM上的memory, 这样和使用一般DRAM没什么区别 | v5 ☑ 5.1-rc1 | [PatchWork v5,0/5](https://lore.kernel.org/patchwork/cover/1045596) |
| 2018/12/26 | Fengguang Wu <fengguang.wu@intel.com> | [PMEM NUMA node and hotness accounting/migration](https://lore.kernel.org/patchwork/cover/1027864) | 1. 隔离DRAM和PMEM. 为PMEM单独构造了一个zonelist, 这样一般的内存分配是不会分配到PMEM上的<br>2. 跟踪内存的冷热. 利用内核中已经有的 idle page tracking 功能(目前主线内核只支持系统全局的tracking), 在per process的粒度上跟踪内存的冷热 <br>3. 利用现有的page reclaim, 在 reclai m时将冷内存迁移到 PMEM 上(只能迁移匿名页).  <br>4. 利用一个 userspace 的 daemon 和 idle page tracking, 来将热内存(在PMEM上的)迁移到 DRA M中. | RFC v2 ☐ 4.20 | PatchWork RFC,v2,00/21](https://lore.kernel.org/patchwork/cover/1027864), [LKML](https://lkml.org/lkml/2018/12/26/138), [github/intel/memory-optimizer](http://github.com/intel/memory-optimizer) |
| 2019/04/11 | Fengguang Wu <fengguang.wu@intel.com> | [Another Approach to Use PMEM as NUMA Node](https://patchwork.kernel.org/project/linux-mm/cover/1554955019-29472-1-git-send-email-yang.shi@linux.alibaba.com) | 通过 memory reclaim 把"冷" 内存迁移到慢速的 PMEM node 中, NUMA Balancing 访问到这些冷 page 的时候可以选择是否把这个页迁移回 DRAM, 相当于是一种比较粗粒度的"热"内存识别. | RFC v2 ☐ 4.20 | PatchWork v2,RFC,0/9](https://patchwork.kernel.org/project/linux-mm/cover/1554955019-29472-1-git-send-email-yang.shi@linux.alibaba.com) |



## 12.4 Top-tier memory management
-------

[Top-tier memory management](https://lwn.net/Articles/857133)
[LWN: Top-tier memory management!](https://blog.csdn.net/Linux_Everything/article/details/117970246)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/06/25 | Dave Hansen <dave.hansen@linux.intel.com><br>Huang Ying <ying.huang@intel.com> | [Migrate Pages in lieu of discard](https://lore.kernel.org/patchwork/cover/1393431) | 页面回收阶段引入页面降级(demote pages)策略. 在一个具备了持久性内存的系统中, 可以把一些需要回收的 page 从 DRAM 迁移到较慢的 memory 中, 后面如果再次需要这些数据了, 也是仍然可以直接访问到, 只是速度稍微慢一些. 目前的版本还不完善, 被迁移的 page 将被永远困在慢速内存区域中, 没有机制使其回到更快的 DRAM. 这个降级策略可以通过 sysctl 的 vm.zone_reclaim_mode 中把 bitmask 设置为 8 从而启用这个功能. | v9 ☐ 5.13 | [PatchWork V9,0/9](https://lore.kernel.org/patchwork/cover/1393431) |
| 2021/03/11 | Huang Ying <ying.huang@intel.com> | [NUMA balancing: optimize memory placement for memory tiering system](https://lore.kernel.org/patchwork/cover/1393431) | 将经常使用的 page 从慢速内存迁移到快速内存的方案. 优化 numa balancing 的页面迁移策略, 利用了这些 numa fault 来对哪些 page 属于常用 page 进行更准确地估算. 新的策略依据从 page unmmap 到发生 page fault 之间的时间差来判断 page 是否常用, 并提供了一个 sysctl 开关来定义阈值: kernel.numa_balancing_hot_threshold_ms. 所有 page fault 时间差低于阈值的 page 都被判定是常用 page. 由于对于系统管理员来说可能很难决定这个阈值应该设置成什么值, 所以这组 patch 中的实现了自动调整的方法. 为了实现这个自动调整, kernel 会根据用户设置的平衡速率限制(balancing rate limit). 内核会对迁移的 page 数量进行监控, 通过增加或减少阈值来使 balancing rate 更接近该目标值. | RFC v6 ☐ 5.13 | [PatchWork RFC,-V6,0/6](https://lore.kernel.org/patchwork/cover/1393431) |
| 2021/04/15 | Tim Chen <tim.c.chen@linux.intel.com>> | [Manage the top tier memory in a tiered memory](https://lore.kernel.org/patchwork/cover/1408180) |  memory tiers 的配置管理. 监控系统和每个 cgroup 中各自使用的 top-tier 内存的数量. 当前使用了 soft limit, 用 kswapd 来把某个 cgroup 中超过 soft limit 限制的 page 迁移到较慢的 memory 类型上去. 这里所说的 soft limit, 是指如果 top-tier memory 很充足的话, cgroup 可以拥有超过此限制的 page 数量, 但如果资源紧张的话则会被迅速削减从而满足这个 limit 值. 后期可用于对于不同的任务划分快、慢内存(fast and slow memory). 即让高优先级的任务获得更多 top-tier memory 访问优先, 而低优先级的任务则要受到更严格的限制 | v1 ☐ 5.13 | [PatchWork RFC,v1,00/11](https://lore.kernel.org/patchwork/cover/1408180) |


# 13 内存管理调试支持
-------




由前面所述, 内存管理相当复杂, 代码量巨大, 而它又是如此重要的一个的子系统, 所以代码质量也要求非常高. 另一方面, 系统各个组件都是内存管理子系统的使用者, 而如果缺乏合理有效的约束, 不正当的内存使用(如内存泄露, 内存覆写)都将引起系统的崩溃, 以至于数据损坏. 基于此, 内存管理子系统引入了一些调试支持工具, 方便开发者/用户追踪,调试内存管理及内存使用中的问题. 本章介绍内存管理子系统中几个重要的调试工具.



## 13.1 页分配的调试支持
-------


**2.5(2003年7月之后发布)**

前面提到过, 内核自己用的内存, 由于是绕过寻常的逐级页表机制, 采用直接映射(提高了效率), 即虚拟地址与页面实际的物理地址存在着一一线性映射的关系. 另一方面, 内核使用的内存又是出于各种重要管理目的, 比如驱动, 模块, 文件系统, 甚至 SLAB 子系统也是构建于页分配器之上. 以上二个事实意味着, 相邻的页, 可能被用于完全不同的目的, 而这两个页由于是直接映射, 它们的虚拟地址也是连续的. 如果某个使用者子系统的编码有 bug, 那么它的对其内存页的写操作造成对相邻页的覆写可能性相当大, 又或者不小心读了一个相邻页的数据. 这些操作可能不一定马上引起问题, 而是在之后的某个地方才触发, 导致数据损坏乃至系统崩溃.



为此, 2.5中, 针对 Intel 的 i386 平台, 内核引入了一个 **CONFIG\_DEBUG\_PAGEALLOC** 开关, 它在页分配的路径上插入钩子, 并利用 i386 CPU 可以对页属性进行修改的特性, 通过修改未分配的页的页表项的属性, 把该页置为"隐藏". 因此, 一旦不小心访问该页(读或写), 都将引起处理器的缺页异常, 内核将进入缺页处理过程, 因而有了一个可以检查捕捉这种内存破坏问题的机会.



在2.6.30中, 又增加了对无法作处理器级别的页属性修改的体系的支持. 对于这种体系, 该特性是将未分配的页**毒化(POISON),** 写入特定模式的值, 因而一旦被无意地访问(读或写), 都将可能在之后的某个时间点被发现. **注意, 这种通用的方法就无法像前面的有处理器级别支持的方法有立刻捕捉的机会.**





**当然, 这个特性是有性能代价的, 所以生产系统中可别用哦.**



## 13.2 SLAB 子系统的调试支持
-------


SLAB 作为一个相对独立的子模块, 一直有自己完善的调试支持, 包括有:



- 对已分配对象写的边界超出的检查

- 对未初始化对象写的检查

- 对内存泄漏或多次释放的检查

- 对上一次分配者进行记录的支持等

## 13.3 内存检测工具
-------


### 13.3.1 错误注入机制
-------

**2.6.20(2007年2月发布)**


内核有着极强的健壮性, 能对各种错误异常情况进行合理的处理. 然而, 毕竟有些错误实在是极小可能发生, 测试对这种小概率异常情况的处理的代码实在是不方便. 所以, 2.6.20中内核引入了错误注入机制, 其中跟 MM 相关的有两个, 一个是对页分配器的失败注入, 一个是对 SLAB 对象分配器的失败注入. 这两个注入机制, 可以触发内存分配失败, 以测试极端情况下(如内存不足)系统的处理情况.

### 13.3.2 KMEMCHECK - 内存非法访问检测工具
-------

**2.6.31(2009年9月发布)**

对内存的非法访问, 如访问未分配的内存, 或访问分配了但未初始化的内存, 或访问了已释放了的内存, 会引起很多让人头痛的问题, 比如程序因数据损坏而在某个地方莫名崩溃, 排查非常困难. 在用户态内存检测工具 valgrind 中, 有一个 Memcheck 插件可以检测此类问题. 2.6.31, Linux 内核也引进了内核态的对应工具, 叫 KMEMCHECK.



它是一个内核态工具, 检测的是内核态的内存访问. 主要针对以下问题:

> 1. 对已分配的但未初始化的页面的访问
> 2. 对 SLAB 系统中未分配的对象的访问
> 3. 对 SLAB 系统中已释放的对象的访问

为了实现该功能, 内核引入一个叫**影子页(shadow page)**的概念, 与被检测的正常**数据页**一一相对. 这也意味着启用该功能, 不仅有速度开销, 还有很大的内存开销.



在分配要被追踪的数据页的同时, 内核还会分配等量的影子页, 并通过数据页的管理数据结构中的一个 **shadow** 字段指向该影子页. 分配后, 数据页的页表中的 **present** 标记会被清除, 并且标记为被 KMEMCHECK 跟踪. 在第一次访问时, 由于 **present** 标记被清除, 将触发缺页异常. 在缺页异常处理程序中, 内核将会检查此次访问是不是正常. 如果发生上述的非法访问, 内核将会记录下该地址, 错误类型, 寄存器, 和栈的回溯, 并根据配置的值大小, 把该地址附近的数据页和影子页的对应内容, 一并保存到一个缓冲区中. 并设定一个稍后处理的软中断任务, 把这些内容报告给上层. 所有这些之后, 将 CPU 标志寄存器 TF (Trap Flag)置位, 于是在缺页异常后, CPU 又能重新访问这条指令, 走正常的执行流程.



这里面有个问题是, **present** 标记在第一次缺页异常后将被置位, 之后如果再次访问, KMEMCHECK 如何再次利用该机制来检测呢? 答案是内核在上述处理后还**打开了单步调试功能,** 所以 CPU 接着执行下一条指令前, 又陷入了调试陷阱(debug trap, 详情请查看 CPU 文档), 在处理程序中, 内核又会把该页的 **present** 标记会清除.


## 13.3.3 KMEMLEAK - 内存泄漏检测工具
-------

**2.6.31(2009年9月发布)**

内存漏洞一直是 C 语言用户面临的一个问题, 内核开发也不例外. 2.6.31 中, 内核引入了 [KMEMLEAK 工具](https://lwn.net/Articles/187979), 用以检测内存泄漏. 它采用了标记-清除的垃圾收集算法, 对通过 SLAB 子系统分配的对象, 或通过 _vmalloc_ 接口分配的连续虚拟地址的对象, 或分配的per-CPU对象(per-CPU对象是指每个 CPU 有一份拷贝的全局对象, 每个 CPU 访问修改本地拷贝, 以提高性能)进行追踪, 把指向对象起始的指针, 对象大小, 分配时的栈踪迹(stack trace) 保存在一个红黑树里(便于之后的查找, 同时还会把对象加入一个全局链表中). 之后, KMEMLEAK 会启动一个每10分钟运行一次的内核线程, 或在用户的指令下, 对整个内存进行扫描. 如果某个对象**从其起始地址到终末地址**内没有别的指针指向它, 那么该对象就被当成是泄漏了. KMEMLEAK 会把相关信息报告给用户.



扫描的大致算法如下:



> 1. 首先会把全局链表中的对象加入一个所谓的**白名单**中, 这是所有待查对象. 然后 , 依次扫描数据区(data段, bss段), per-CPU区, 还有针对每个 NUMA 节点的所有页. 另外, 如果用户有指定, 还会描扫所有线程的栈区(之所以这个不是强制扫描, 是因为栈区是函数的活动记录, 变动迅速, 引用可能稍纵即逝). 扫描过程中, 与之前存的红黑树中的对象进行比对, 一旦发现有指针指向红黑树中的对象, 说明该对象仍有人引用 , 没被泄漏, 把它加入一个所谓的**灰名单**中.
> 2. 然后, 再扫描一遍灰名单, 即已经被确认有引用的对象, 找出这些对象可能引用的别的所有对象, 也加入灰名单中.
> 3. 最后剩下的, 在白名单中的, 就是被 KMEMLEAK 认为是泄漏了的对象.



由于内存的引用情况各异, 存在很多特殊情况, 可能存在误报或漏报的情况, 所以 KMEMLEAK 还提供了一些接口, 方便使用者告知 KMEMLEAK 某些对象不是泄露, 某些对象不用检查,等等.



这个工具当然也存在着显著的影响系统性能的问题, 所以也只是作为调试使用.

### 13.3.4 KASan - 内核地址净化器
-------

**4.0(2015年4月发布)**



4.0 引入的 [KASan (The kernel address sanitizer)](https://lwn.net/Articles/612153) 可以看作是 KMEMCHECK 工具的替代器, 它的目的也是为了检测诸如**释放后访问(use-after-free), 访问越界(out-of-bouds)**等非法访问问题. 它比后者更快, 因为它利用了编译器的 **Instrument** 功能, 也即编译器会在访问内存前插入探针, 执行用户指定的操作, 这通常用在性能剖析中. 在内存的使用上, KASan 也比 KMEMCHECK 有优势: 相比后者1:1的内存使用 , 它只要1:1/8.


总的来说, 它利用了 GCC 5.0的新特性, 可对内核内存进行 Instrumentaion, 编译器可以在访问内存前插入指令, 从而检测该次访问是否合法. 对比前述的 KMEMCHECK 要用到 CPU 的陷阱指令处理和单步调试功能, KASan 是在编译时加入了探针, 因此它的性能更快.

ARM 引入了一个[内存标签扩展](https://community.arm.com/developer/ip-products/processors/b/processors-ip-blog/posts/enhancing-memory-safety)的硬件特性. 然后 Google 的 [Andrey Konovalov](https://github.com/xairy/linux)基于此硬件特性重写了 KASan. 实现了 [hardware tag-based mode 的 KASan](https://lore.kernel.org/patchwork/cover/1344197).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/11/23 | Andrey Konovalov <andreyknvl@google.com> | [kasan: add hardware tag-based mode for arm64](https://lore.kernel.org/patchwork/cover/1344197) | NA | v11 ☑ [5.11-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v11,00/42](https://patchwork.kernel.org/project/linux-mm/cover/cover.1606161801.git.andreyknvl@google.com) |
| 2020/11/23 | Andrey Konovalov <andreyknvl@google.com> | [kasan: boot parameters for hardware tag-based mode](https://lore.kernel.org/patchwork/cover/1344258) | NA | v4 ☑ [5.11-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v4,00/19](https://patchwork.kernel.org/project/linux-mm/patch/748daf013e17d925b0fe00c1c3b5dce726dd2430.1606162397.git.andreyknvl@google.com) |
| 2021/01/15 | Andrey Konovalov <andreyknvl@google.com> | [kasan: HW_TAGS tests support and fixes](https://lore.kernel.org/patchwork/cover/1366086) | NA | v4 ☑ [5.12-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v4,00/15](https://patchwork.kernel.org/project/linux-mm/cover/cover.1610733117.git.andreyknvl@google.com) |
| 2021/02/05 | Andrey Konovalov <andreyknvl@google.com> | [kasan: optimizations and fixes for HW_TAGS](https://lore.kernel.org/patchwork/cover/1376340) | NA | v3 ☑ [5.12-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v3,mm,00/13](https://patchwork.kernel.org/project/linux-mm/cover/cover.1612546384.git.andreyknvl@google.com) |


### 13.3.5 KFENCE 一个新的内存安全检测工具
-------

5.12 引入一个[新的内存错误检测工具 : KFENCE (Kernel Electric-Fence, 内核电子栅栏)](https://mp.weixin.qq.com/s/62Oht7WRnKPUBdOJfQ9cBg), 是一个低开销的基于采样的内存安全错误检测器, 用于检测堆后释放、无效释放和越界访问错误. 该系列使 KFENCE 适用于 x86 和 arm64 体系结构, 并将 KFENCE 钩子添加到 SLAB 和 SLUB 分配器.

KFENCE 被设计为在生产内核中启用, 它的性能开销几乎为零. 与 KASAN 相比, KFENCE 以性能换取精度. KFENCE 设计背后的主要动机是, 如果总正常运行时间足够长, KFENCE 将检测非生产测试工作负载通常不会执行的代码路径中的 BUG.

KFENCE 的灵感来自于 [GWP-ASan](http://llvm.org/docs/GwpAsan.html), 这是一个具有类似属性的用户空间工具. "KFENCE" 这个名字是对 [Electric Fence Malloc Debugger](https://linux.die.net/man/3/efence) 的致敬.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/11/03 | Marco Elver <elver@google.com> | [KFENCE: A low-overhead sampling-based memory safety error detector](https://lore.kernel.org/patchwork/cover/1331483) | 轻量级基于采样的内存安全错误检测器 | v7 ☑ 5.12-rc1 | [PatchWork v24](https://lore.kernel.org/patchwork/cover/1331483) |
| 2020/04/21 | Marco Elver <elver@google.com> | [kfence: optimize timer scheduling](https://lore.kernel.org/patchwork/cover/1416384) | ARM 支持 PTDUMP | RFC ☑ 3.19-rc1 | [PatchWork v24](https://lore.kernel.org/patchwork/cover/1416384) |


## 13.4 debugfs & sysfs 接口
-------

### 13.4.1 ptdump
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2014/11/26 | Russell King - ARM Linux | [ARM: add support to dump the kernel page tables](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20131024071600.GC16735@n2100.arm.linux.org.uk) | ARM 支持 PTDUMP | RFC ☑ 3.19-rc1 | [PatchWork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20131024071600.GC16735@n2100.arm.linux.org.uk), [LWN](https://lwn.net/Articles/572320) |
| 2014/11/26 | Laura Abbott | [arm64: add support to dump the kernel page tables](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1416961719-6644-1-git-send-email-lauraa@codeaurora.org) | ARM64 支持 PTDUMP | RFC ☑ 3.19-rc1 | [PatchWork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1416961719-6644-1-git-send-email-lauraa@codeaurora.org) |
| 2019/12/19 | Colin Cross | [Generic page walk and ptdump](https://lore.kernel.org/patchwork/cover/1169746) | 重构了 ARM64/X86_64 的 PTDUMP, 实现了通用的 PTDUMP 框架. 目前许多体系结构都有一个 debugfs 文件用于转储内核页表. 目前, 每个体系结构都必须为此实现自定义函数, 因为内核使用的页表的遍历细节在不同的体系结构之间是不同的. 本系列扩展了walk_page_range()的功能, 使其能够处理内核的页表(内核没有vma, 可以包含比用户空间现有页面更大的页面). 通用的 PTDUMP 实现是利用walk_page_range()的新功能实现的, 最终arm64和x86将转而使用它, 删除了自定义的表行器. | v17 ☑ 5.6-rc1 |[PatchWork RFC](https://lore.kernel.org/patchwork/cover/1169746) |
| 2020/2/10 |  Zong Li | [RISC-V page table dumper](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20131024071600.GC16735@n2100.arm.linux.org.uk) | ARM 支持 PTDUMP | RFC ☑ 3.19-rc1 | [LKML](https://lkml.org/lkml/2020/2/10/100) |


### 13.4.2   Page Owner
-------

早在 2.6.11 的时代就通过 [Page owner tracking leak detector](https://lwn.net/Articles/121271) 给大家展示了页面所有者跟踪 [Page Owner](https://lwn.net/Articles/121656). 但是它一直停留在 Andrew 的源代码树上, 然而, 没有人试图 upstream 到 mainline, 虽然已经有不少公司使用这个特性来调试内存泄漏或寻找内存占用者. 最终在 v3.19, Joonsoo Kim 在一个重构中, 将这个特性推到主线.

这个功能帮助我们知道谁分配了页面. 当分配一个页面时, 我们将一些关于分配的信息存储在额外的内存中. 之后, 如果我们需要知道所有页面的状态, 我们可以从这些存储的信息中获取并分析它.

虽然我们已经有了跟踪页分配/空闲的跟踪点, 使用它来分析页面所有者是相当复杂的. 我们需要扩大跟踪缓冲区, 以防止重叠, 直到用户空间程序启动. 而且, 启动的程序不断地将跟踪缓冲区转储出来供以后分析, 它将更有可能改变系统行为, 而不是仅仅将其保存在内存中, 因此不利于调试.

此外, 我们还可以将 page_owner 特性用于各种目的. 例如, 我们可以借助 page_owner 实现的碎片统计以及一些 CMA 故障调试特性.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/11/24 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [Resurrect and use struct page extension for some debugging features](https://lore.kernel.org/patchwork/cover/520462) | 1. 引入 page_ext 作为 page 的扩展来存储调试相关的变量<br>2. upstream page_owner | v7 ☑ 3.19-rc1 | [PatchWork v3](https://lore.kernel.org/patchwork/cover/520462), [Kernel Newbies](https://kernelnewbies.org/Linux_3.19#Memory_management) |


### 13.4.3 PROC_PAGE_MONITOR
-------

[深入理解Linux内存回收 - Round One](https://zhuanlan.zhihu.com/p/320688306)

| 文件 | 描述 |
|:---:|:----:|
| `/proc/pid/smaps` | 使用 `/proc/pid/maps` 可以高效的确定映射的内存区域、跳过未映射的区域. |
| `/proc/kpagecount` | 这个文件包含64位计数 ,  表示每一页被映射的次数, 按照PFN值固定索引. |
| [`/proc/kpageflags`](https://www.kernel.org/doc/html/latest/admin-guide/mm/pagemap.html?highlight=kpageflags) | 此文件包含为64位的标志集, 表示该页的属性, 按照PFN索引. |

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/10/09 | Matt Mackall <mpm@selenic.com> | [maps4: pagemap monitoring v4](https://lore.kernel.org/patchwork/cover/95279) | 引入 CONFIG_PROC_PAGE_MONITOR, 管理了 `/proc/pid/clear_refs`, `/proc/pid/smaps`, `/proc/pid/pagemap`, `/proc/kpagecount`, `/proc/kpageflags` 多个接口. | v1 ☑ 2.6.25-rc1 | [PatchWork v4 0/12](https://lore.kernel.org/patchwork/cover/95279) |
| 201=09/05/08 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [export more page flags in /proc/kpageflags (take 6)](https://lore.kernel.org/patchwork/cover/155330) | 在 kpageflags 中导出了更多的 type. 同时新增了一个用户态工具 page-types 可以调试进程和内核的 page-types. 该工具后期[移到了 tools/vm 目录下](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c6dd897f3bfc54a44942d742d6dfa842e33d88e0) | v6 ☑ 3.4-rc1 | [PatchWork v3](https://lore.kernel.org/patchwork/cover/520462), [Kernel Newbies](https://kernelnewbies.org/Linux_3.19#Memory_management) |

### 13.4.4   数据访问监视器 DAMON
-------

[linux data access monitor (DAMON)](https://blog.csdn.net/zqh1630/article/details/109954910)

[LWN：用DAMON来优化memory-management!](https://blog.csdn.net/Linux_Everything/article/details/104707923)

对指定的程序进行内存相关优化, 了解业务给定工作负载的数据访问模式至关重要. 但是, 从庞大和复杂的工作量中手动提取此类模式非常详尽. 更糟糕的是, 现有的内存访问分析工具会为不必要的详细分析结果带来不可接受的高开销.

内存管理在很大程度上基于预测 : 给定进程在不久的将来将需要哪些内存页?

不幸的是, 事实证明, 预测是困难的, 尤其是对于未来事件的预测. 在没有从未来发送回的有用信息的情况下, 内存管理子系统被迫依赖于对最近行为的观察, 并假设该行为可能会继续. 但是, 内核的内存管理决策对于用户空间是不透明的, 并且常常导致性能不佳. SeongJae Park 实现的 [DAMON](https://lwn.net/Articles/812707) 试图使内存使用模式对用户空间可见, 并让用户空间作为响应来更改内存管理决策.

亚马逊工程师公开了他们关于 ["DAMON"](https://github.com/awslabs/damo) 的工作, 这是一个Linux内核模块, 用于监控特定用户空间过程的数据访问.

[DAMON](https://damonitor.github.io/_index) 允许用户监控特定用户空间过程的实际内存访问模式. 从概念上讲, 它的操作很简单;

1.  首先, 将进程的地址空间划分为多个大小相等的区域.

2.  然后, 它监视对每个区域的访问, 并提供对每个区域的访问次数的直方图作为其输出.

由此, 该信息的使用者(在用户空间或内核中)可以请求更改以优化进程对内存的使用.

它的目标是 :

1.  足够准确, 可用于实际性能分析和优化;

2.  足够轻量, 以便它可以在线使用;

DAMON 利用两个核心机制 : **基于区域的采样**和**自适应区域调整**, 允许用户将跟踪开销限制在有界范围内, 而与目标工作负载的大小和复杂性无关, 同时保留结果的质量.

1.  基于区域的抽样允许用户在监控质量和开销之间做出自己的权衡, 并限制监控开销的上限.

2.  自适应区域调整机制使 DAMON 在保持用户配置的权衡的同时, 最大限度地提高精度, 减少开销.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/05/20 | SeongJae Park <sjpark@amazon.com> | [Introduce Data Access MONitor (DAMON)](https://damonitor.github.io) | 数据访问监视器 DAMON | v29 ☐ | [PatchWork v29](https://lore.kernel.org/patchwork/cover/1375732), [LWN](https://lwn.net/Articles/1432223) |
| 2021/06/08 | SeongJae Park <sjpark@amazon.com> | [Introduce DAMON-based Proactive Reclamation](https://damonitor.github.io) | 该补丁集改进了用于生产质量的通用数据访问模式内存管理的引擎, 并在其之上实现了主动回收. | v2 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/1375732)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/cover/1442732) |


### 13.4.5   其他
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/02/03 | Christian Borntraeger <borntraeger@de.ibm.com> | [Optimize CONFIG_DEBUG_PAGEALLOC (x86 and s390)](https://damonitor.github.io) | 优化 CONFIG_DEBUG_PAGEALLOC, 提供了 debug_pagealloc_enabled(), 可以动态的开启 DEBUG_PAGEALLOC. | v4 ☑ 4.6-rc1 | [PatchWork v4,0/4](https://lore.kernel.org/patchwork/cover/642851) |

# 14 杂项
-------


这是最后一章, 讲几个 Linux 内存管理方面的属于锦上添花性质的功能, 它们使 Linux 成为一个更强大, 更好用的操作系统.



## 14.1 KSM - 内存去重
-------


**2.6.32(2009年12月发布)**

现代操作系统已经使用了不少共享内存的技术, 比如共享库, 创建新进程时子进程共享父进程地址空间. 而 KSM(Kernel SamePage Merging, 内存同页合并, 又称内存去重), 可以看作是存储领域去重(de-duplication)技术在内存使用上的延伸, 它是为了解决服务器虚拟化领域的内存去重方案. 想像在一个数据中心, 一台物理服务器上可能同时跑着多个虚拟客户机(Guest OS), 并且这些虚拟机运行着很多相同的程序, 如果在物理内存上, 这些程序文本(text)只有一份拷贝, 将会节省相当可观的内存. 而客户机间是相对独立的, 缺乏相互的认知, 所以 KSM 运作在监管机(hypervisor)上.



原理上简单地说, KSM 依赖一个内核线程, 定期地或可手动启动地, 扫描物理页面(通常稳定不修改的页面是合并的候选者, 比如包含执行程序的页面. 用户也可通过一个系统调用给予指导, 告知 KSM 进程的某部份区间适合合并, 见 [Kernel Samepage Merging](https://www.kernel.org/doc/html/latest/admin-guide/mm/ksm.html), 寻找相同的页面并合并, 多余的页面即可释放回系统另为它用. 而剩下的唯一的页面, 会被标为只读, 当有进程要写该页面, 该会为其分配新的页面.



值得一提的是, 在匹配相同页面时, 一种常规的算法是对页面进行哈希, 放入哈希列表, 用哈希值来进行匹配. 最开始 KSM 确定是用这种方法, 不过 VMWare 公司拥有跟该做法很相近的算法专利, 所以后来采用了另一种算法, 用红黑树代替哈希表, 把页面内容当成一个字符串来做内容比对, 以代替哈希比对. 由于在红黑树中也是以该"字符串值"大小作为键, 因此查找两个匹配的页面速度并不慢, 因为大部分比较只要比较开始若干位即可. 关于算法细节, 感兴趣者可以参考这两篇文章:

1.  [Linux内存管理： Linux Kernel Shared Memory 剖析 Linux 内核中的内存去耦合](https://www.cnblogs.com/ajian005/archive/2012/12/18/2841108.html).

2.  [KSM tries again](https://lwn.net/Articles/330589)

## 14.2 HWPoison - 内存页错误的处理
-------

**2.6.32(2009年12月发布)**


[一开始想把这节放在第12章"**内存管理调试支持**"中, 不过后来觉得这并非用于主动调试的功能, 所以还是放在此章. ]



随着内存颗粒密度的增大和内存大小的增加, 内存出错的概率也随之增大. 尤其是数据中心或云服务商, 系统内存大(几百 GB 甚至上 TB 级别), 又要提供高可靠的服务(RAS), 不能随随便便宕机; 然而, 内存出错时, 特别是出现多于 ECC(Error Correcting Codes) 内存条所支持的可修复 bit 位数的错误时, 此时硬件也爱莫能助, 将会触发一个 MCE(Machine Check Error) 异常, 而通常操作系统对于这种情况的做法就是 panic (操作系统选择 go die). 但是, 这种粗暴的做法显然是 over kill, 比如出错的页面是一个**文件缓存页(page cache),** 那么操作系统完全可以把它废弃掉(随后它可以从后备文件系统重新把该页内容读出), 把该页隔离开来不用即是.



这种需求在 Intel 的 Xeon 处理器中得到实现. Intel Xeon 处理器引入了一个所谓 MCA(Machine Check Abort)架构, 它支持包括**内存出错的的毒化(Poisoning)**在内的硬件错误恢复机制. 当硬件检测到一个无法修复的内存错误时,会把该数据标志为损坏(poisoned); 当之后该数据被读或消费时, 将会触发机器检查(Machine Check), 不同的时, 不再是简单地产生 MCE 异常, 而是调用操作系统定义的处理程序, 针对不同的情况进行细致的处理.



2.6.32 引入的 HWPoison 的 patch, 就是这个操作系统定义的处理程序, 它对错误数据的处理是以页为单位, 针对该错误页是匿名页还是文件缓存页, 是系统页还是进程页, 等等, 多种细致情况采取不同的措施. 关于此类细节, 可看此文章: [HWPOISON](https://lwn.net/Articles/348886)



## 14.3 Cross Memory Attach - 进程间快速消息传递
-------

**3.2(2012年1月发布)**

这一节相对于其他本章内容是独立的. MPI(Message Passing Interface, 消息传递接口) [The Message Passing Interface (MPI) standard](https://www.mcs.anl.gov/research/projects/mpi) 是一个定义并行编程模型下用于进程间消息传递的一个高性能, 可扩展, 可移植的接口规范(注意这只是一个标准, 有多个实现). 之前的 MPI 程序在进程间共享信息是用到共享内存(shared memory)方式, 进程间的消息传递需要 2 次内存拷贝. 而 3.2 版本引入的 "Cross Memory Attach" 的 patch, 引入两个新的系统调用接口. 借用这两个接口, MPI 程序可以只使用一次拷贝, 从而提升性能.


## 14.4 CPA(Change Page Attribute)
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/09/17 | Srivatsa S. Bhat <srivatsa.bhat@linux.vnet.ibm.com> | [x86/mm/cpa: Improve large page preservation handling](https://lore.kernel.org/patchwork/cover/987147) | 优化 页面属性(CPA) 代码中的 try_preserve_large_page(), 降低 CPU 消耗. | v3 ☑ 4.20-rc1 | [PatchWork RFC v3](https://lore.kernel.org/patchwork/cover/987147) |


## 14.5 功耗管理
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/04/15 | Thomas Gleixner <tglx@linutronix.de> | [mm: Memory Power Management](https://lore.kernel.org/patchwork/cover/408914) | 内存的电源管理策略  | v4 ☑ 4.4-rc1 | [PatchWork RFC v4](https://lore.kernel.org/patchwork/cover/408914), [LWN](https://lwn.net/Articles/547439) |


## 14.6 PER_CPU
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2009/02/18 | Tejun Heo <tj@kernel.org> | [implement new dynamic percpu allocator](https://lore.kernel.org/patchwork/cover/144750) | 实现了 [vm_area_register_early()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f0aa6617903648077dffe5cfcf7c4458f4610fa7) 以支持在启动阶段注册 vmap 区域. 基于此特性实现了可伸缩的动态 percpu 分配器(CONFIG_HAVE_DYNAMIC_PER_CPU_ARE), 可用于静态(pcpu_setup_static/pcpu_setup_first_chunk)和动态(`__alloc_percpu`) percpu 区域, 这将允许静态和动态区域共享更快的直接访问方法. | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/144750) |
| 2009/02/24 | Tejun Heo <tj@kernel.org> | [percpu: fix pcpu_chunk_struct_size](https://lore.kernel.org/patchwork/cover/145272) | 1. 为对 static per_cpu(第一个 percpu 块)分配增加更多的自由度: 引入 PERCPU_DYNAMIC_RESERVE 用于预留 percpu 空间, 修改 pcpu_setup_static() 为 pcpu_setup_first_chunk(), 并让其更灵活.<br>2. 实现嵌入式的 percpu 分配器, 在 !NUMA 模式下, 可以简单地分配连续内存并将其用于第一个块, 而无需将其映射到 vmalloc 区域. 由于内存区域是由大页物理内存映射覆盖的, 所以它允许在基于可伸缩的动态 percpu 分配器分配静态 percpu 区域不引入任何 TLB 开销, 而且实现也非常简单.  | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/145272) |
| 2009/03/26 | Tejun Heo <tj@kernel.org> | [percpu: clean up percpu constants](https://lore.kernel.org/patchwork/cover/146513) | NA | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/146513) |
| 2009/07/21 | Tejun Heo <tj@kernel.org> | [percpu:  implement pcpu_get_vm_areas() and update embedding first chunk allocator](https://lore.kernel.org/patchwork/cover/164588) | NA | v1 ☑ 2.6.32-rc1 | [PatchWork](https://lore.kernel.org/patchwork/cover/164588) |



---

**引用:**

<div id="ref-anchor-1"></div>
- [1] [Single UNIX Specification](https://en.wikipedia.org/wiki/Single_UNIX_Specification%23Non-registered_Unix-like_systems)

<div id="ref-anchor-2"></div>
- [2] [POSIX 关于调度规范的文档](http://nicolas.navet.eu/publi/SlidesPosixKoblenz.pdf)

<div id="ref-anchor-3"></div>
- [3] [Towards Linux 2.6](https://link.zhihu.com/?target=http%3A//www.informatica.co.cr/linux-scalability/research/2003/0923.html)

<div id="ref-anchor-4"></div>
- [4] [Linux内核发布模式与开发组织模式(1)](https://link.zhihu.com/?target=http%3A//larmbr.com/2013/11/02/Linux-kernel-release-process-and-development-dictator-%26-lieutenant-system_1)

<div id="ref-anchor-5"></div>
- [5] IBM developworks 上有一篇综述文章, 值得一读 :[Linux 调度器发展简述](https://link.zhihu.com/?target=http%3A//www.ibm.com/developerworks/cn/linux/l-cn-scheduler)

<div id="ref-anchor-6"></div>
- [6] [CFS group scheduling [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/240474)

<div id="ref-anchor-7"></div>
- [7] [http://lse.sourceforge.net/numa/](https://link.zhihu.com/?target=http%3A//lse.sourceforge.net/numa)

<div id="ref-anchor-8"></div>
- [8] [CFS bandwidth control [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/428230)

<div id="ref-anchor-9"></div>
- [9] [kernel/git/torvalds/linux.git](https://link.zhihu.com/?target=https%3A//git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/%3Fid%3D5091faa449ee0b7d73bc296a93bca9540fc51d0a)

<div id="ref-anchor-10"></div>
- [10] [DMA模式\_百度百科](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/196502.htm)

<div id="ref-anchor-11"></div>
- [11] [进程的虚拟地址和内核中的虚拟地址有什么关系? - 詹健宇的回答](http://www.zhihu.com/question/34787574/answer/60214771)

<div id="ref-anchor-12"></div>
- [12] [Physical Page Allocation](https://link.zhihu.com/?target=https%3A//www.kernel.org/doc/gorman/html/understand/understand009.html)


<div id="ref-anchor-17"></div>
- [17] [kernel 3.10内核源码分析--TLB相关--TLB概念、flush、TLB lazy模式-humjb\_1983-ChinaUnix博客](https://link.zhihu.com/?target=http%3A//blog.chinaunix.net/uid-14528823-id-4808877.html)

<div id="ref-anchor-18"></div>
- [18] [Toward improved page replacement[LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/226756)

<div id="ref-anchor-20"></div>
- [20] [The state of the pageout scalability patches [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/286472)

<div id="ref-anchor-21"></div>
- [21] [kernel/git/torvalds/linux.git](https://link.zhihu.com/?target=https%3A//git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/%3Fid%3D894bc310419ac95f4fa4142dc364401a7e607f65)

<div id="ref-anchor-22"></div>
- [22] [Being nicer to executable pages [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/333742)

<div id="ref-anchor-23"></div>
- [23] [kernel/git/torvalds/linux.git](https://link.zhihu.com/?target=https%3A//git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/%3Fid%3D8cab4754d24a0f2e05920170c845bd84472814c6)

<div id="ref-anchor-24"></div>
- [24] [Better active/inactive list balancing [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/495543)

<div id="ref-anchor-29"></div>
- [29] [On-demand readahead [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/235164)

<div id="ref-anchor-30"></div>
- [30] [Transparent huge pages in 2.6.38 [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/423584)

<div id="ref-anchor-32"></div>
- [32] [transcendent memory for Linux [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/338098)

<div id="ref-anchor-33"></div>
- [33] [linux kernel monkey log](https://link.zhihu.com/?target=http%3A//www.kroah.com/log/linux/linux-staging-update.html)

<div id="ref-anchor-34"></div>
- [34] [zcache: a compressed page cache [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/397574)

<div id="ref-anchor-36"></div>
- [36] [Linux-Kernel Archive: Linux 2.6.0](https://link.zhihu.com/?target=http%3A//lkml.iu.edu/hypermail/linux/kernel/0312.2/0348.html)

<div id="ref-anchor-37"></div>
- [37]抢占支持的引入时间: [https://www.kernel.org/pub/linux/kernel/v2.5/ChangeLog-2.5.4](https://link.zhihu.com/?target=https%3A//www.kernel.org/pub/linux/kernel/v2.5/ChangeLog-2.5.4)

<div id="ref-anchor-38"></div>
- [38] [RAM is 100 Thousand Times Faster than Disk for Database Access](https://link.zhihu.com/?target=http%3A//www.directionsmag.com/entry/ram-is-100-thousand-times-faster-than-disk-for-database-access/123964)

<div id="ref-anchor-39"></div>
- [39] [http://www.uefi.org/sites/default/files/resources/ACPI\_6.0.pdf](https://link.zhihu.com/?target=http%3A//www.uefi.org/sites/default/files/resources/ACPI_6.0.pdf)

<div id="ref-anchor-40"></div>
- [40] [Injecting faults into the kernel [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/209257)

<div id="ref-anchor-47"></div>
- [47] [Fast interprocess messaging [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/405346)



---8<---

**更新日志:**

**- 2015.9.12**

o 完成调度器子系统的初次更新, 从早上10点开始写, 写了近７小时, 比较累, 后面更新得慢的话大家不要怪我(对手指

**- 2015.9.19**

o 完成内存管理子系统的前4章更新. 同样是写了一天, 内容太多, 没能写完......

**- 2015.9.21**

o 完成内存管理子系统的第5章"页面写回"的第1小节的更新.
**- 2015.9.25**

o 更改一些排版和个别文字描述. 接下来周末两天继续.
**- 2015.9.26**

o 完成内存管理子系统的第5, 6, 7, 8章的更新.
**- 2015.10.14**

o 国庆离网10来天, 未更新.  今天完成了内存管理子系统的第9章的更新.
**- 2015.10.16**

o 完成内存管理子系统的第10章的更新.
**- 2015.11.22**

o 这个月在出差和休假, 一直未更新.抱歉! 根据知友 [@costa](https://www.zhihu.com/people/78ceb98e7947731dc06063f682cf9640) 提供的无水印图片和考证资料, 进行了一些小更新和修正. 特此感谢 !

o 完成内存管理子系统的第11章关于 NVDIMM 内容的更新.
**- 2016.1.2**

o 中断许久, 今天完成了内存管理子系统的第11章关于调试支持内容的更新.
**- 2016.2.23**

o 又中断许久, 因为懒癌发作Orz... 完成了第二个子系统的所有章节.
[编辑于 06-27](https://www.zhihu.com/question/35484429/answer/62964898)
