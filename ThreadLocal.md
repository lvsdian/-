* [静态内部类](#%E9%9D%99%E6%80%81%E5%86%85%E9%83%A8%E7%B1%BB)
  * [ThreadLocalMap](#threadlocalmap)
    * [静态内部类](#%E9%9D%99%E6%80%81%E5%86%85%E9%83%A8%E7%B1%BB-1)
      * [Entry](#entry)
    * [属性](#%E5%B1%9E%E6%80%A7)
      * [Entry [] table](#entry--table)
* [实例分析](#%E5%AE%9E%E4%BE%8B%E5%88%86%E6%9E%90)
  * [实例](#%E5%AE%9E%E4%BE%8B)
  * [get()](#get)
  * [get()分析总结](#get%E5%88%86%E6%9E%90%E6%80%BB%E7%BB%93)
  * [set()](#set)
  * [根据key计算下标](#%E6%A0%B9%E6%8D%AEkey%E8%AE%A1%E7%AE%97%E4%B8%8B%E6%A0%87)
  * [remove()](#remove)
* [ThreadLocal内存泄露](#threadlocal%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2)
  * [为什么使用弱引用](#%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BD%BF%E7%94%A8%E5%BC%B1%E5%BC%95%E7%94%A8)
  * [线程池](#%E7%BA%BF%E7%A8%8B%E6%B1%A0)

### 静态内部类

####`SuppliedThreadLocal`

- `SuppliedThreadLocal`继承了`ThreadLocal`

#### `ThreadLocalMap`

- `ThreadLocalMap`是一个哈希表，表里面是`Entry`对象，通过`Entry`来存储数据。

##### 静态内部类

###### Entry

- ```java
          static class Entry extends WeakReference<ThreadLocal<?>> {
              /** The value associated with this ThreadLocal. */
              Object value;
        
              Entry(ThreadLocal<?> k, Object v) {
                  super(k);
                  value = v;
              }
          }
  ```

- `Entry`是`ThreadLocalMap`的静态内部类，是一个**弱引用**。

- `Entry`以键值对的形式存在，`key`是`ThreadLocal`类型。

##### 属性

###### `Entry [] table`

```java
        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;
```

- `Entry`数组可扩容，那么就涉及到扩容因子

  ```java
          //扩容因子
          private int threshold; // Default to 0
  
          //Set the resize threshold to maintain at worst a 2/3 load factor
          private void setThreshold(int len) {
              threshold = len * 2 / 3;
          }
  ```

- `Entry`数组初始容量

  ```java
          //The initial capacity -- MUST be a power of two.
          private static final int INITIAL_CAPACITY = 16;
  ```

### 实例分析

#### 实例

```java
public class ThreadLocalDemo {
    private static int a = 10;
    private static ThreadLocal<Integer> local;

    /**
     * result:
     *  Thread-0初始取值20
     *  Thread-0remove后再取值：null
     *  Thread-2直接取值：null
     */
    public static void main(String[] args) {
        //启动2个线程
        new Thread(new ThreadA()).start();
        new Thread(new ThreadB()).start();
    }
    static class ThreadA implements Runnable {
        @Override
        public void run() {
            local = new ThreadLocal<>();
            //线程A给ThreadLocal对象设置值
            local.set(a + 10);
            //线程A取值
            System.out.println(Thread.currentThread().getName() + "初始取值" + local.get());
            local.remove();
            //线程A remove后再取值
            System.out.println(Thread.currentThread().getName() + "remove后再取值：" + local.get());
        }
    }
     static class ThreadB extends Thread {
        @Override
        public void run() {
            //线程B直接取值
            System.out.println(Thread.currentThread().getName() + "直接取值：" + local.get());
        }
    }
}
```

#### `get()`

  ```java
     public T get() {
          Thread t = Thread.currentThread();
          //根据当前线程获取ThreadLocalMap
          ThreadLocalMap map = getMap(t);
          if (map != null) {
              //如果map不为null并且map的Entry不为null,返回Entry的value
              ThreadLocalMap.Entry e = map.getEntry(this);
              if (e != null) {
                  @SuppressWarnings("unchecked")
                  T result = (T)e.value;
                  return result;
              }
          }
         //否则设置初始值并返回
          return setInitialValue();
    }
  
    //根据当前线程获取ThreadLocalMap
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
  
    //静态内部类ThreadLocalMap的方法，以ThreadLocal对象为key获取value
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
     }
    
    //设置初始值
    private T setInitialValue() {
        //设置初始值value为null
        T value = initialValue();
        Thread t = Thread.currentThread();
        //根据当前线程获取ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        //要么给map设置键值，要么创建新的map并设置键值
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
  
    protected T initialValue() {
        return null;
    }
    
    //根据当前线程获取的ThreadLocalMap为null，就创建一个map，这个map的key为当前ThreadLocal对象，value为传进来的value
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
	
	//ThreadLocalMap构造方法，用Entry键值对来存数据
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
  ```

#### `get()`分析总结

- `get()`流程
  1. 获取`currentThread`，根据`currentThread`获取`ThreadLocalMap`对象。**`Thread`类中本身就有一个`ThreadLocal.ThreadLocalMap`类型的属性` threadLocals`。**
  2. 如果`map`不为空并且`map`的`Entry`不为空就以当前`ThreadLocal`对象为`key`获取`value`并返回。
  3. `map`为空就返回`null`，并创建一个`ThreadLocalMap`，它的`Entry`的`key`为当前`ThreadLocal`，`value`为`null`
  4. `map`不为空但`map`的`Entry`为空，设置当前`map`的`Entry`，`key`为当前`ThreadLocal`，`value`为`null`
- 总结
  1. 每个`Thread`内部维护了一个`ThreadLocalMap`，`ThreadLocalMap`是`ThreadLocal`的静态内部类。
  2. 只有当前线程才能找到当前线程的`ThreadLocalMap`，这就实现了线程隔离。
  3. `ThreadLocalMap`是一个哈希表，通过它的静态内部类`Entry`来存储数据。
  4. `Entry`为键值对结构，`key`为`ThreadLocal`类型。
  5. **`Entry`继承了`WeakReference`接口，是一个弱引用类型。**
  6. **如果有两个线程，持有同一个`ThreaLocal`的实例，这样的情况也就是`Entry `对象持有的`ThreadLocal`的弱引用是一样的，但是两个线程的`ThreadLocalMap`是 不同的，`ThreadLocalMap`是每个线程单独维护的。**

#### `set()`

```java
        //ThreadLocal的set()方法
        public void set(T value) {
            Thread t = Thread.currentThread();
            //获取当前线程的ThreadLocalMap
            ThreadLocalMap map = getMap(t);
            if (map != null)
                //ThreadLocalMap的set()方法
                map.set(this, value);
            else
                createMap(t, value);
        }

        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.
            
			//拿到Entry数组
            Entry[] tab = table;
            int len = tab.length;
            //计算新的下标值来存放键值对
            int i = key.threadLocalHashCode & (len-1);
			//根据下标值得到Entry键值对
            for (Entry e = tab[i];
                 e != null;     		
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                //key不为null但与当前threadLocal不是同一对象，后移一位再判断
				//key不为null并且与当前的threadLocal是同一对象，更新键值对
                if (k == key) {
                    e.value = value;
                    return;
                }
				//key为null，添加键值对
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
			//创建具体对象
            tab[i] = new Entry(key, value);
            //长度+1
            int sz = ++size;
            //判断是否需要扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
		//扩容
        private void rehash() {
           	////删除key为null的Entry防止内存泄露
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            //已使用量 >= 3/4阈值就扩容
            if (size >= threshold - threshold / 4)
                resize();
        }
```

#### 根据`key`计算下标

```java
//i即key在Entry中的下标，注意key是ThreadLocal类型，所以以下都是ThreadLocal中的属性和方法
int i = key.threadLocalHashCode & (len-1);

private final int threadLocalHashCode = nextHashCode();

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;
```

####  `remove()`

```java
//ThreadLocal的remove()方法
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
//ThreadLocalMap的remove方法，删除Entry的key
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            //删除key为null的Entry防止内存泄露
            expungeStaleEntry(i);
            return;
        }
    }
}
```



### `ThreadLocal`内存泄露

-  内存泄露：已经不再使用的内存没有得到释放。

- 弱引用在垃圾回收的时候会被回收掉，无论内存空间是否足够都会被回收。
- `Entry`的`key`是一个弱引用的`ThreadLocal`对象，所以在垃圾回收时会回收这个`key`，那么`Entry`中会出现`key`为`null`的`Entry`，这些`key`的`value`不会再被访问但这个`Entry`又被`ThreadLocalMap`所引用，所以在`currentThread`结束前占用内存不会被释放，因此导致内存泄露。
- 因为以上会导致内存泄露，所以`get()`、`set()`、`remove()`方法中均有实现回收`key`为`null`的`Entry`

#### 为什么使用弱引用

- 如果`key`为强引用，引用`ThreadLocal`的对象被回收了，但`ThreadLocalMap`中仍有`ThreadLoal`的强引用，如果没有手动删除，就不会被回收，如果线程生命很长，就会导致内存泄露。
- 如果`key`为弱引用，引用`ThreadLocal`的对象被回收了，但`ThreadLocalMap`中持有`ThreadLoal`的弱引用，如果没有手动删除，会自动回收，对应的`value`在`get()`、`set()`、`remove()`时会删除，如果线程生命很长，不会导致内存泄露。

#### 线程池

- 使用了线程池，可以达到“线程复用”的效果。但是归还线程之前记得清除`ThreadLocalMap`，要不然再取出该线程的时候，`ThreadLocal`变量还会存在。这就不仅仅是内存泄露的问题了，整个业务逻辑都可能会出错。

- 参考方案

  - 重写 `ThreadPoolExecutor#afterExecute(r, t)}`方法，对`ThreadLocalMap`进行清理，比如：

    ```java
    protected void afterExecute(Runnable r, Throwable t) { 
        // you need to set this field via reflection.
        Thread.currentThread().threadLocals = null;
    }
    
    ```

    