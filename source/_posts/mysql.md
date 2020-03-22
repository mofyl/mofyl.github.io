---
title: mysql
date: 2020-03-22 21:33:07
tags:
	- mysql
Category: mysql
---

[TOC]
### 简介
- golang的runtime，**<u>抛弃了传统的分配方式，改为自主管理</u>**。
- 核心思想就是，内存多级化管理，从而降低锁的粒度。提高并发。
    - 它将可用的堆内存采用二级分配的方式进行管理，**每个线程独自维护一个自己的内存池**，在进行内存分配时，会先使用内存池中的资源，若是不够才会到全局内存池中申请。**这样就避免了不同线程对全局内存资源的竞争**。
- golang中会对所有的资源进行切分，并统一分配。runtime启动时会向操作系统申请一块内存，这是还是虚拟内存，然后对这块内存进行切分。
- 这些切分好的资源会先到mheap中，mheap是一个全局的内存池。
    - 申请链：
        - 每个goroutine 在运行时 都会去绑定一个P，这个P 里面就会有一个mcache。 mcache 先向全局的  mcentral 请求资源，mcentral在向mheap 申请资源。
        
#### 内存划分
![da82243430c3f5116535b608f262991e.png](evernotecid://2D3CD95A-08BE-4200-BAEB-3591E2B144FF/appyinxiangcom/27335019/ENResource/p105)
- golang在申请到 虚拟内存后 会先将内存分割为 上图所示。
- **go里面动态分配的内存都会在这里进行，而且这里内存的回收都是需要GC参与的** 普通的栈分配，就会直接在栈上。
- arena
    - 这里就是传统意义上的堆区，**在这里会将内存分割成8k大小的区域，这些8k大小的区域称之为页**。而页组合起来就构成了 spans区
- bitmap
    - 标识了arena中那些区域保存了，并用4bit的标志位，标记了该对象中是否包含指针，GC的标记信息。bitmap中 一个byte的内存对应arena中4个指针的内存使用情况，所以 size bitmap=512G / (4*8B) = 16G
    - **bitmap 的内存使用是 从后向前的**
    ![abddb7393103615a2879d9165f04cac5.png](evernotecid://2D3CD95A-08BE-4200-BAEB-3591E2B144FF/appyinxiangcom/27335019/ENResource/p106)
- spans：
    - 存放 mspan指针，就是arena分割的页组合起来的的内存管理基本单元。每个指针对应一页。
- mspan:
    - runtime 中内存管理的基本单元，是由一片连续的8kb的页组成大块内存。
    - 注意:这里的页和操作系统的页不是一回事，一般是操作系统页的几倍。
    - mspan是一个包含起始地址，mspan规格，页的数量等内容的双端链表。
    - object：mspan按照 自身的**属性 size class**的大小，分割成若干个object，**每个object可以用来存放对象**。而且会使用**一个位图**来标记**尚未使用**的object
        - 属性size class 决定了object 的大小
        - **mspan只会分配给和object尺寸大小相近的对象**。对象的大小 会小于 object 的大小
    - 这里会有两个数组 来决定 当前mspan有多少页，和该页中的object有多大
        - **每个mspan 会有一个自己的sizeclass属性，sizeclass 最大67**
        - **会先将该 size class  到 class_to_allocnpages 这个数组中对比 获取该mspan中page的个数**
        - **然后将该 size class 到 class_to_size 这个数组中获取 page中object 的大小**。
            - **class_to_size中最大的是 32b，超过这个大小就是大对象了，直接由堆分配对象**，0也是大对象，跟32b处理一样
        - 对于微小对象(小于16B)，分配器就会将其合并，将几个对象分配到一个object中。
    - 小对象(小于等于32B)由mspan 来分配，大对象(大于32B)直接由堆来分配。
   ![8c02e587ef200d651a4a701d5794695f.png](evernotecid://2D3CD95A-08BE-4200-BAEB-3591E2B144FF/appyinxiangcom/27335019/ENResource/p107)
   
    ```go
    // mspan结构体定义：
    // path: /usr/local/go/src/runtime/mheap.go

    type mspan struct {
        //链表前向指针，用于将span链接起来
    	next *mspan	
    	
    	//链表前向指针，用于将span链接起来
    	prev *mspan	
    	
    	// 起始地址，也即所管理页的地址
    	startAddr uintptr 
    	
    	// 管理的页数
    	npages uintptr 
    	
    	// 块个数，表示有多少个块可供分配
    	nelems uintptr 
    
        //分配位图，每一位代表一个块是否已分配
    	allocBits *gcBits 
    
        // 已分配块的个数
    	allocCount uint16 
    	
    	// class表中的class ID，和Size Classs相关
    	spanclass spanClass  
    
        // class表中的对象大小，也即块大小
    	elemsize uintptr 
    }
    ```
    ![d70bf95318aac84ee3178953e2f117f7.png](evernotecid://2D3CD95A-08BE-4200-BAEB-3591E2B144FF/appyinxiangcom/27335019/ENResource/p108)
    
    - 假设最左边第一个mspan的Size Class等于10，根据前面的class_to_size数组，得出这个msapn分割的object大小是144B，算出可分配的对象个数是8KB/144B=56.89个，取整56个，所以会有一些内存浪费掉了，Go的源码里有所有Size Class的mspan浪费的内存的大小；再根据class_to_allocnpages数组，得到这个mspan只由1个page组成

#### 内存管理组件
- mcache
    - 每个工作线程想要运行，都会去绑定一个P,而mcache就在P里面。本地缓存 可用的 mspan资源。这样就可以直接给goroutine分配，不存在多个goroutine竞争，所以不会消耗锁资源。
    ```go
    //path: /usr/local/go/src/runtime/mcache.go

    type mcache struct {
        alloc [numSpanClasses]*mspan
    }
    // 是SizeClasses 的2倍
    numSpanClasses = _NumSizeClasses << 1
    ```
    - mcache 中使用span_class作为索引来管理 mspan，alloc包含所有的规格的mspan，他是size_class的2倍。由于一半要保存 不包含指针的，一半要保存包含指针的，所以是2倍。
    - mcache在初始化的时候是没有任何mspan资源的。在使用的时候会去mcentral中动态的申请，之后会缓存下来
    - 小于等于32B的时候会直接从 mcache中分配
- mcentral
    - **每个 mcentral保存了一种特定大小的 mspan全局列表**，包括已使用的和未使用的。当goroutine中的mcache中的mspan不够了，就会到mcentral中获取。请求不同大小的mspan会到不同的mcentral中
        - 特定大小：就是sizeclass的大小
    - 当mcache从mcentral中申请时，**只需要在特定的mcentral上加锁**。并不会影响申请其他规格的mspan
    ```go
    //path: /usr/local/go/src/runtime/mcentral.go

    type mcentral struct {
        // 互斥锁
        lock mutex 
        
        // 规格
        sizeclass int32 
        
        // 尚有空闲object的mspan链表
        nonempty mSpanList 
        
        // 没有空闲object的mspan链表，或者是已被mcache取走的msapn链表
        empty mSpanList 
        
        // 已累计分配的对象个数
        nmalloc uint64 
    }
    ```
    ![1c0598667022187bd1c73e1e71a6bb79.png](evernotecid://2D3CD95A-08BE-4200-BAEB-3591E2B144FF/appyinxiangcom/27335019/ENResource/p109)
    
    - empty中保存了已被使用的 mspan，分配了mspan后 该mspan就被那个工作线程独占了。
    - nonempty保存了未被使用的mspan。
    简单说下mcache从mcentral获取和归还mspan的流程：
        - 获取：
            - 加锁；从nonempty链表找到一个可用的mspan；并将其从nonempty链表删除；将取出的mspan加入到empty链表；将mspan返回给工作线程；解锁。
        -归还
            - 加锁；将mspan从empty链表删除；将mspan加入到nonempty链表；
        - 解锁。

- mheap：代表了Go程序所持有的所有堆空间。runtime使用一个全局的mheap 对象 _mheap 来管理堆内存。
    - 当mcache没有空闲时，会向mcentral申请。当mcentral没有空闲时，会向mheap申请。mheap没有空闲时，会向操作系统申请。
    - mheap主要用于分配大对象，以及管理未切割的mspan，**将为切割的mspan切割为小对象给mcentral使用**。
    - **mheap中有所有规格的mspan**。
    ```go
    //path: /usr/local/go/src/runtime/mheap.go

    type mheap struct {
    	lock mutex
    	
    	// spans: 指向mspans区域，用于映射mspan和page的关系
    	spans []*mspan 
    	
    	// 指向bitmap首地址，bitmap是从高地址向低地址增长的
    	bitmap uintptr 
    
        // 指示arena区首地址
    	arena_start uintptr 
    	
    	// 指示arena区已使用地址位置
    	arena_used  uintptr 
    	
    	// 指示arena区末地址
    	arena_end   uintptr 
    
    	central [67*2]struct {
    		mcentral mcentral
    		pad [sys.CacheLineSize - unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte
    	}
    }
    ```
    ![3907f4c3b3035f4cba1926a5c2b1b828.png](evernotecid://2D3CD95A-08BE-4200-BAEB-3591E2B144FF/appyinxiangcom/27335019/ENResource/p110)
    
    - 这里 134个不是67个：是因为 分成了两半，一半用于分配包含指针的，一半用于分配不包含指针的。
### 总结
- 变量分配在栈上还是堆上 是由**逃逸分析**决定的。
- 一般的 编译器还是希望倾向于分配到栈上。因为开销小。所有的变量都在栈上分配，就不会有内存碎片，垃圾回收的开销。
- go在启动的时候，会向操作系统申请一大块内存。之后自行管理。
- go内存管理的基本单位是 mspan。mspan中包含了若干个页。每个页中可以分配特定大小的object
- 极小对象(<= 16B)通过mcache中的tiny分配器来分配。而且可以的话会合并到一个object中。
- 一般小对象 (16B,32B],计算了对象的规格以后，在mcache分配。
- 大对象 直接由mheap 分配
- **每个mspan中的 object都是 相同大小的。由于每个mspan的size_class都是一定的**，size_class确定了 该mspan 的object大小就是一定的。
- **mcache中会有不同大小的mspan，但是 一个 mcentral中只会有一中大小的mspan， mheap中不会直接去保存mspan**
