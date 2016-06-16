# GCC 提供的原子操作

## Memory barrier 简介

程序在运行时内存实际的访问顺序和程序代码编写的访问顺序不一定一致，这就是内存乱序访问。内存乱序访问行为出现的理由是为了提升程序运行时的性能。内存乱序访问主要发生在两个阶段：

    编译时，编译器优化导致内存乱序访问（指令重排）
    运行时，多 CPU 间交互引起内存乱序访问
Memory barrier 能够让 CPU 或编译器在内存访问上有序。一个 Memory barrier 之前的内存访问操作必定先于其之后的完成。Memory barrier 包括两类：

    编译器 barrier
    CPU Memory barrier

Memory barrier 常用场合包括：

    实现同步原语（synchronization primitives）
    实现无锁数据结构（lock-free data structures）
    驱动程序
实际的应用程序开发中，开发者可能完全不知道 Memory barrier 就可以开发正确的多线程程序，这主要是因为各种同步机制中已经隐含了 Memory barrier（但和实际的 Memory barrier 有细微差别），这就使得不直接使用 Memory barrier 不会存在任何问题。但是如果你希望编写诸如无锁数据结构，那么 Memory barrier 还是很有用的。


在 Linux 内核中，除了前面说到的编译器 barrier — barrier() 和 ACCESS_ONCE()，还有 CPU Memory barrier：

    通用 barrier，保证读写操作有序的，mb() 和 smp_mb()
    写操作 barrier，仅保证写操作有序的，wmb() 和 smp_wmb()
    读操作 barrier，仅保证读操作有序的，rmb() 和 smp_rmb()

## GCC提供的原子操作
gcc从4.1.2提供了__sync_*系列的built-in函数，用于提供加减和逻辑运算的原子操作。应用实例可以参考ceph源代码的common/simple_spin.h，一个ceph中实现的自旋锁。

    type __sync_fetch_and_add (type *ptr, type value, ...)
    type __sync_fetch_and_sub (type *ptr, type value, ...)
    type __sync_fetch_and_or (type *ptr, type value, ...)
    type __sync_fetch_and_and (type *ptr, type value, ...)
    type __sync_fetch_and_xor (type *ptr, type value, ...)
    type __sync_fetch_and_nand (type *ptr, type value, ...)

    type __sync_add_and_fetch (type *ptr, type value, ...)
    type __sync_sub_and_fetch (type *ptr, type value, ...)
    type __sync_or_and_fetch (type *ptr, type value, ...)
    type __sync_and_and_fetch (type *ptr, type value, ...)
    type __sync_xor_and_fetch (type *ptr, type value, ...)
    type __sync_nand_and_fetch (type *ptr, type value, ...)
type可以是1,2,4或8字节长度的int类型，即：

    int8_t / uint8_t
    int16_t / uint16_t
    int32_t / uint32_t
    int64_t / uint64_t



后面的可扩展参数(...)用来指出哪些变量需要memory barrier,因为目前gcc实现的是full barrier（类似于linux kernel 中的mb(),表示这个操作之前的所有内存操作不会被重排序到这个操作之后）,所以可以略掉这个参数。

    bool __sync_bool_compare_and_swap (type *ptr, type oldval type newval, ...)
    type __sync_val_compare_and_swap (type *ptr, type oldval type newval, ...)

这两个函数提供原子的比较和交换，如果*ptr == oldval,就将newval写入*ptr,
第一个函数在相等并写入的情况下返回true.
第二个函数在返回操作之前的值。

    __sync_synchronize (...)

发出一个full barrier.
