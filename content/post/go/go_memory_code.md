---
title: "go 内存管理"
date: 2024-12-10T20:55:56+08:00
lastmod: 2024-12-10T20:55:56+08:00

categories:
  - go
tags:
  - go

draft: true
---


## 操作系统内存管理

操作系统管理内存的存储单元是页（page），在 linux 中一般是 4KB。而且，操作系统还会使用 `虚拟内存` 来管理内存，在用户程序中，我们看到的内存是不是真实的内存，而是虚拟内存。当访问或者修改内存的时候，操作系统会将虚拟内存映射到真实的内存中。申请内存的组件是 Page Table 和 MMU（Memory Management Unit）。因为这个性能很重要，所以在 CPU 中专门有一个 TLB（Translation Lookaside Buffer）来缓存 Page Table 的内容。

为什么要用虚拟内存？

1. 保护内存，每个进程都有自己的虚拟内存，不会相互干扰。防止修改和访问别的进程的内存。
2. 减少内存碎片，虚拟内存是连续的，而真实的内存是不连续的。
3. 当内存不够时，可以把虚拟内存映射到硬盘上，这样就可以使用硬盘的空间来扩展内存。

![](/images/virtual_memory.png)

如上图所示，如果直接使用真是的内存，想要连续的肯定是申请不到的，这就是内存碎片的问题。而使用虚拟内存，通过 Page 映射的方式，保证内存连续。

## Go 内存管理单元

### page

在 go 中，管理内存的存储单元也是页（Page）, 每个页的大小是 8KB。Go 内存管理是由 runtime 来管理的，runtime 会维护一个内存池，用来分配和回收内存。这样可以避免频繁的系统调用申请内存，提高性能。

### mspan

mspan 是 go 内存管理基本单元，一个 mspan 包含一个或者多个 page。go 中有多种 mspan，每种 mspan 给不同的内存大小使用。

| class | bytes/obj | bytes/span | objects | tail waste | max waste | min align |
|-------|-----------|------------|---------|------------|-----------|-----------|
| 1 | 8 | 8192 | 1024 | 0 | 87.50% | 8 |
| 2 | 16 | 8192 | 512 | 0 | 43.75% | 16 |
| 3 | 24 | 8192 | 341 | 8 | 29.24% | 8 |
| 4 | 32 | 8192 | 256 | 0 | 21.88% | 32 |
| 5 | 48 | 8192 | 170 | 32 | 31.52% | 16 |
| 6 | 64 | 8192 | 128 | 0 | 23.44% | 64 |
| 7 | 80 | 8192 | 102 | 32 | 19.07% | 16 |
| 8 | 96 | 8192 | 85 | 32 | 15.95% | 32 |
| 9 | 112 | 8192 | 73 | 16 | 13.56% | 16 |
| ... | ... | ... | ... | ... | ... | ... |
| 64 | 24576 | 24576 | 1 | 0 | 11.45% | 8192 |
| 65 | 27264 | 81920 | 3 | 128 | 10.00% | 128 |
| 66 | 28672 | 57344 | 2 | 0 | 4.91% | 4096 |
| 67 | 32768 | 32768 | 1 | 0 | 12.50% | 8192 |

1. class 是 mspan 的类型，每种类型对应不同的内存大小。
2. obj 是每个对象的大小。
3. span 是 mspan 的大小。
4. objects 是 mspan 中对象的个数。
5. tail waste 是 mspan 中最后一个对象的浪费空间。（不能整除造成的）
6. max waste 是 mspan 中最大的浪费空间。（比如第一个中 每个都使用 1 byte，那么就所有都浪费 7 byte,1 / 7 = 87.50%）
7. min align 是 mspan 中对象的对齐大小。如果超过这个就会分配下一个 mspan。

## 数据结构

### mspan
  
```go
type mspan struct {
	// 双向链表 下一个 mspan 和 上一个 mspan
	next *mspan    
	prev *mspan    
  // debug 使用的
	list *mSpanList 

  // 起始地址和页数 当 class 太大 要多个页组成 mspan
	startAddr uintptr 
	npages    uintptr 

  // 手动管理的空闲对象链表
	manualFreeList gclinkptr 

	// 下一个空闲对象的地址 如果小于它 就不用检索了 直接从这个地址开始 提高效率
	freeindex uint16
	// 对象的个数
	nelems uint16 
	// GC 扫描使用的空闲索引
	freeIndexForScan uint16

	// bitmap 每个 bit 对应一个对象 标记是否使用
	allocCache uint64

	// ...
  // span 的类型 
	spanclass             spanClass     // size class and noscan (uint8)
	//  ...
}
```

#### spanClass

```go
type spanClass uint8

func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
	return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}

//go:nosplit
func (sc spanClass) sizeclass() int8 {
	return int8(sc >> 1)
}

//go:nosplit
func (sc spanClass) noscan() bool {
	return sc&1 != 0
}
```

spanClass 是 unint8 类型，一共有 8 位，前 7 位是 sizeclass，也就是上边 table 中的内容，一共有（67 + 1）种类型 0 代表比 67 class 需要的内存还大。最后一位是 noscan，也就是表示这个对象中是否含有指针，用来给 GC 扫描加速用的。

#### mspan 详解

![](/images/go_memory_mspan.png)

如果所示

- mspan 是一个双向链表，如果不够用了，在挂一个就行了。
- startAddr 是 mspan 的起始地址，npages 是 page 数量。根据 startAddr + npages * 8KB 就可以得到 mspan 的结束地址。
- allocCache 是一个 bitmap，每个 bit 对应一个对象，标记是否使用。使用了 ctz(count trailing zero)。
- freeindex 是下一个空闲对象的地址，如果小于它，就不用检索了，直接从这个地址开始，提高效率。

### mcache

mache 是每个 P （processor）的结构体中都有的，是用来缓存的，因为每个 P 同一时间只有一个 goroutine 在执行，所以 mcache 是不需要加锁的。这也是 mcache 的设计初衷，减少锁的竞争，提高性能。

```go
type p struct {
  // ...
  mcache      *mcache
  // ...
}
```

```go


```