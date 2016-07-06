---
layout: post
title: C++11 Memory Model
category: another
---

# C++11 Memory Model

## Memory Modle Basics

### objects and memory locations

* object([cppreference-object](http://en.cppreference.com/w/cpp/language/object)) 
An object, in C++, is a **region of storage**. 内置类型`int`, `float`以及user-defined类型都是object，注意bit fields不是object
* memory location([cppreference-memory-location](http://en.cppreference.com/w/cpp/language/memory_model#Memory_location)).
A memory location is 
    1. an object of scalar type(arithmetic, pointer, pointer to member, enumeration) 
	2. or the largest contiguous sequence of bit fields of non-zero length
	
举个例子：

```
struct S {
    char a;     // memory location #1
    int b : 5;  // memory location #2
    int c : 11, // memory location #2 (continued)
          : 0,
        d : 8;  // memory location #3
    struct {
        int ee : 8; // memory location #4
    } e;
} obj; // The object 'obj' consists of 4 separate memory locations
```

a是个scalar type，因此它自己占一个memory location，b和c是连续的bit fields，它们占同一个memory location。匿名bit field用来强制下一个bit field新占一个memory location，因此d占location 3。ee在新的结构体中因此与d不是连续的，于是占location 4. 结构体e占一个memory location而obj占4个memory location。
因此，c++中的objec可以有1个或多个memory location。

### memory location and concurrency
为什么要先介绍memory location呢，因此C++中的多线程和memory location息息相关。如果两个线程操作不同的memory location，那么没有任何问题；如果两个线程操作相同的memory location，那么就有可能造成race condition了。

为了避免race condition，线程对相同memory location的访问必须有enforced ordering。两种方法可以保证enforced ordering：
1. 在访问前先加锁(mutex)
2. 利用atomic operation的synchronization性质。

### Modification orders
C++程序中的每一个object从它被初始化开始，有一个定义好的modification order, 即所有线程对它的写顺序是定义好的。虽然这个顺序在每次run的时候可能不一样，但是在同一次run的时候，所有线程必须agree on the order。要使所有线程看到的order都是一样的，你必须提供足够的synchronization（如前面说的mutex和atomic operation）。
需要注意的是，虽然所有线程对于同一个object必须agree on the modification order，但对于不同的objects的修改线程不一定要agree on the order（即不同线程看到的修改不同object的顺序可能是不一样的？）

## Synchronizing operations and enforcing ordering

### the synchronizes-with relationship
synchronizes-with关系只出现在原子操作之间。

基本思想是：a suitably tagged atomic write operation W on a variable x synchronizes-with a suitably tagged atomic read operation on x that reads the value stored by either that write (W), or a subsequent atomic write operation on x by the same thread that performed the initial write W, or a sequence of atomic read-modify-write operations on x (such as fetch_add() or compare_exchange_weak()) by any thread, where the value read by the first thread in the sequence is the value written by W. 

简单来说，**如果线程A store a value，线程B read that value，那么A的store操作synchronizes-with B的load操作。**

这里suitably tagged atomic表示需要atomic variable必须tag为Sequentially consistency（后面会讲）

### the happens-before relationship
happens-before:

1. 在单线程中，如果A sequenced before B，那么A happens-before B
2. 在多线程中，如果A inter-thread happens-before B，那么A happens-before B


* 什么是sequenced before？
若operation A和B不在同一句statement中，那么如果A在代码中排在B前面，则A sequenced before B。sequenced before具有传递性。 
若A和B在同一个statement中，这个A和B的evaluate顺序可能是不一定的（[cppreference-sequence-before](http://en.cppreference.com/w/cpp/language/eval_order)）。比如下面代码：

```
#include <iostream>
void foo(int a,int b)
{
    std::cout<<a<<”,”<<b<<std::endl;
}
int get_num()
{
    static int i=0;
    return ++i;
}
int main()
{
    // 哪个get_num()先执行是不一定的. 因此它们没有sequence before关系
    foo(get_num(),get_num());
}
```

* 什么是inter-thread happens-before?
如果一个线程中的operation A synchronizes-with 另一个线程中的operation B，那么A inter-thread happens-beofre B。inter-thread happens-before也具有传递性。

* sequenced before与inter-thread happens-before的关系
如果A sequenced before B，且B inter-thread happens-before C，那么A inter-thread happens-before C。
如果A inter-thread happens-before B，且B sequenced before C，那么A inter-thread happens-before C。

举一个利用synchronizes-with和happens-before的例子：

```
#include <vector>
#include <atomic>
#include <iostream>
std::vector<int> data;
std::atomic<bool> data_ready(false);
void reader_thread()
{　　
    // operation A
    while(!data_ready.load())
    {
        std::this_thread::sleep(std::milliseconds(1));
    }
	// operation B
    std::cout<<”The answer=”<<data[0]<<”\n”;
}
void writer_thread()
{
    // operation C
    data.push_back(42);
	// operation D
    data_ready=true;
}
```

在这个例子中，假如我们不对`data`提供synchronization，那么writer_thread和reader_thread对data的访问那就会造成race condition，产生undefined behavior。因此，我们需要enforced ordering来避免这个问题，这段代码就利用**atomic operation的synchronization性质**来完成enforced ordering。具体地，C sequenced before D, D synchronizes-with A, A sequenced before B, 根据传递性，C happens-before B。因此，完成了enforced ordering。

### Memory ordering for atomic operations
在c++中提供了六种不同的memory ordering，代表了三种memory-ordering models:

1. sequentially consistent ordering (default)
    * memory_order_seq_cst
2. acquire-release ordering
    * memory_order_consume
	* memory_order_acquire
	* memory_order_release
	* memory_order_acq_rel
3. relaxed ordering
    * memory_order_relaxed

不同memory-ordering models在不同CPU上的cost是不一样的。在现代的X86或X86-64架构中，acquire-release甚至sequentially-consistent所需的额外cost非常小。下面介绍这三种model

* sequentially consistent ordering
在SC ordering中，**所有线程看到的operation顺序是一样的。** 从同步角度来看，一个SC的store(write) operation synchronizes-with 一个SC的load(read) operation,并且在load operation之后的任何SC operation也必须在store之后。 这个约束不适用于relaxed memory orderings。

* non-sequentially consistent memory orderings
在non-sequentially的世界里，相同operation在不同线程里会有不同结果。**线程不再agree on the order of events**，也就是operation没有一个全局order了。
唯一的要求是，所有线程**agree on the modification order of each individual variable**。

* relaxed ordering
具有relaxed ordering的atomic type operation没有synchronizes-with关系（这就是为什么说suitably-tagged）。They only guarantee atomicity and modification order consistency.
总结了下有三点要求：
    1. **在一个线程中，对于同一个变量的operation仍然遵守happens-before**，多个线程没有这个要求。
    2. **access to a single atomic variable from the same thread can't be reordered**。即如果一个线程见到了一个变量的值，那它就见不到在这个值之前的值（可以见到之后的值）
    3. **多个线程之间仍然share全局的modification order**.

举个例子来理解这两点，假如每个variable是一个人，他负责按顺序记录(list)你告诉他的value或者告诉你variable的值。如果你让他记录value，他会把value append在list后面，如果你想知道variable的值，它会从list中返回一个值给你。

当你第一次询问variable的值时，他从list中随便给你一个a，如果你再询问一次，他只能给你a或者list里a后面的值（要求的第二点）。假如你让他记录一个value b然后问他variable的值，他会返回b或者list中在b之后的值（要求的第一点）。

假如又有个人来操作这个variable，由于varaible一次只能为一个人服务（原子性），因此他记录value用的list大家看来是一致的（要求的第三点）。 需要注意的是，如果我要求他记录一个value，另一个人询问variable的值的时候，并不是从我的记录的这个value之后的list中返回（线程之间不影响）

可以看出来，relaxed ordering会出现我们完全无法预期的结果，因此最好别用，除非你能牢牢掌握其精髓（tensorflow里用了，所以我看不懂。。）

最后看个code的例子：

```
#include <atomic>
#include <thread>
#include <assert.h>
std::atomic<bool> x,y;
std::atomic<int> z;
void write_x_then_y()
{
    x.store(true,std::memory_order_relaxed);
	y.store(true,std::memory_order_relaxed);
}
void read_y_then_x()
{
    while(!y.load(std::memory_order_relaxed));
    if(x.load(std::memory_order_relaxed))
        ++z;
}
int main()
{
    x=false;
    y=false;
    z=0;
    std::thread a(write_x_then_y);
    std::thread b(read_y_then_x);
    a.join();
    b.join();
    assert(z.load()!=0);
}
```

在这段代码中，z.load()有可能是0. 解释：在a线程中，遵循happens-before关系，x先store，y后store，因此x对应的全局list看起来可能是这样的：false, true, y对应的全局list看起来可能是这样的: true, false. 由于线程b和线程a并没有关系，load y可以是true或者false，load x也可以是true或false。

Reference: Cpp concurrency in action Chapter5

{% include references.md %}
