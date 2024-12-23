---
title: golang GC 垃圾回收机制
date: '2021-02-20T00:00:00+08:00'
tags:
- go
showToc: true
categories:
- go
---



垃圾回收(Garbage Collection，简称GC)是编程语言中提供的自动的内存管理机制，自动释放不需要的对象，让出存储器资源，无需程序员手动执行。

   Golang中的垃圾回收主要应用三色标记法，GC过程和其他用户goroutine可并发运行，但需要一定时间的**STW(stop the world)**，STW的过程中，CPU不执行用户代码，全部用于垃圾回收，这个过程的影响很大，Golang进行了多次的迭代优化来解决这个问题。

## Go V1.3之前的标记-清除(mark and sweep)算法

此算法主要有两个主要的步骤：

- 标记(Mark phase)
- 清除(Sweep phase)

**第一步**，暂停程序业务逻辑, 找出不可达的对象，然后做上标记。第二步，回收标记好的对象。

操作非常简单，但是有一点需要额外注意：mark and sweep算法在执行的时候，需要程序暂停！即 `STW(stop the world)`。也就是说，这段时间程序会卡在哪儿。

![](/images/e7452cc0-956d-44ea-8ef6-22bad882f1d8.png)

**第二步**, 开始标记，程序找出它所有可达的对象，并做上标记。如下图所示：

![](/images/a2d30455-7d6a-4039-8091-c369e405ce2b.png)

**第三步**,  标记完了之后，然后开始清除未标记的对象. 结果如下.

![](/images/532f5acd-aac6-4a12-858c-4f79c7eeba24.png)

**第四步**, 停止暂停，让程序继续跑。然后循环重复这个过程，直到process程序生命周期结束。

## 标记-清扫(mark and sweep)的缺点

- STW，stop the world；让程序暂停，程序出现卡顿 **(重要问题)**。
- 标记需要扫描整个heap
- 清除数据会产生heap碎片

所以Go V1.3版本之前就是以上来实施的,  流程是

![](/images/c24c6980-7fb9-4c34-8576-064bef2b2ba1.png)

Go V1.3 做了简单的优化,将STW提前, 减少STW暂停的时间范围.如下所示

![](/images/3cfb6c99-91b8-4693-9446-203000b2e3de.png)

**这里面最重要的问题就是：mark-and-sweep 算法会暂停整个程序** 。

Go是如何面对并这个问题的呢？接下来G V1.5版本 就用**三色并发标记法**来优化这个问题.

## Go V1.5的三色并发标记法

三色标记法 实际上就是通过三个阶段的标记来确定清楚的对象都有哪些. 我们来看一下具体的过程.

**第一步** , 就是只要是新创建的对象,默认的颜色都是标记为“白色”.

![](/images/f97b6289-f495-4afd-a3b7-f89c6f106d82.png)

这里面需要注意的是, 所谓“程序”, 则是一些对象的跟节点集合.

![](/images/16192886-775b-461a-b9d0-d74a2f3f76d0.png)

所以上图,可以转换如下的方式来表示.

**第二步**, 每次GC回收开始, 然后从根节点开始遍历所有对象，把遍历到的对象从白色集合放入“灰色”集合。

![](/images/89041a5e-2ff0-484f-acb5-486aa84cc0f1.png)

**第三步**, 遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合

![](/images/3451b546-570c-4995-9eab-2eb50f5b25cf.png)

**第四步**, 重复**第三步**, 直到灰色中无任何对象.

![](/images/8cdbacc0-3b34-46c8-9f49-89f9bda89902.png)

![](/images/fcee2e45-8d1f-4421-a38a-6310ea5f7620.png)

**第五步**: 回收所有的白色标记表的对象. 也就是回收垃圾.

![](/images/1a47f5e7-acc4-4dbe-810e-379695bdc531.png)

以上便是`三色并发标记法`, 不难看出,我们上面已经清楚的体现`三色`的特性, 那么又是如何实现并行的呢?

> Go是如何解决标记-清除(mark and sweep)算法中的卡顿(stw，stop the world)问题的呢？

## 没有STW的三色标记法

​       我们还是基于上述的三色并发标记法来说, 他是一定要依赖STW的. 因为如果不暂停程序, 程序的逻辑改变对象引用关系, 这种动作如果在标记阶段做了修改，会影响标记结果的正确性。我们举一个场景.

如果三色标记法, 标记过程不使用STW将会发生什么事情?

------

![](/images/21516ce2-d194-4394-9b62-9c36563d4b2b.png)

![](/images/d6592a51-f284-4fbc-905c-50dc59d92103.png)

![](/images/b107beef-233d-4968-8b55-ab2dd42cad4f.png)

![](/images/a10a9998-e626-4b6c-8aa9-4030238f28b1.png)

![](/images/787819ca-f386-4c30-bc51-819ad368dfdb.png)

可以看出，有两个问题, 在三色标记法中,是不希望被发生的

- 条件1: 一个白色对象被黑色对象引用**(白色被挂在黑色下)**
- 条件2: 灰色对象与它之间的可达关系的白色对象遭到破坏**(灰色同时丢了该白色)**

当以上两个条件同时满足时, 就会出现对象丢失现象!

   当然, 如果上述中的白色对象3, 如果他还有很多下游对象的话, 也会一并都清理掉.

​       为了防止这种现象的发生，最简单的方式就是STW，直接禁止掉其他用户程序对对象引用关系的干扰，但是**STW的过程有明显的资源浪费，对所有的用户程序都有很大影响**，如何能在保证对象不丢失的情况下合理的尽可能的提高GC效率，减少STW时间呢？

​       答案就是, 那么我们只要使用一个机制,来破坏上面的两个条件就可以了.

## 屏障机制

   我们让GC回收器,满足下面两种情况之一时,可保对象不丢失. 所以引出两种方式.

### “强-弱” 三色不变式

- 强三色不变式

不存在黑色对象引用到白色对象的指针。

![](/images/c7612fff-702e-4ffe-8055-dcc7d93b0fea.png)

- 弱三色不变式

所有被黑色对象引用的白色对象都处于灰色保护状态.

![](/images/466357c5-9a01-40f8-9596-e1b767033d20.png)

为了遵循上述的两个方式,Golang团队初步得到了如下具体的两种屏障方式“插入屏障”, “删除屏障”.

### 插入屏障

`具体操作`: 在A对象引用B对象的时候，B对象被标记为灰色。(将B挂在A下游，B必须被标记为灰色)

`满足`: **强三色不变式**. (不存在黑色对象引用白色对象的情况了， 因为白色会强制变成灰色)

伪码如下:

```go
添加下游对象(当前下游对象slot, 新下游对象ptr) {   
  //1
  标记灰色(新下游对象ptr)   
  //2
  当前下游对象slot = 新下游对象ptr                   
}
```

场景：

```go
A.添加下游对象(nil, B)   //A 之前没有下游， 新添加一个下游对象B， B被标记为灰色
A.添加下游对象(C, B)     //A 将下游对象C 更换为B，  B被标记为灰色
```

​       这段伪码逻辑就是写屏障,. 我们知道,黑色对象的内存槽有两种位置, `栈`和`堆`. 栈空间的特点是容量小,但是要求相应速度快,因为函数调用弹出频繁使用, 所以“插入屏障”机制,在**栈空间的对象操作中不使用**. 而仅仅使用在堆空间对象的操作中.

​       接下来，我们用几张图，来模拟整个一个详细的过程， 希望您能够更可观的看清晰整体流程。

------

![](/images/82e4e137-4b21-4a48-a2c2-0ca38eec9d7d.png)

![](/images/63488ff2-4d52-4a77-b55b-2052b9c66b30.png)

------

![](/images/f1392990-751d-4b0f-9eeb-69697d821aad.png)

------

![](/images/2e4c728d-3d54-403c-b8e2-e2e19bb97ae2.png)

![](/images/7b3ec6ef-1694-4ba4-b120-6066451f6e44.png)

![](/images/a5bf6b22-b34b-4c7e-81c3-7e422d560825.png)

------

   但是如果栈不添加,当全部三色标记扫描之后,栈上有可能依然存在白色对象被引用的情况(如上图的对象9).  所以要对栈重新进行三色标记扫描, 但这次为了对象不丢失, 要对本次标记扫描启动STW暂停. 直到栈空间的三色标记结束.

------

![](/images/0805cff3-e89e-4eed-a40c-31d8af5d8ad4.png)

------

![](/images/c5f528e4-d7ae-4249-9370-78e514144b0a.png)

![](/images/b19b22ef-8484-4708-979a-2d1cbdefc23b.png)

------

最后将栈和堆空间 扫描剩余的全部 白色节点清除.  这次STW大约的时间在10~100ms间.

------

![](/images/446787fa-1ad8-451b-a8f3-e87ca8f46b82.png)

------

### 删除屏障

`具体操作`: 被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。

`满足`: **弱三色不变式**. (保护灰色对象到白色对象的路径不会断)

伪代码：



```go
添加下游对象(当前下游对象slot， 新下游对象ptr) {
  //1
  if (当前下游对象slot是灰色 || 当前下游对象slot是白色) {
        标记灰色(当前下游对象slot)     //slot为被删除对象， 标记为灰色
  }
  //2
  当前下游对象slot = 新下游对象ptr
}
```

场景：

```go
A.添加下游对象(B, nil)   //A对象，删除B对象的引用。  B被A删除，被标记为灰(如果B之前为白)
A.添加下游对象(B, C)       //A对象，更换下游B变成C。   B被A删除，被标记为灰(如果B之前为白)
```

接下来，我们用几张图，来模拟整个一个详细的过程， 希望您能够更可观的看清晰整体流程。

------

![](/images/37cec983-9229-45e1-b5b1-cc17cefaebd3.png)

------

![](/images/f3affdd8-36d5-4c28-9efb-932e8eafe855.png)

------

![](/images/231684ca-812d-4ce5-af4f-772503d1be63.png)

![](/images/ed0497f6-15b1-4375-bf60-2df7a1af08b8.png)

![](/images/b51da9dd-7b88-403e-a4e7-1e1c3614e897.png)

![](/images/430453b0-b9f1-420a-bfb4-5c6e8cc6deda.png)

![](/images/e1546bc1-a282-4746-94c6-fc468329e7d3.png)

------

这种方式的回收精度低，一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮GC中被清理掉。

## Go V1.8的混合写屏障(hybrid write barrier)机制

插入写屏障和删除写屏障的短板：

- 插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；
- 删除写屏障：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。

Go V1.8版本引入了混合写屏障机制（hybrid write barrier），避免了对栈re-scan的过程，极大的减少了STW的时间。结合了两者的优点。

------

### 混合写屏障规则

`具体操作`:

1、GC开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需STW)，

2、GC期间，任何在栈上创建的新对象，均为黑色。

3、被删除的对象标记为灰色。

4、被添加的对象标记为灰色。

`满足`: 变形的**弱三色不变式**.

伪代码：

```go
添加下游对象(当前下游对象slot, 新下游对象ptr) {
    //1 
        标记灰色(当前下游对象slot)    //只要当前下游对象被移走，就标记灰色
    //2 
    标记灰色(新下游对象ptr)    
    //3
    当前下游对象slot = 新下游对象ptr
}
```

> 这里我们注意， 屏障技术是不在栈上应用的，因为要保证栈的运行效率。

### 混合写屏障的具体场景分析

接下来，我们用几张图，来模拟整个一个详细的过程， 希望您能够更可观的看清晰整体流程。

> 注意混合写屏障是Gc的一种屏障机制，所以只是当程序执行GC的时候，才会触发这种机制。

#### GC开始：扫描栈区，将可达对象全部标记为黑

![](/images/cd30cf15-058a-4a64-a2f8-775c87dfea89.png)

![](/images/a9c3a537-6c7b-4d07-8fc7-6f1c4c181cfb.png)

------

#### 场景一： 对象被一个堆对象删除引用，成为栈对象的下游

> 伪代码

```go
//前提：堆对象4->对象7 = 对象7；  //对象7 被 对象4引用
栈对象1->对象7 = 堆对象7；  //将堆对象7 挂在 栈对象1 下游
堆对象4->对象7 = null；    //对象4 删除引用 对象7
```

![](/images/ebaedc9a-8cd0-4598-8962-2f2e7fb3ac36.png)

------

![](/images/a3066126-ad87-44a0-b712-4e80b69ad25d.png)

#### 场景二： 对象被一个栈对象删除引用，成为另一个栈对象的下游

> 伪代码

```go
new 栈对象9；
对象8->对象3 = 对象3；      //将栈对象3 挂在 栈对象9 下游
对象2->对象3 = null；      //对象2 删除引用 对象3
```

------

![](/images/4899f7b2-d951-4d9c-9880-6fb79fa01fee.png)

![](/images/a185fd86-0ca0-4294-9ebf-6c8a73f8af19.png)

![](/images/eb499138-8d6a-4c75-a4a9-1dad614879bf.png)

------

#### 场景三：对象被一个堆对象删除引用，成为另一个堆对象的下游

> 伪代码

```go
堆对象10->对象7 = 堆对象7；       //将堆对象7 挂在 堆对象10 下游
堆对象4->对象7 = null；         //对象4 删除引用 对象7
```

------

![](/images/cb9a84fe-73ec-43e7-9d37-f14683e4f6e4.png)

------

![](/images/e010d5a5-4867-43df-8e82-1a4368f2c8e2.png)

![](/images/d11efffb-50de-4aed-b6a6-06eb20a4824e.png)

------

#### 场景四：对象从一个栈对象删除引用，成为另一个堆对象的下游

> 伪代码

```go
堆对象10->对象7 = 堆对象7；       //将堆对象7 挂在 堆对象10 下游
堆对象4->对象7 = null；         //对象4 删除引用 对象7
```

------

![](/images/29e311a4-4387-4387-be10-06bf33338c01.png)

![](/images/ba80f06b-42a9-463d-b88f-62ee6f83bc4d.png)

![](/images/497960cd-b419-430a-90e5-88f2e37bc848.png)

------

​       Golang中的混合写屏障满足`弱三色不变式`，结合了删除写屏障和插入写屏障的优点，只需要在开始时并发扫描各个goroutine的栈，使其变黑并一直保持，这个过程不需要STW，而标记结束后，因为栈在扫描后始终是黑色的，也无需再进行re-scan操作了，减少了STW的时间。

## 总结

​       以上便是Golang的GC全部的标记-清除逻辑及场景演示全过程。

GoV1.3- 普通标记清除法，整体过程需要启动STW，效率极低。

GoV1.5- 三色标记法， 堆空间启动写屏障，栈空间不启动，全部扫描之后，需要重新扫描一次栈(需要STW)，效率普通

GoV1.8-三色标记法，混合写屏障机制， 栈空间不启动，堆空间启动。整个过程几乎不需要STW，效率较高。

## 文章转自

https://www.jianshu.com/p/4c5a303af470