---
title: Java内存区域划分
categories: Java
tags:
  - Java
abbrlink: c3952793
date: 2021-10-10 22:25:08
---
## Native Memory 

在Java中， Native Memory 也叫做direct memory？ 后续我们都以`direct memory`来进行称呼。 

### 什么是direct memory？

Java应用程序执行时会启动一个Java进程，这个进程的用户地址空间可以被分成两份：JVM数据区 + direct memory。

通俗的说，JVM数据区就是Java代码可以直接操作的那部分内存，由heap/stack/pc/method area等组成，GC也工作在这一片区域里。

direct memory则是额外划分出来的一段内存区域，无法用Java代码直接操作，GC无法直接控制direct memory，全靠手工维护。

### Direct Memory相关配置

直接内存的最大大小可以通过 `-XX:MaxDirectMemorySize` 来设置，默认是 64M。

 在 Java 中分配内存的方式一般是通过 `sun.misc.Unsafe`类的公共 native 方法实现的（比如 文件以及网络 IO 类，但是非常不建议开发者使用，使用时一定要确保安全），而类 DirectByteBuffer 类的也是借助于此向物理内存(比如 JVM 运行于 Linux 上，那么 Linux 的内存就被称为物理内存)。

 Unsafe 是位于 sun.misc 包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升 Java 运行效率、增强 Java 语言底层资源操作能力方面起到了很大的作用。但由于 Unsafe 类使 Java 语言拥有了类似 C 语言指针一样操作内存空间的能力，这无疑也增加了程序发生相关指针问题的风险。在程序中过度、不正确使用 Unsafe 类会使得程序出错的概率变大，使得 Java 这种安全的语言变得不再“安全”，因此对 Unsafe 的使用一定要慎重。

 而 ByteBuffer 提供的静态方法：`java.nio.ByteBuffer#allocateDirect` 将 Unsafe 类分配内存的相关操作封装好提供给开发者

### direct memory是怎么来的？

我们且来跟踪一下ByteBuffer.allocateDirect()方法的调用流程：

```
    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }

    // Primary constructor
    //
    DirectByteBuffer(int cap) {                   // package-private
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);//记录已经申请了多少direct memory

        long base = 0;
        try {
            base = unsafe.allocateMemory(size);//申请内存
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);//初始化内存
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));//注册Cleaner
        att = null;
    }
```

其中比较重要的是调用了Unsafe.allocateMemory与Unsafe.setMemory这两个native方法来申请并初始化内存

我们且来跟踪一下这两个方法

Unsafe的实际实现位于[src/share/vm/prims/unsafe.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/prims/unsafe.cpp)

Unsafe.allocateMemory的实现则在[这里](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/prims/unsafe.cpp#l583)：

```
UNSAFE_ENTRY(jlong, Unsafe_AllocateMemory(JNIEnv *env, jobject unsafe, jlong size))
  UnsafeWrapper("Unsafe_AllocateMemory");
  size_t sz = (size_t)size;
  if (sz != (julong)size || size < 0) {
    THROW_0(vmSymbols::java_lang_IllegalArgumentException());
  }
  if (sz == 0) {
    return 0;
  }
  //前面都是检查参数
  sz = round_to(sz, HeapWordSize);//没找到round_to方法的定义，但是应该为了内存对齐而额外申请一点内存做padding
  void* x = os::malloc(sz, mtInternal);//直接调用malloc
  if (x == NULL) {
    THROW_0(vmSymbols::java_lang_OutOfMemoryError());
  }
  //Copy::fill_to_words((HeapWord*)x, sz / HeapWordSize);
  return addr_to_java(x);//将返回的内存地址转成long类型并返回给Java应用
UNSAFE_END
```

可以看到是直接调用了malloc方法来申请的一片内存空间

Unsafe.setMemory的实现在[这里](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/prims/unsafe.cpp#l629)：

```
UNSAFE_ENTRY(void, Unsafe_SetMemory(JNIEnv *env, jobject unsafe, jlong addr, jlong size, jbyte value))
  UnsafeWrapper("Unsafe_SetMemory");
  size_t sz = (size_t)size;
  if (sz != (julong)size || size < 0) {
    THROW(vmSymbols::java_lang_IllegalArgumentException());
  }
  //检查参数
  char* p = (char*) addr_from_java(addr);//将从Java应用传来的long型变量强制转成char指针，现在p指向的就是那一块direct memory的起始位置了
  Copy::fill_to_memory_atomic(p, sz, value);
UNSAFE_END
```

可以看到是调用了[Copy::fill_to_memory_atomic方法](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/utilities/copy.cpp#l57)来将指定的内存空间清空。

 

现在我们就明白了，这些direct memory，其实就跟一般的c语言编程里一样，是直接用malloc方法申请的。

JVM会将malloc方法的返回值（申请到的内存空间的首地址）转换成long类型的address变量，然后返还给Java应用程序。

Java应用程序在需要操作direct memory的时候，会调用native方法将address传给JVM，然后JVM就能对这块内存为所欲为了。

 



## Java应用程序是如何访问direct memory的？

以DirectByteBuffer.get()方法为例

```
    public byte get() {
        return ((unsafe.getByte(ix(nextGetIndex()))));
    }
```

逻辑看起来很简单，就是直接调用Unsafe的getByte方法来从指定的内存地址获取数据（偏移量已经给你算好了，只用取内存数据就行了）

有趣的是，我找了一圈没有发现Unsafe.getByte()方法的native实现，可能是因为这个方法太经常调用了，处于性能缘故JVM已经把它搞成intrinsics的了。

也就是说，跑在JVM内部的Java代码无法直接操作direct memory里的数据，需要经过Unsafe带来的中间层，而这必然也会带来一定的开销，所以操作direct memory比heap memory要慢一些



## 为什么说direct memory更加适合IO操作？

因为在JVM层面来看，所谓的direct memory就是在进程空间中申请的一段内存，而且指向direct memory的指针是**固定不变**的，因此可以直接用direct memory作为参数来执行各种系统调用，比方说read/pread/mmap等。

而为什么heap memory不能直接用于系统IO呢，因为GC会移动heap memory里的对象的位置。如果强行用heap memory来搞系统IO的话，IO操作的中途出现的GC会导致缓冲区位置移动，然后程序就跑飞了。

除非采用一定的手段将这个对象pin住，但是hotspot不提供单个对象层面的object pinning，一定要pin的话就只能暂时禁用gc了，也就是把整个Java堆都给pin住，这显然代价太高了。

总结一下就是：heap memory不可能直接用于系统IO，数据只能先读到direct memory里去，然后再复制到heap memory。



## 实例说明

就用[上一篇](http://www.cnblogs.com/stevenczp/p/7506033.html)中提到的FileChannel.read()方法作为例子，而且使用heap memory作为缓冲区，其调用流程如下：

1. 先申请一块临时的direct memory

2. 调用native的FileDispatcherImpl.pread0或者FileDispatcherImpl.read0，将step1中申请的direct memory的地址传进去

3. jvm调用Linux提供的read或者pread系统调用，传入direct memory对应的内存空间指针，以及正在操作的fd

4. **触发中断，进程从用户态进入到内核态（1-3步全是在用户态中完成）**

5. 操作系统检查kernel中维护的buffer cache是否有数据，如果没有，给磁盘发送命令，让磁盘将数据拷贝到buffer cache里

6. 操作系统将buffer cache中的数据复制到step3中传入的指针对应的内存里
7. **触发中断，进程从内核态退回到用户态（5-6步全在内核态中完成）**

8. FileDispatcherImpl.pread0或者FileDispatcherImpl.read0方法返回，此时临时创建的direct memory中已经有用户需要的数据了

9. 将direct memory里的数据复制到heap memory中（这中间又要调用Unsafe里的一些方法，例如copyMemory）

10. 现在heap memory中终于有我们想要的数据了。

总结一下，数据的流转过程是：hard disk -> kernel buffer cache -> direct memory -> heap memory


