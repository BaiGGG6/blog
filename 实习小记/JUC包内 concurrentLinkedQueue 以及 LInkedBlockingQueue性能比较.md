# ConcurrentLinkedQueue & LinkedBlockingQueue & CopyOnWriteArrayList & 的性能对比 以及部分JUC包八股

## 前言
此篇的由来，其实是在实习过程中，遇到了个需求，涉及到多线程并发获取对应的数据结果集
哪，在cr过程中，发现获取结果集的集合，是普通list，哪可能会造成数据丢失，index超出问题，
与leader进行讨论，并衍生到一些juc包下的并发类，以及具体几个的性能对比

## 术语解释
1. CR: codeReview代码评审
2. CAS：compare and swap比较再交换，一种无锁实现多线程原子操作的方法

## 具体测试
测试代码 tips：性能也和运行电脑cpu的核数性能强度有关，代码不是很优雅，存在一定误差
```java
public class ThreadTest {
    private static int CAPACITY = 5000;
    private static int THREAD_COUNT = 10;
    private static int YURE_COUNT = 10;
    private static int RUN_COUNT = 10;
    public static void main(String[] args) throws InterruptedException {

        ConcurrentLinkedQueue<Object> concurrentLinkedQueue = new ConcurrentLinkedQueue<>();
        LinkedBlockingQueue<Object> linkedBlockingQueue = new LinkedBlockingQueue<>();
        // 初始化list大小，减去扩容的时间
        List<Object> list = Collections.synchronizedList(new ArrayList<>(THREAD_COUNT * CAPACITY));
        CopyOnWriteArrayList<Object> copyOnWriteArrayList = new CopyOnWriteArrayList<>();
        for(int i = 0; i < YURE_COUNT; i++){
            test("concurrentLinkedQueue", null, concurrentLinkedQueue);
            test("linkedBlockingQueue", null, linkedBlockingQueue);
            test("synchronizedList", list, null);
            test("copyOnWriteArrayList", copyOnWriteArrayList, null);
            System.out.println();
        }
        long concurrentLinkedQueueReadTime = 0, concurrentLinkedQueueGetTime = 0, concurrentLinkedQueueAllTime = 0;
        long linkedBlockingQueueReadTime = 0, linkedBlockingQueueGetTime = 0, linkedBlockingQueueAllTime = 0;
        long synchronizedListReadTime = 0, synchronizedListGetTime = 0, synchronizedListAllTime = 0;
        long copyOnWriteArrayListReadTime = 0, copyOnWriteArrayListGetTime = 0, copyOnWriteArrayListAllTime = 0;
        for(int i = 0; i < RUN_COUNT; i++){
            long[] concurrentLinkedQueueTimeArray = test("concurrentLinkedQueue", null, concurrentLinkedQueue);
            long[] linkedBlockingQueueTimeArray = test("linkedBlockingQueue", null, linkedBlockingQueue);
            long[] synchronizedListTimeArray = test("synchronizedList", list, null);
            long[] copyOnWriteArrayListTimeArray = test("copyOnWriteArrayList", copyOnWriteArrayList, null);
            concurrentLinkedQueueReadTime += concurrentLinkedQueueTimeArray[0];
            concurrentLinkedQueueGetTime += concurrentLinkedQueueTimeArray[1];
            concurrentLinkedQueueAllTime += concurrentLinkedQueueTimeArray[2];
            linkedBlockingQueueReadTime += linkedBlockingQueueTimeArray[0];
            linkedBlockingQueueGetTime += linkedBlockingQueueTimeArray[1];
            linkedBlockingQueueAllTime += linkedBlockingQueueTimeArray[2];
            synchronizedListReadTime += synchronizedListTimeArray[0];
            synchronizedListGetTime += synchronizedListTimeArray[1];
            synchronizedListAllTime += synchronizedListTimeArray[2];
            copyOnWriteArrayListReadTime += copyOnWriteArrayListTimeArray[0];
            copyOnWriteArrayListGetTime += copyOnWriteArrayListTimeArray[1];
            copyOnWriteArrayListAllTime += copyOnWriteArrayListTimeArray[2];
            System.out.println();
        }
        System.out.println("----------------总结-------------------");
        System.out.println("concurrentLinkedQueue Read平均耗时: " + concurrentLinkedQueueReadTime / RUN_COUNT + " ms, Get平均耗时: " + concurrentLinkedQueueGetTime / RUN_COUNT + " ms, AllTime平均耗时:" + concurrentLinkedQueueAllTime / RUN_COUNT + " ms");
        System.out.println("linkedBlockingQueue Read平均耗时: " + linkedBlockingQueueReadTime / RUN_COUNT + " ms, Get平均耗时: " + linkedBlockingQueueGetTime / RUN_COUNT + " ms, AllTime平均耗时:" + linkedBlockingQueueAllTime / RUN_COUNT + " ms");
        System.out.println("synchronizedList Read平均耗时: " + synchronizedListReadTime / RUN_COUNT + " ms, Get平均耗时: " + synchronizedListGetTime / RUN_COUNT + " ms, AllTime平均耗时:" + synchronizedListAllTime / RUN_COUNT + " ms");
        System.out.println("copyOnWriteArrayList Read平均耗时: " + copyOnWriteArrayListReadTime / RUN_COUNT + " ms, Get平均耗时: " + copyOnWriteArrayListGetTime / RUN_COUNT + " ms, AllTime平均耗时:" + copyOnWriteArrayListAllTime / RUN_COUNT + " ms");
    }

    public static long[] test(String name, List list, Queue queue) throws InterruptedException {
        long start = System.currentTimeMillis();
        CountDownLatch readLatch = new CountDownLatch(THREAD_COUNT);
        Thread[] threads = new Thread[THREAD_COUNT];
        for (int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < CAPACITY; j++) {
                    if(list != null){
                        list.add(j);
                    }
                    if(queue != null){
                        queue.add(j);
                    }
                }
                readLatch.countDown();
            });
            threads[i].start();
        }
        // 等待全部线程都初始化完成
        readLatch.await();
        long mid = System.currentTimeMillis();
        CountDownLatch getLatch = new CountDownLatch(THREAD_COUNT);
        for (int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < CAPACITY; j++) {
                    if(list != null){
                        list.remove(0);
                    }
                    if(queue != null){
                        queue.poll();
                    }
                }
                getLatch.countDown();
            });
            threads[i].start();
        }
        getLatch.await();
        long end = System.currentTimeMillis();
        System.out.println(name + " ,readTime: " + (mid - start) + " ms, getTime: " + (end - mid) + " ms, allTime: " + (end - start) + " ms");
        return new long[]{mid - start, end - mid, end - start};
    }
}
```
#### 测试结果集

- 第一次测试，数量10000 * 10线程进行写入和读取
![img.png](../imge/实习小记/JUC包内%20concurrentLinkedQueue%20以及%20LInkedBlockingQueue性能比较/img.png)
会发现其中的copyOnWriteArrayList的非常慢，所以我们在后面的测试将其去除


- 第二次测试, 数量100000 * 10线程进行写入和读取
![img_1.png](../imge/实习小记/JUC包内%20concurrentLinkedQueue%20以及%20LInkedBlockingQueue性能比较/img_1.png)
会发现我们的synchronizedList在get就是操作的时候非常的慢


- 第三次测试，数量500000 * 10线程进行写入和读取
![img_2.png](../imge/实习小记/JUC包内%20concurrentLinkedQueue%20以及%20LInkedBlockingQueue性能比较/img_2.png)

- 第四次测试, 数量2000 * 1000线程进行写入和读取
![img_3.png](../imge/实习小记/JUC包内%20concurrentLinkedQueue%20以及%20LInkedBlockingQueue性能比较/img_3.png)

- 第五次测试，数量10 * 10000线程进行写入和读取
![img_4.png](../imge/实习小记/JUC包内%20concurrentLinkedQueue%20以及%20LInkedBlockingQueue性能比较/img_4.png)

- 第六次测试，数量10 * 200000线程进行写入和读取
![img_5.png](../imge/实习小记/JUC包内%20concurrentLinkedQueue%20以及%20LInkedBlockingQueue性能比较/img_5.png)

## 底层实现
1. `Collections.synchronizedList(list);`
底层实现：
- 可以发现其底层是在list外部包了一层，对所有list操作都加了Synchronized锁!

总而言之，其底层就是通过Synchronized锁保证原子性操作
```java 
// 对部分代码进行了省略
static class SynchronizedList<E>
        extends SynchronizedCollection<E>
        implements List<E> {
        private static final long serialVersionUID = -7754090372962971524L;

        final List<E> list;

        SynchronizedList(List<E> list) {
            super(list);
            this.list = list;
        }
        SynchronizedList(List<E> list, Object mutex) {
            super(list, mutex);
            this.list = list;
        }

        public boolean equals(Object o) {
            if (this == o)
                return true;
            synchronized (mutex) {return list.equals(o);}
        }
        public int hashCode() {
            synchronized (mutex) {return list.hashCode();}
        }

        public E get(int index) {
            synchronized (mutex) {return list.get(index);}
        }
        
    }
```

2. `ConcurrentLinkedQueue` 并发的链表队列，底层实现:
- 其Node节点对象其中使用casItem和casNext，通过cas无锁的方式保证操作的原子性
- casItem底层是通过UNSAFE类中的方法进行实现，UNSAFE的方法底层又是Native方法，是c++实现的代码

总的而言，ConcurrentLinkedQueue是通过CAS的方式保证原子性的
```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {
    private static final long serialVersionUID = 196745693267521676L;
    // 部分源码
    private static class Node<E> {
        volatile E item;
        volatile Node<E> next;
        
        Node(E item) {
            UNSAFE.putObject(this, itemOffset, item);
        }

        boolean casItem(E cmp, E val) {
            return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        void lazySetNext(Node<E> val) {
            UNSAFE.putOrderedObject(this, nextOffset, val);
        }

        boolean casNext(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                itemOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
    public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);
        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                if (p.casNext(null, newNode)) {
                    if (p != t) // hop two nodes at a time
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
            }
            else if (p == q)
                p = (t != (t = tail)) ? t : head;
            else
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
}
```

3. LinkedBlockingQueue 链表阻塞队列, 底层实现:
- 基于ReentrantLock实现， 由takeLock和putLock两把锁，分别锁住队头和队尾，也就是他是2把锁进行操作，队头执行不影响队尾出队
- 使用Condition是实现不同线程之间的等待和唤醒，以及take这样的阻塞等待方法

总而言之，LinkedBlockingQueue是基于2把ReentrantLock实现的原子性
```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    private static final long serialVersionUID = -6903933977591709194L;
    // 部分源码
    static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) {
            item = x;
        }
    }

    private final int capacity;

    private final AtomicInteger count = new AtomicInteger();

    transient Node<E> head;

    private transient Node<E> last;

    private final ReentrantLock takeLock = new ReentrantLock();

    private final Condition notEmpty = takeLock.newCondition();

    private final ReentrantLock putLock = new ReentrantLock();

    private final Condition notFull = putLock.newCondition();

    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }

    public boolean offer(E e, long timeout, TimeUnit unit)
            throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
}
```
4. `CopyOnWriteArrayList` 写时复制集合，底层实现：
- 主要是使用了ReentrantLock保证原子性
- 在过程中使用了Arrays.copyOf()以及System.arraycopy()方法对数组进行拷贝
总而言之，这个CopyOnWriteArrayList是通过ReentrantLock实现的
```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    private static final long serialVersionUID = 8673264195747942595L;

    final transient ReentrantLock lock = new ReentrantLock();

    private transient volatile Object[] array;
    
    final Object[] getArray() {
        return array;
    }
    
    final void setArray(Object[] a) {
        array = a;
    }
    
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }
    
    
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                Object[] newElements = new Object[len - 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                setArray(newElements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
}
```


## 总结
tips: 结果不一定准，仅供参考，场景不同所选择的队列也不同！

通过测试结果以及配合源码实现可以得出：
1. copyOnWriteArrayList是最慢的操作，其底层每次操作都会涉及到数组的拷贝
2. 其次是synchronizedList，其是通过直接在外层加锁添加速度不错，但操作速度过慢
3. concurrentLinkedQueue是基于CAS实现的，性能不俗
4. LinkedBlockingQueue是基于2把lock锁实现，队头队尾不影响，性能不俗

## 拓展
- 对于concurrentLinkedQueue和LinkedBlockingQueue这2个队列，一个是CAS一个是lock，
就类似于一个是乐观锁的思想，一个是悲观锁的思想
- 对于JUC包下其实是由concurrent，copyOnWrite，Blocking，三大实现组成的并发集合，核心原理就是上面实现方式的区别
- 对于copyOnWriteArrayList这类比较重的队列，其实他是保证了我们原来的数组不会被修改，保证了我们fail-safe，像其他未进行拷贝的，他会导致我们的fail-fast
- fail-fast就是当我们用迭代器进行遍历时，容器内值进行了修改，就会触发ConcurrentModificationException异常
- fail-safe就是当我们遍历时，容器值进行修改，但是还是会按修改前的值进行遍历，不会造成ConcurrentModificationException异常
