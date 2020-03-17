---
title: 内存管理
date: 2019-08-21T11:00:41+08:00
draft: false
categories: os
---

# 内存管理

## 存储器工作原理

应用程序如何在计算机系统上运行的呢？首先，用编程语言编写和编辑应用程序，所编写的程序称为源程序，源程序不能再计算机上直接被运行，需要通过三个阶段的处理：**编译程序处理源程序并生成目标代码，链接程序把他们链接为一个可重定位代码，此时该程序处于逻辑地址空间中；下一步装载程序将可执行代码装入物理地址空间，直到此时程序才能运行**。

### 程序编译

源程序经过编译程序的处理生成目标模块（目标代码）。**一个程序可由独立编写且具有不同功能的多个源程序模块组成，由于模块包含外部引用（即指向其他模块中的数据或指令地址，或包含对库函数的引用），编译程序负责记录引用发生的位置，其处理结果将产生相应的多个目标模块，每个模块都附有供引用使用的内部符号表和外部符号表。符号表中依次给出各个符号名及在本目标模块中的名字地址，在模块链接时进行转换**。

### 程序链接

链接程序(Linker)的作用是根据目标模块之间的调用和依赖关系，将主调模块、被调模块以及所用到的库函数装配和链接成一个完整的可装载执行模块。根据程序链接发生的时间和链接方式，程序链接可分为以下三种方式：

  - **静态链接**：在程序装载到内存和运行前，就已将它所有的目标模块及所需要的库函数进行链接和装配成一个完整的可执行程序且此后不再拆分。

  - **动态链接**：在程序装入内存前并未事先进行程序各目标模块的链接，而是在程序装载时一边装载一边链接，生成一个可执行程序。在装载目标模块时，若发生外部模块调用，将引发响应外部目标模块的搜索、装载和链接。

  - **运行时链接**：在程序执行过程中，若发现被调用模块或库函数尚未链接，先在内存中进行搜索以查看是否装入内存；若已装入，则直接将其链接到调用程序中，否则进行该模块在外存上的搜索，以及装入内存和进行链接，生成一个可执行程序。

运行时链接将链接推迟到程序执行时，可以很好的提高系统资源的利用率和系统效率。

### 程序装载

程序装载就是将可执行程序装入内存，这里有三种方式：

  - **绝对装载**：装载模块中的指令地址始终与其内存中的地址相同，即模块中出现的所有地址均为绝对地址。

  - **可重定位装载**：根据内存当时的使用情况，决定将装载代码模块放入内存的物理位置。模块内使用的都是相对地址。

  - **动态运行时装载**：为提高内存利用率，装入内存的程序可换出到磁盘上，适当时候再换入内存中，对换前后程序在内存中的位置可能不同，即允许进程的内存映像在不同时候处于不同位置，此时模块内使用的地址必定为相对地址。

磁盘中的装载模块所使用的是逻辑地址，其逻辑地址集合称为进程的逻辑地址空间。进程运行时，其装载代码模块将被装入物理地址空间中，此时程序和数据的实际地址不可能同原来的逻辑一致。**可执行程序逻辑地址转换为物理地址的过程被称为 “地址重定位”**。

  - **静态地址重定位**：由装载程序实现装载代码模块的加载和物理地址转换，把它装入分配给进程的内存指定区域，其中所有逻辑地址修改为物理地址。地址转换在进程执行前一次完成，易于实现，但不允许程序在执行过程中移动位置。

  - **动态地址重定位**：由装载程序实现装载代码模块的加载，把它装入分配给进程的内存指定区域，但对链接程序处理过的程序的逻辑地址不做任何改变，程序内存起始地址被置入硬件专用寄存器 —— **重定位寄存器**。程序执行过程中，每当CPU引用内存地址时，有硬件截取此逻辑地址，并在它被发送到内存之前加上重定位寄存器的值，以实现地址转换。

  - **运行时链接地址重定位**：对于静态和动态地址重定位装载方式而言，装载代码模块是由整个程序的所有目标模块及库函数经链接和整合构成的可执行程序，即在程序启动执行前已经完成了程序的链接过程。可见，装载代码的正文结构是静态的，在程序运行期间保持不变。**运行时链接装载方式必然采用运行时链接地址重定位**。

> 重定位寄存器：用于保存程序内存起始地址。

## 连续存储管理

### 固定分区存储管理

固定分区存储管理又称为静态分区模式，基本思想是：内存空间被划分成数目固定不变的分区，各分区大小不等，每个分区装入一个作业，若多个分区中都有作业，则他们可以并发执行。

为说明各分区分配和使用情况，需要设置一张内存分配表，记录内存中划分的分区及其使用情况。内存分配表中指出各分区起始地址和长度，占用标志用来指示此分区是否被使用。

### 可变分区存储管理

可变分区存储管理按照作业大小来划分分区，但划分的时间、大小、位置都是动态的。系统把作业装入内存时，根据其所需要的内存容量查看是否有足够空间，若有则按需分割一个分区分配给此作业；若无则令此作业等待内存资源。

## 虚拟内存

在连续存储管理的方式下，程序直接操作物理内存。这样的内存管理方式会造成诸多错误：**地址空间不隔离**、运行时内存地址不确定、内存使用率低。为解决这些问题人们提出了 **内存分段** 技术，同时内存分段需要引入 **虚拟地址空间** 的概念。

> 地址空间：可寻址的一片空间，如果这个空间是虚拟的，就是虚拟地址空间；如果是物理的，就是物理地址空间

虚拟内存是计算机系统内存管理的一种技术，**虚拟地址空间构成虚拟内存**。它使得应用程序认为它拥有连续可用的内存（一个连续完整的地址空间），而实际上，它通常是被分隔成多个物理内存碎片。还有部分暂时存储在外部磁盘存储器上（Swap），在需要时进行数据交换。与没有使用虚拟内存技术的系统相比，使用这种技术的系统使得大型程序的编写变得更容易，对真正的物理内存的使用也更有效率。

**虚拟内存不只是用磁盘空间来扩展物理内存** 的意思——这只是扩充内存级别以使其包含硬盘驱动器而已。把内存扩展到磁盘只是使用虚拟内存技术的一个结果，它的作用也可以通过覆盖或者把处于不活动状态的程序以及它们的数据全部交换到磁盘上等方式来实现。

### 内存分页

内存分段能很好的解决地址空间不隔离、运行时内存地址不确定的问题，但对于内存使用率低不能很好的解决。这个问题的关键是能不能在换出一个完整的程序之后，把另一个完整的程序换进来。而这种分段机制，映射的是一片连续的物理内存，所以得不到解决。

**内存分页仍然是一种虚拟地址空间到物理地址空间映射的机制，但粒度更加的小了**。内存分页的虚拟地址空间仍然是连续的，但是，每一页映射后的物理地址就不一定是连续的了。正是因为有了分页的概念，程序的换入换出就可以以页为单位了。

### MMU

内存管理单元（memory management unit，MMU），它是一种负责处理中央处理器（CPU）的内存访问请求的计算机硬件。它的功能包括 **虚拟地址到物理地址的转换**（即虚拟内存管理）、内存保护、中央处理器高速缓存的控制。

内存管理单元通常借助一种 **转译旁观缓冲器（Translation Lookaside Buffer，TLB）** 的高速缓存来将虚拟页号转换为物理页号。当 TLB 中没有转换记录时，则使用一种较慢的机制，其中包括专用硬件（hardware-specific）的数据结构（Data structure）或软件辅助手段。这个数据结构称为 **分页表**，页表中的数据就叫做 **分页表项（page table entry，PTE）**。物理页号结合页偏移量便提供出了完整的物理地址。

有时，TLB 或 PTE 会禁止对虚拟页的访问，这可能是因为没有物理随机存取存储器（random access memory）与虚拟页相关联。如果是这种情况， MMU 将向 CPU 发出 **缺页错误（page fault）** 的信号。操作系统将进行处理，也许会尝试寻找 RAM 的空白帧，同时创建一个新的 PTE 将之映射到所请求的虚拟地址。如果没有空闲的 RAM，可能必须关闭一个已经存在的页面，使用一些替换算法，将之保存到磁盘中（这被称之为 **页面调度**）。

### TLB

最简单的分页表系统通常维护一个帧表和一个分页表，帧表处理帧映射信息。分页表处理页的虚拟地址和物理帧的映射。还有一些辅助信息，如当前存在标识位(present bit)，**脏数据标识位** 或已修改的标识位，地址空间或 **进程ID** 信息。

辅助存储，比如硬盘可以用于增加物理内存。页可以调入和调出到物理内存和磁盘。当前存在标识位可以指出那些页在物理内存，哪些页在硬盘上。页可以指出如何处理这些不同页。是否从硬盘读取一个页，或把其他页从物理内存调出。页面的频繁更换，导致整个系统效率急剧下降，这个现象称为 **内存抖动（或颠簸）**。抖动一般是内存分配算法不好，内存太小引或者程序的算法不佳引起的。

脏数据标识位可以优化性能。一个页从硬盘调入到物理内存中，读取后调出，当页没有更改不需要写回硬盘。但是如果页在调入物理内存后被修改，会标记其为脏数据，意味着该页必需被写回备用存储。这种策略需要当页被调入到内存后，后备存储保留一份页的拷贝。有了脏数据标志位，一些页随时会同时存在于物理内存和后备存储中。

地址空间 或 进程ID 信息是必要的，内存管理系统需要知道页对应的进程。两个不同的进程可以使用两个相同的虚拟地址。分页表必需为两个进程提供不同的虚拟内存。用 **虚拟内存页关联进程ID** 页可以帮助选择要调出的页，当页关联到不活跃进程上，特别是那些主要代码页被调出的进程，相比活跃进程需要页的可能性会更小。

### 进程空间

有内存分页后，进程的概念很自然就被发明出来了。TLB 是 MMU 的输入，那么一个自然的想法，我们有多个 TLB ，一会儿用这个，一会儿用那个，这样我们可以根据需要用不同的内存，这个用来表达不同 MMU 的概念，体现在软件的观感上，就是进程。

当然， MMU 封装为进程，还和“线程”概念进行了一定程度上进行了绑定，所以，软件的进程，不是简单的 MMU 的封装，但我们在理解寻址的概念的时候，是可以简单这样认为的。有了页寻址的概念，进程的空间和物理地址看到的空间就很不一样了。

### 进程内核空间

从进程发出的内存访问，都通过 MMU 翻译，包括请求从一个进程切换到另一个进程。这样一来，进程的权力太大了。所以，还是很自然的，我们在 MMU 中可以设置一个标志，说明 **某些地址是具有更高优先级** 的，一般情况下不允许访问，要访问就要提权。一旦提权，代码执行流就强行切换到这片高级内存的特定位置。这样，一个进程就有了两个权限级别：用户态和内核态，用户态用于一般的程序，内核态用来做进程的管理。

用户态和内核态都在同一个页表中管理，所以，本质上它们属于 **同一个进程**。只是它们属于进程的不同权限而已。为了实现的方便，现在通常让所有进程的 MMU 页表的内核空间指向物理空间的同一个位置，这样，看起来我们就有了一个独立于所有进程的实体，这个实体被称为 *操作系统内核*，它是管理所有进程的一个中心软件。

### 页面调度算法

  - **FIFO算法**：先入先出，即淘汰最早调入的页面。

  - **OPT(MIN)算法**：选未来最远将使用的页淘汰，是一种最优的方案，可以证明缺页数最小。可惜，MIN需要知道将来发生的事，只能在理论中存在，实际不可应用。

  - **LRU(Least-Recently-Used)算法**：用过去的历史预测将来，选最近最长时间没有使用的页淘汰(也称最近最少使用)。性能最接近OPT。**与页面使用时间有关**。

  - **LFU(Least Frequently Used)算法**：即最不经常使用页置换算法，要求在页置换时置换引用计数最小的页，因为经常使用的页应该有一个较大的引用次数。**与页面使用次数有关**。

  - **Clock**：给每个页帧关联一个使用位，当该页第一次装入内存或者被重新访问到时，将使用位置为1。每次需要替换时，查找使用位被置为0的第一个帧进行替换。在扫描过程中，如果碰到使用位为1的帧，将使用位置为0，在继续扫描。如果所谓帧的使用位都为0，则替换第一个帧。

## 参考链接

- [地址空间的故事](https://zhuanlan.zhihu.com/p/25999484)