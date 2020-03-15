

* [内部类](#%E5%86%85%E9%83%A8%E7%B1%BB)
* [属性](#%E5%B1%9E%E6%80%A7)
* [构造方法](#%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95)
* [ReentrantLock\#lockLock](#reentrantlocklocklock)
* [AQS的静态内部类\-Node](#aqs%E7%9A%84%E9%9D%99%E6%80%81%E5%86%85%E9%83%A8%E7%B1%BB-node)
* [ReentrantLock\#unLock](#reentrantlockunlock)
* [condition](#condition)

#### 内部类

- `ReentrantLock`有3个内部类，`Sync`继承了`AQS`，`NonfairSync`和`FairSync`为`Sync`的子类。通过构造器参数`fair`来指定使用公平锁还是非公平锁。

- 使用公平锁时，会让活跃的线程得不到锁，进入等待状态，频繁上下文切换，从而降低整体效率

```java
    /** 
     * 是ReentrantLock内部类，但继承了AQS
     *
     * Sync是ReentrantLock实现同步控制的基础,非公平锁和公平锁都是其子类。 用AQS的state来表示获取的		 * 锁的数量 
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        //序列号
        private static final long serialVersionUID = -5179523762034025860L;

        /** 获取锁的方法 */
        abstract void lock();

        /**
         * 非公平锁的tryLock
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //state表示是否被锁和持有数量
            int c = getState();
            //没锁
            if (c == 0) {
                //cas获取锁
                if (compareAndSetState(0, acquires)) {
                    //获取成功后，将exclusiveOwnerThread属性设为当前线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //已经被锁，并且是当前线程获得了锁
            else if (current == getExclusiveOwnerThread()) {
                //增加锁的数量
                int nextc = c + acquires;
                if (nextc < 0) 
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //这种情况即其他线程获得了锁，返回false
            return false;
        }

		//释放锁操作
        protected final boolean tryRelease(int releases) {
            //当前锁数量-要释放的锁数量
            int c = getState() - releases;
            //获取锁的线程不是当前线程，抛异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            //free局部变量标识是否已经释放锁
            boolean free = false;
            //锁释放完了
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            //锁释放了，但没有释放完
            setState(c);
            return free;
        }

        //判断获得锁的线程是不是当前线程
        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
        
		//创建一个新的环境
        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        //得到获得锁的的线程
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        //获得持有锁的数量
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        //是否被锁
        final boolean isLocked() {
            return getState() != 0;
        }

        /** 从流中重建一个实例 */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

    /** 非公平锁 */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /** 锁 */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    /** 公平锁 */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /** 公平锁尝试获取锁 */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

```

#### 属性

```java
    private static final long serialVersionUID = 7373984872572414699L;

    /** 内部类对象 */
    private final Sync sync;
```

#### 构造方法

```java
    /** 创建一个非公平锁的ReentrantLock实例 */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /** 根据参数创建指定Sync对象 */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

#### `ReentrantLock#lockLock`

```java
    //ReentrantLock的lock方法，会调用Sync的lock方法，根据公平、非公平又有不同的实现
	public void lock() {
        sync.lock();
    }
	
	//公平的实现
    final void lock() {
        acquire(1);
    }
	//非公平的实现
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
     }
	//所以lock的实现，核心是acquire()方法，acquire是AQS中的方法
	//基本过程：能获得锁就立即获得，否则加入等待队列，被唤醒后检查自己是不是第一个等待的线程，如果是且能获	//得锁，就返回，否则继续等待。这个过程如果发生了中断，lock会记录中断标志位，但不会提前返回或抛异常

	//AQS中acquire()方法的实现
    public final void acquire(int arg) {
        //tryAcquire尝试去获取锁，如果返回true,即获得了锁,后面就不再执行,如果返回false,即获取锁失败
        //addWaiter()将该线程封装为节点加入等待队列的尾部，并标记为独占模式
        //acquireQueued()使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中
        //	断过，则返回true，否则返回false。
        //如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断，selfInterrupt()
        //	方法将中断补上。
        if (!tryAcquire(arg) &&acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
	
	//tryAcquire又分公平、不公平两种实现，表示是否可以得到锁，
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

	//tryAcquire的公平实现，获得锁返回true,否则返回false
    protected final boolean tryAcquire(int acquires) {
        //获得当前线程
        final Thread current = Thread.currentThread();
        //获取锁数量
        int c = getState();
        //目前还没锁
        if (c == 0) {
            //判断“当前线程”是不是CLH队列中的第一个线程线程，若是的话，则获取该锁，设置锁的状态，并切设			 //置锁的拥有者为“当前线程”。
            if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //被锁了，但是是当前线程获得的锁
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        //获取锁失败
        return false;
    }

	//tryAcquire的不公平实现,获得锁返回true,否则返回false
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //这里只需要cas获取锁即可
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;-
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

	//将该线程封装为节点加入等待队列的尾部
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //有前驱节点，把当前节点放在队列末尾并返回当前节点
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //没有前驱节点，即队列为空，则执行enq方法，把node节点放入等待队列中
        enq(node);
        return node;
    }
	
	//如果等待队列为空,enq方法不停的循环初始化同步队列,直至将node节点入队返回
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

	//使在等待队列中的线程节点获取锁，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否	  //则返回false。
    final boolean acquireQueued(final Node node, int arg) {
        //是否成功拿到锁
        boolean failed = true;
        try {
            //是否被中断
            boolean interrupted = false;
            //自旋
            for (;;) {
                //拿到前驱节点
                final Node p = node.predecessor();
                //如果前驱节点是头节点就让p尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    //p获取到了锁，那么node就是头节点
                    setHead(node);
                    //让p出队
                    p.next = null; // help GC
                    //成功拿到锁，更新标志
                    failed = false;
                    //返回等待过程中是否被中断
                    return interrupted; 
                }
                //如果p不是head或者是head但没获取到锁
                if (shouldParkAfterFailedAcquire(p,node)&&parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }	
	
	//作用：节点node能不能去休息，取决于它的前驱节点p的装态是不是Node.SIGNAL，即会不会通知它
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //获取pred的waitStatus
        int ws = pred.waitStatus;
        //Node.SIGNAL：已告诉pred释放锁后通知自己，返回true
        if (ws == Node.SIGNAL) 
            return true;
        //ws>0对应Node.CANCELLED，即pred被取消了,将node前移
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        //其他情况，继续等待
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

	//执行到这里，表示shouldParkAfterFailedAcquire返回true，即可以休息了，就调用LockSupport.park	   //让它去休息，进入waitting转态。此时有两种途径可唤醒,(unpark和interrupt)
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```



#### `AQS`的静态内部类-`Node`

```java
    //等待队列中的节点
	static final class Node {
        
        /** SHARED标识 */
        static final Node SHARED = new Node();
        
        /** EXCLUSIVE标识 */
        static final Node EXCLUSIVE = null;

        /** 表明该线程被取消了 */
        static final int CANCELLED =  1;
        
        /** 表示可以被前一个节点唤醒 */
        static final int SIGNAL    = -1;
        
        /** 表明正在等待某个条件 */
        static final int CONDITION = -2;
        
        /** 表示下一个共享模式的节点应该无条件的传播下去 */
        static final int PROPAGATE = -3;

        /**
         * waitStatus不同值含义：
         *   SIGNAL:当前节点的后继节点已经 (或即将)被阻塞(通过park),  所以当当前节点释放或则被
         * 取消时候，一定要unpark它的后继节点。为了避免竞争，获取方法一定要首先设置node为signal，然后
         * 再次重新调用获取方法，如果失败，则阻塞。
         *
         *   CANCELLED:  当前节点由于超时或者被中断而被取消。一旦节点被取消后，那么它的状态值不在会被
         * 改变，且当前节点的线程不会再次被阻塞。
         *
         *   CONDITION:  该节点的线程处于等待条件状态,不会被当作是同步队列上的节点,直到被唤醒
         * (signal),设置其值为0,重新进入阻塞状态.
         *
         *   PROPAGATE: 共享模式下的释放操作应该被传播到其他节点。该状态值在doReleaseShared方法中被
         * 设置的。
         *
         *   0:以上都不是
         *
         * 该状态值为了简便使用，所以使用了数值类型。非负数值意味着该节点不需要被唤醒。所以，大多数代码中
         * 不需要检查该状态值的确定值。
         * 
         * 一个正常的Node，它的waitStatus初始化值是0。如果想要修改这个值，可以使用AQS提供CAS进行修改
         */
        volatile int waitStatus;

        /** 前驱节点 */
        volatile Node prev;

        /** 后继节点 */
        volatile Node next;

        /** 等待锁的线程 */
        volatile Thread thread;

        /**
         * ConditionObject链表的后继节点或者代表共享模式的节点。
         * 因为Condition队列只能在独占模式下被能被访问,我们只需要简单的使用链表队列来链接正在等待条件的		   * 节点。然后它们会被转移到同步队列（AQS队列）再次重新获取。由于条件队列只能在独占模式下使用，所		  * 以我们要表示共享模式的节点的话只要使用特殊值SHARED来标明即可。
		 */
        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```



#### `ReentrantLock#unLock`

```java
    /** ReentrantLock释放锁 */
	public void unlock() {
        //调用AQS中的release方法
        sync.release(1);
    }

	//AQS中释放锁的方法
    public final boolean release(int arg) {
        //tryRelease尝试释放锁，如果锁都释放了返回true
        if (tryRelease(arg)) {
            //找到头节点(即等待队列中的下个节点)
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

	//AQS中尝试释放锁，会调用下面ReentrantLock中的实现tryRelease
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }

	//ReentrantLock中实现的tryRelease方法
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

	
    private void unparkSuccessor(Node node) {
       
        int ws = node.waitStatus;
        //如果ws<0，就用cas设为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        //继续找到下一个要唤醒的节点s
        Node s = node.next;
        //s为空或被取消
        if (s == null || s.waitStatus > 0) {
            s = null;
            //从后往前遍历，即找到离s最近的一个没有被取消的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

#### condition

```java
    //java.util.concurrent.locks.Lock接口中定义newCondition方法
    Condition newCondition();

	//ReentrantLock中的实现是调用抽象静态内部类Sync中的newCondition方法
    public Condition newCondition() {
        return sync.newCondition();
    }

	//Sync中返回一个ConditionObject对象
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

	//ConditionObject在AQS中作为成员内部类实现了java.util.concurrent.locks.Condition接口
	public interface Condition {
		
        void await() throws InterruptedException;
		//不会由于中断结束，但当它返回时，如果等待过程发生了中断，中断标志位会被设置
        void awaitUninterruptibly();

        long awaitNanos(long nanosTimeout) throws InterruptedException;

        boolean await(long time, TimeUnit unit) throws InterruptedException;

        boolean awaitUntil(Date deadline) throws InterruptedException;

        void signal();

        void signalAll();
	}
	
	//ConditionObject中对await方法的实现
    public final void await() throws InterruptedException {
        //如果中断标志位已被设置，直接抛异常
        if (Thread.interrupted())
            throw new InterruptedException();
        //1.为当前线程创建节点，加入条件等待队列    
        Node node = addConditionWaiter();
        //2.释放持有的锁
        //获取锁的数量
        int savedState = fullyRelease(node);
        //中断模式，0：不中断，-1：抛异常中断，1：不抛异常，只设标志
        int interruptMode = 0;
        //3.放弃CPU,进行等待
        //isOnSyncQueue为true，表示节点已被其他线程从等待队列移到外部的锁等待队列，等待的条件满足
        while (!isOnSyncQueue(node)) {
            //唤醒
            LockSupport.park(this);
            //检查中断
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        //4.重新获取锁
        //会一直尝试获取锁，获取失败会挂起，直到获取，返回true表示这个过程中被中断过
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        //5.处理中断，抛异常或设置中断标志位
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }

	//ConditionObject中对signal方法的实现
    public final void signal() {
        //验证当前线程持有锁
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        //调用doSignal唤醒等待队列中第一个线程
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);  
    }
```

- 调用`await`、`signal/signalAll`、`wait`、`notify/notifyAll`方法前需要获取锁，如果没有锁，会抛出`IllegalMonitorStateException`。
- `await`在进入等待队列后，会释放锁、CPU，当其他线程将他唤醒后，或者等待超时后，或发生中断异常后，它都需要重新获取锁，获取锁后，才会从`await`方法中退出。
- `notify/notifyAll`是`Object`中定义的方法，`Condition`对象也有。
- **显式锁与显式条件配合使用，即`await/signal/signalAll`与`Lock`配合使用，`wait/notify/notifyAll`与`synchronized`配合使用。**
- **await/signal与ReentrantLock、Condition配合使用粒度更细**

 