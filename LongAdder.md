* [引入原因](#%E5%BC%95%E5%85%A5%E5%8E%9F%E5%9B%A0)
* [原理](#%E5%8E%9F%E7%90%86)
* [场景](#%E5%9C%BA%E6%99%AF)
* [实现](#%E5%AE%9E%E7%8E%B0)

### 引入原因

- AtomicLong底层利用CAS来提供并发性，通过不断自旋来更新目标值，直到更新成功。如果并发数较低，冲突概率较小，自旋次数不会很多，如果并发量大，就会出现大量失败并不断自旋的情况，就会有性能瓶颈。
- LongAdder就是为了解决高并发下的AtomicLong自旋瓶颈的问题

### 原理

- AtomicLong内部变量value保存实际的long值，所有操作都是针对该变量进行，高并发环境下就是一个热点key。
- LongAdder基本思路就是**分散热点**，把value分散到一个**Cell数组**中，不同线程会命中到数组的不同槽中，每个线程只针对自己槽中的那个值进行CAS操作，热点key就被分散了。如果想获取真正的long值，就把各个槽中的变量累加返回。
- **类似于ConcurrentHashMap中的分段锁思想**

### 场景

- LongAdder是**空间换时间**的思想，低并发量AtomicLong好，高并发量LongAdder好

- LongAdder只能得到某个时刻的近似值，**不能完全替代AtomicLong**
  $$
  value = base + \sum_{i=1}^nCell[i]
  $$

### 实现

- LongAdder继承了Striped64，Cell是Striped64的静态内部类
- 调用LongAdder的add方法-->判断Cell []是否初始化-->如果没有(既没有出现并发冲突)-->CAS更新base值-->更新失败(发生了并发冲突)-->初始化Cell[]，hash定位到数组的槽位-->更新数据-->调用Striped64的longAccumulate得到结果
- 懒加载，无竞争时只更新base，有竞争时才更新Cell[]



数据结构：数组，链表，栈，队列(阻塞队列)，堆栈(优先级队列)，树(红黑树)，数组+链表(跳表)

并发安全：锁，写时复制，CAS，多实例(ThreadLocal)

性能提升：粒度细化，化整为零再合并；空间换时间

细粒度化：G1