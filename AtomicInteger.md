### 概述

1. `AtomicInteger `是Java提供的原子操作类，其内部通过` UnSafe`工具类，使用 `CAS(compare and set)`的方式保证更新操作的原子性。
2. `CAS`可以看成是一种乐观锁的实现方式，每次更新前会检查当前的值是否符合预期，符合才进行更新；其依赖于底层硬件的指令支持，不同的体系结构的处理器不一样。因为其锁的粒度低，所以在并发下能够提供更优秀的吞吐量表现。

#### 属性

```java
    /** 序列化id */
	private static final long serialVersionUID = 6214790243416807050L;

    // 获取Unsafe实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();

	//标识value字段的偏移量
    private static final long valueOffset;
	
	//静态代码块，通过unsafe获取value的偏移量
    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

	//存储int类型值的地方
    private volatile int value;
```

#### 构造器

```java
    /** 根据指定的初始化值创建新的AtomicInteger */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    /** 创建新的AtomicInteger，厨初始值为0 */
    public AtomicInteger() {
    }
```

#### 核心方法

```java
    /** 自动将当前值 +1 */
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

	/**
	 * 这是Unsafe类的做法，首先获取了对象this对应的offset内存偏移的值。也就是当前值：var5。接下来在		 * while里进行判断，如果该offset内存偏移对应的值是var5，即没有被更改过，那么就更新成var5+var4
	 * 如果没更新成功，那么就直接返回啦。
	 * 
	 * 很显然，一个cas的做法，采用了乐观锁的概念。
	 */
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

