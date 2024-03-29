---
title: Gc回收机制与分代回收策略
date: 2023-06-13 15:32:50
tags: JVM
categories: 
- Java
- JVM
---

## GC回收机制

### **简介**

GC是Garbage Collection垃圾回收的英文释义

什么是垃圾呢，垃圾是指内存中无用的对象。如何判断那些对象是无用对象呢，可以通过可达性分析来判断。

那什么是可达性分析呢，可达性分析是指将内存中的对象引用关系用一张图来表示，GC Root对象作为顶点，对象引用关系连接线走过的路径叫做引用链，如果一个对象的引用链对Gc Root是不可达的那就认为这个对象是无用对象。



### **Gc Root**

- 虚拟机栈局部变量表中引用的对象
- 方法区中静态引用的对象
- 存活的线程对象
- Native方法中JNI引用的对象



### **何时回收**

不同虚拟机会有不同的gc回收机制，一般来说以下两种情况都会触发垃圾回收：

1.Allocation Failure：当内存过低导致内存分配时，会触发一次gc

2.System.gc():应用层主动调用gc api



### **回收算法**

gc回收算法常见的有三种分别是标记清除、复制清除、标记压缩

#### 标记清除

标记清除分为两个阶段

标记阶段：

从顶点的Gc Root对象开始遍历引用链，将垃圾对象标记为黑色，否则标记为灰色

清除阶段：

标记完成后，将垃圾对象全部清除

优点：操作简单

缺点：可能会产生内存碎片





#### 复制清除

将内存分为AB两块，每次只使用一块内存，将A内存设置为可用内存，gc回收时遍历Gc Root引用链进行标记，

标记完成后，将存活对象复制到B内存块中，将A内存中的对象全部清除，将B内存设置为可用内存

优点：运行高效，不会产生内存碎片

缺点：可用内存会变为原来的一半



#### 标记压缩

标记压缩也分为两个阶段

标记阶段：

遍历Gc Root引用链，标记无用对象为黑色否则为灰色

压缩阶段：

将存活对象压缩到内存的另一端，清除垃圾对象

优点：不会产生内存碎片，不会造成内存浪费

缺点：对象需要移动，一定程度上造成性能损失



## 分代回收策略

虚拟机根据对象存活周期不同将堆内存分为几块，一般来说会分为新生代和老年代。新生代又分为eden、survivior0、survivor2三个区，新生代多次回收后还存活的对象会进入老年代。

> Hotspot除了新生代和老年代还有一个永久代

<img src="https://github.com/hezd/Image-Folders/blob/master/Blog/%E5%88%86%E4%BB%A3%E7%AD%96%E7%95%A5.png?raw=true" style="zoom:90%;" />

### 新生代

新建的对象优先分配到新生代

新生代对象会优先分配到Eden区，当Eden区满了以后，会将存活的对象复制到s0区并清空Eden区，当Eden区第二次满的时候，会将Eden区和s0去的存活对象复制到s1区并清空Eden区和s0区，s0和s1多次切换后（默认15次）还有存活的对象会进入老年代。

从新生代的回收过程可以看到使用的是复制清除算法

### 老年代

新生代多次回收后还存活的对象会进入老年代，新生代分配大对象时如果内存不足也会进入老年代

