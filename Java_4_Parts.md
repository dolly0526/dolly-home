# 集合 #
2019/9/19 22:20:52 

## List ##
### ArrayList ###
1. `ArrayList list = new ArrayList();`  
jdk7: 底层创建了长度为**10**的Object[]数组elementData  
jdk8: 底层Object[]数组elementData初始化为**{}**, 并没有创建长度为10的数组(代码改了但注释没改)
 ```
	/** Constructs an empty list with an initial capacity of ten. */
	public ArrayList() {
		this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
	}
 ```
2. `list.add(123);`  
jdk7: `elementData[0] = new Integer(123);`  
jdk8: 第一次调用add()时, 底层才创建了长度**10**的数组, 并将数据123添加到elementData[0]  
 ```
	/**
	 * Appends the specified element to the end of this list.
	 *
	 * @param e element to be appended to this list
	 * @return <tt>true</tt> (as specified by {@link Collection#add})
	 */
	public boolean add(E e) {
		ensureCapacityInternal(size + 1);  // Increments modCount!!
    	elementData[size++] = e;
    	return true;
	}

	private void ensureCapacityInternal(int minCapacity) {
		ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
	}

	private static int calculateCapacity(Object[] elementData, int minCapacity) {
    	if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
    	    return Math.max(DEFAULT_CAPACITY, minCapacity);
    	}
    	return minCapacity;
	}
 ```
3. `... list.add(11);`  
 - 先由**ensureExplicitCapacity**方法判断是否需要扩容; 如果此次的添加导致底层elementData数组容量不够, 则通过**grow**方法扩容
 ```
	private void ensureExplicitCapacity(int minCapacity) {
    	modCount++;

    	// overflow-conscious code
    	if (minCapacity - elementData.length > 0)
        	grow(minCapacity);
	}
 ```
 - 默认情况下, 扩容为原来的容量的**1.5倍**, 同时需要将原有数组中的数据复制到新的数组中
 ```
	/**
	 * Increases the capacity to ensure that it can hold at least the
	 * number of elements specified by the minimum capacity argument.
	 *
	 * @param minCapacity the desired minimum capacity
	 */
	private void grow(int minCapacity) {
	    // overflow-conscious code
	    int oldCapacity = elementData.length;
	    int newCapacity = oldCapacity + (oldCapacity >> 1);
	    if (newCapacity - minCapacity < 0)
	        newCapacity = minCapacity;
	    if (newCapacity - MAX_ARRAY_SIZE > 0)
	        newCapacity = hugeCapacity(minCapacity);
	    // minCapacity is usually close to size, so this is a win:
	    elementData = Arrays.copyOf(elementData, newCapacity);
	}
 ```
 - **建议**开发中使用带参的构造器: `ArrayList list = new ArrayList(int capacity);`
4. jdk7中的ArrayList的对象的创建类似于单例的**饿汉式**; 而jdk8中的ArrayList的对象的创建, 类似于单例的**懒汉式**, 延迟了数组的创建, 节省内存

### Vector ###
1. List接口的古老实现类; **线程安全, 但效率低**; 底层和ArrayList一样, 使用Object[] elementData存储
2. `Vector vector = new Vector();`
jdk7和jdk8中通过Vector()构造器创建对象时, 底层都创建了长度为**10**的数组
3. 在扩容方面, 默认扩容为原来的数组长度的**2倍**

### LinkedList ###
1. `LinkedList list = new LinkedList();`  
 - 内部声明了Node类型的first和last属性, 默认值为null
 ```
	transient Node<E> first;
	transient Node<E> last;
 ```
 - 其中, Node定义如下, 体现了LinkedList的双向链表的说法
 ```
	private static class Node<E> {
	    E item;
	    Node<E> next;
	    Node<E> prev;

	    Node(Node<E> prev, E element, Node<E> next) {
	        this.item = element;
	        this.next = next;
	        this.prev = prev;
	    }
	}
 ```
2. `list.add(123);` 创建了Node对象, 将123封装到该对象中
 ```
    /** Links e as last element. */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
 ```

## Map ##
### HashMap ###
0. 参考资料:  
[Java 8系列之重新认识HashMap](https://zhuanlan.zhihu.com/p/21673805)
1. jdk7中的HashMap  
 - `HashMap map = new HashMap()`:在实例化以后，底层创建了长度是**16**的一维数组Entry[] table。  
 - `map.put(key1,value1)`:  
首先，调用key1所在类的hashCode()计算key1哈希值，此哈希值经过某种算法计算以后，得到在Entry数组中的存放位置。  
如果此位置上的数据为空，此时的key1-value1添加成功。 ----情况1  
如果此位置上的数据不为空，(意味着此位置上存在一个或多个数据(以链表形式存在)),比较key1和已经存在的一个或多个数据的哈希值：  
如果key1的哈希值与已经存在的数据的哈希值都不相同，此时key1-value1添加成功。----情况2  
如果key1的哈希值和已经存在的某一个数据(key2-value2)的哈希值相同，继续比较：调用key1所在类的equals(key2)方法，比较：
如果equals()返回false:此时key1-value1添加成功。----情况3  
如果equals()返回true:使用value1替换value2。
 ```
    /**
     * Implements Map.put and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
 ```
 - 补充：关于情况2和情况3：此时key1-value1和原来的数据以链表的方式存储。  
 - 在不断的添加过程中，会涉及到扩容问题，当超出临界值(且要存放的位置非空)时，扩容。默认的扩容方式：扩容为原来容量的**2倍**，并将原有的数据复制过来。
2. jdk8中的HashMap  
 - new HashMap(): 底层没有创建一个长度为16的数组
 ```
    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
 ```
 - jdk 8底层的数组是：Node[], 而非Entry[]
 ```
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
		...
	}
 ```
 - 首次调用put()方法时，底层创建长度为**16**的数组
 ```
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
		...
	}
 ```
 - jdk7底层结构只有：数组+链表；jdk8中底层结构：数组+链表+红黑树。  
 - 形成链表时，七上八下（jdk7:新的元素指向旧的元素。jdk8：旧的元素指向新的元素）
 ```
    if ((e = p.next) == null) {
        p.next = newNode(hash, key, value, null);
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        break;
    }
 ```
 - 当数组的某一个索引位置上的元素以链表形式存在的数据个数 > 8 且当前数组的长度 > 64时，此时此索引位置上的所数据改为使用红黑树存储。
 ```
    /**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
 ```
3. 一些常量:  
DEFAULT_INITIAL_CAPACITY: HashMap的默认容量，16  
DEFAULT_LOAD_FACTOR: 默认填充因子, 0.75(越小则链表越少)  
threshold：扩容的临界值(不会等到满才扩容, 因为不一定会满)，= 容量*填充因子：16 * 0.75 => 12  
TREEIFY_THRESHOLD：Bucket中链表长度大于该默认值，转化为红黑树: 8  
MIN_TREEIFY_CAPACITY：桶中的Node被树化时最小的hash表容量: 64  

### LinkedHashMap ###
调用HashMap的putVal方法, 重写了newNode方法
 ```
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

    /**
     * HashMap.Node subclass for normal LinkedHashMap entries.
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
		//before和after用于记录添加元素的顺序
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
 ```

### HashSet ###
底层还是用HashMap实现
 ```
    /**
     * Adds the specified element to this set if it is not already present.
     * More formally, adds the specified element <tt>e</tt> to this set if
     * this set contains no element <tt>e2</tt> such that
     * <tt>(e==null&nbsp;?&nbsp;e2==null&nbsp;:&nbsp;e.equals(e2))</tt>.
     * If this set already contains the element, the call leaves the set
     * unchanged and returns <tt>false</tt>.
     *
     * @param e element to be added to this set
     * @return <tt>true</tt> if this set did not already contain the specified
     * element
     */
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
 ```


# 基础 #
2019/8/5 17:05:52 


# 并发 #
2019/10/3 14:20:01 

## 基础 ##
### 进程/线程 ###
进程: 每个任务(例如.exe)会起一个进程  
线程: 轻量级的进程, 每个进程可以起多个线程

**注意:**  
1. 线程的启动不是在start()方法后立刻执行, 由底层CPU调度  
2. 并发编程套路:  
 - 在高内聚低耦合的前提下, 线程操作资源类  
 - 判断/干活/通知  
 - 防止虚假唤醒

### 并发/并行 ###
并发: 多个线程抢同一份资源, 偏重于多个任务交替执行  
并行: 多件事情同时发生, 真正的同时执行

## JUC ##
### CopyOnWriteArrayList ###
1. ArrayList是线程不安全的
 ```
	/**
	 * 1.问题场景
	 *   30个线程并发读写同一个ArrayList
	 * 2.故障现象
	 *   java.util.ConcurrentModificationException
	 * 3.解决方法
	 *   1) new Vector<>() --> 过时
	 *   2) Collections.synchronizedList(new ArrayList<>()) --> 低效
	 * 4.优化建议(同样的错误不犯第2次)
	 *   读写分离, new CopyOnWriteArrayList<>()
	 */
    public static void main(String[] args) {
        List<String> list = new CopyOnWriteArrayList<>();
        for (int i = 0; i < 30; i++) {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0, 8));
                System.out.println(list);
            }, String.valueOf(i)).start();
        } //java.util.ConcurrentModificationException
    }
 ```
2. CopyOnWriteArrayList底层实现类也是Object[]数组; jdk8中调用空参构造器时, 会初始化一个空数组
 ```
	/** The array, accessed only via getArray/setArray. */
	private transient volatile Object[] array;

	/** Creates an empty list. */
	public CopyOnWriteArrayList() {
    	setArray(new Object[0]);
	}
 ```
3. **写时复制**对add方法加锁, 做到读写分离, 同时保证并发性和数据一致性
 ```
	/**
	 * Appends the specified element to the end of this list.
	 *
	 * @param e element to be appended to this list
	 * @return {@code true} (as specified by {@link Collection#add})
	 */
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
 ```

### CopyOnWriteArraySet ###
1. HashSet也是线程不安全的, 报错和ArrayList相同, 解决办法类似
2. CopyOnWriteArraySet底层是CopyOnWriteArrayList
 ```
	private final CopyOnWriteArrayList<E> al;

	/** Creates an empty set. */
	public CopyOnWriteArraySet() {
	    al = new CopyOnWriteArrayList<E>();
	}
 ```
3. 添加时调用**addIfAbsent**方法, 先判断是否存在, 不存在则添加
 ```
	/**
	 * Appends the element, if not present.
	 *
	 * @param e element to be added to this list, if absent
	 * @return {@code true} if the element was added
	 */
	public boolean addIfAbsent(E e) {
	    Object[] snapshot = getArray();
	    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
 	       addIfAbsent(e, snapshot);
	}

	/**
	 * A version of addIfAbsent using the strong hint that given
	 * recent snapshot does not contain e.
	 */
	private boolean addIfAbsent(E e, Object[] snapshot) {
    	final ReentrantLock lock = this.lock;
    	lock.lock();
    	try {
        	Object[] current = getArray();
        	int len = current.length;
        	if (snapshot != current) {
            	// Optimize for lost race to another addXXX operation
            	int common = Math.min(snapshot.length, len);
            	for (int i = 0; i < common; i++)
                	if (current[i] != snapshot[i] && eq(e, current[i]))
                    	return false;
            	if (indexOf(e, current, common, len) >= 0)
                    	return false;
        	}
        	Object[] newElements = Arrays.copyOf(current, len + 1);
        	newElements[len] = e;
        	setArray(newElements);
			return true;
		} finally {
        	lock.unlock();
    	}
	}
 ```

### ConcurrentHashMap ###
1. 并发版HashMap, 报错相同, 解决办法类似
2. 待补充

## 三个线程循环打印 ##
```
public class Condition06 {
    public static void main(String[] args) {

        ShareData data = new ShareData();
        Lock lock = data.getLock();
        Condition c1 = lock.newCondition();
        Condition c2 = lock.newCondition();
        Condition c3 = lock.newCondition();

        new Thread(() -> {
            for (int i = 0; i < 5; i++) data.print(c1, c2, 0, 1, 5);
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 5; i++) data.print(c2, c3, 1, 2, 10);
        }, "B").start();
        new Thread(() -> {
            for (int i = 0; i < 5; i++) data.print(c3, c1, 2, 0, 15);
        }, "C").start();
    }
}

class ShareData {
    //A:1, B:2, C:3
    private int num = 0;
    @Getter
    private Lock lock = new ReentrantLock();

    /**
     * @param c1 该线程绑定的对象
     * @param c2 下一个线程绑定的对象
     * @param p  控制开始的标记
     * @param q  控制结束的标记
     * @param n  打印次数
     */
    public void print(Condition c1, Condition c2, int p, int q, int n) {
        lock.lock();
        try {
            //1.判断
            while (num != p) c1.await();
            //2.干活
            for (int i = 0; i < n; i++)
                System.out.println(Thread.currentThread().getName() + "\t" + i);
            //3.通知(如何通知第2个)
            num = q;
            c2.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

## synchronized ##
0. 参考资料:  
 - [Java并发编程：Synchronized及其实现原理](https://www.cnblogs.com/paddix/p/5367116.html)  
 - [synchronized锁定的到底是什么？](https://www.zhihu.com/question/57794716/answer/606126905)
1. 一个对象里面如果有多个synchronized方法，某一个时刻内，只要一个线程去调用其中的一个synchronized方法了，其它的线程都只能等待；换句话说，某一个时刻内，只能有唯一一个线程去访问这些synchronized方法
2. 锁的是当前对象this，被锁定后，其它的线程都不能进入到当前对象的其它的synchronized方法
3. 加个普通方法后发现和同步锁无关
4. 换成两个对象后，不是同一把锁了，情况立刻变化。
5. synchronized实现同步的基础：Java中的每一个对象都可以作为锁。具体表现为以下3种形式:  
 - 对于普通同步方法，锁是**当前实例对象**，锁的是当前对象this  
 - 对于同步方法块，锁是**Synchonized括号里配置的对象**  
 - 对于静态同步方法，锁是**当前类的Class对象**
6. 当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。也就是说如果一个实例对象的非静态同步方法获取锁后，该实例对象的其他非静态同步方法必须等待获取锁的方法释放锁后才能获取锁，可是别的实例对象的非静态同步方法因为跟该实例对象的非静态同步方法用的是不同的锁，所以毋须等待该实例对象已获取锁的非静态同步方法释放锁就可以获取他们自己的锁。
7. 所有的静态同步方法用的也是同一把锁——类对象Class本身，这两把锁是两个不同的对象，所以静态同步方法与非静态同步方法之间是不会有竞态条件的。但是一旦一个静态同步方法获取锁后，其他的静态同步方法都必须等待该方法释放锁后才能获取锁，而不管是同一个实例对象的静态同步方法之间，还是不同的实例对象的静态同步方法之间，只要它们同一个类的实例对象！


# 虚拟机 #
2019/10/15 14:51:12 

## ClassLoader ##
1. 负责加载.class文件，.class文件在文件开头有特定的文件标示，将.class文件字节码内容加载到内存中，并将这些内容转换成**方法区**中的运行时数据结构。ClassLoader只负责.class文件的加载，至于它是否可以运行，则由**Execution Engine**决定  
![](https://i.imgur.com/oHjv06T.png)
2. 不同的类加载器
![](https://i.imgur.com/eB7Noi3.png)
3. **sun.misc.Launcher**是一个java虚拟机的入口应用
 ```
public class MyObject {
    public static void main(String[] args) {
        System.out.println(new Object().getClass().getClassLoader()); //null
        System.out.println(new MyObject().getClass().getClassLoader()); //sun.misc.Launcher$AppClassLoader
        System.out.println(new MyObject().getClass().getClassLoader().getParent()); //sun.misc.Launcher$ExtClassLoader
        System.out.println(new MyObject().getClass().getClassLoader().getParent().getParent()); //null
    }
}
 ```
4. 双亲委派模型(我爸是李刚, 有事找我爹, 往上捅):  
当一个类收到了类加载请求，他首先不会尝试自己去加载这个类，而是把这个请求委派给父类去完成，每一个层次类加载器都是如此，因此所有的加载请求都应该传送到启动类加载其中，只有当父类加载器反馈自己无法完成这个请求的时候（在它的加载路径下没有找到所需加载的Class），子类加载器才会尝试自己去加载。 
5. 沙箱安全机制: 采用双亲委派的一个好处是比如加载位于**rt.jar**包中的类java.lang.Object，不管是哪个加载器加载这个类，最终都是委托给顶层的启动类加载器进行加载，这样就保证了使用不同的类加载器最终得到的都是同样一个Object对象，防止Java原生的类被污染。 
6. Execution Engine执行引擎负责解释命令，提交操作系统执行。 

## native ##
1. Thread类只能start一次
 ```
public class MyThread {
    public static void main(String[] args) {

        Thread thread = new Thread();
        thread.start();
        thread.start(); //java.lang.IllegalThreadStateException
    }

	start()源码中: 
		 * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
}
 ```
2. 本地接口的作用是融合不同的编程语言为Java所用，它的初衷是融合C/C++程序，Java诞生的时候是C/C++横行的时候，要想立足，必须有调用C/C++程序，于是就在内存中专门开辟了一块区域处理标记为native的代码，它的具体做法是Native Method Stack中登记native方法，在Execution Engine 执行时加载native libraies。**(已经比较少见)**
3. 它的具体做法是Native Method Stack中登记native方法，在Execution Engine执行时加载本地方法库。

## PC寄存器 ##
1. 每个线程都有一个程序计数器，是线程私有的,就是一个**指针**，指向方法区中的方法字节码（用来存储指向下一条指令的地址,也即将要执行的指令代码），由执行引擎读取下一条指令，是一个非常小的内存空间，几乎可以忽略不记。
2. 这块内存区域很小 **(几乎不发生GC)**，它是当前线程所执行的字节码的行号指示器，字节码解释器通过改变这个计数器的值来选取下一条需要执行的字节码指令。
3. 如果执行的是一个native方法，那这个计数器是空的。
4. 用以完成分支、循环、跳转、异常处理、线程恢复等基础功能。**不会发生内存溢出** (OutOfMemory, OOM) 错误。

## 方法区 ##
1. 供各线程共享的运行时内存区域。它存储了每一个类的**结构信息(模版)**，例如运行时常量池（Runtime Constant Pool）、字段和方法数据、构造函数和普通方法的**字节码**内容。上面讲的是规范，在不同虚拟机里头实现是不一样的，最典型的就是永久代(PermGen space)和元空间(Metaspace)。
2. But实例变量存在堆内存中,和方法区无关
3. jdk7: 方法区 f = new 永久带; jdk8: 方法区 f = new 元空间
4. 查看字节码文件: javap -v xx.class

## 栈 ##
1. **栈管运行, 堆管存储**
2. 栈也叫栈内存，主管Java程序的运行，是在线程创建时创建，它的生命期是跟随线程的生命期，线程结束栈内存也就释放，对于栈来说**不存在垃圾回收问题**，只要线程一结束该栈就Over，生命周期和线程一致，是线程私有的。8种基本类型的变量+对象的引用变量+实例方法都是在函数的栈内存中分配。
3. 栈帧中主要保存 3 类数据：  
本地变量（Local Variables）: 输入参数和输出参数以及方法内的变量；  
栈操作（Operand Stack）: 记录出栈、入栈的操作；  
栈帧数据（Frame Data）: 包括类文件、方法等等。
4. 栈运行原理：  
a. 栈中的数据都是以栈帧（Stack Frame）的格式存在，栈帧是一个内存区块，是一个数据集，是一个有关方法(Method)和运行期数据的数据集，当一个方法A被调用时就产生了一个栈帧 F1，并被压入到栈中， 
A方法又调用了 B 方法，于是产生栈帧 F2 也被压入栈，  
B方法又调用了 C 方法，于是产生栈帧 F3 也被压入栈，  
……  
执行完毕后，先弹出F3栈帧，再弹出F2栈帧，再弹出F1栈帧……  
b. 遵循“先进后出”/“后进先出”原则。  
c. 每个方法执行的同时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息，每一个方法从调用直至执行完毕的过程，就对应着一个栈帧在虚拟机中入栈到出栈的过程。栈的大小和具体JVM的实现有关，通常在256K~756K之间,与等于1Mb左右。
![](https://i.imgur.com/zPDQNBS.png)
5. Exception in thread "main" java.lang.**StackOverflowError**  
![](https://i.imgur.com/0oBqD7W.png)
6. 栈+堆+方法区的交互关系  
![](https://i.imgur.com/CGfvHhP.png)

## 堆 ##
### 基础知识 ###
1. 一个JVM实例只存在一个堆内存，堆内存的大小是可以调节的。类加载器读取了类文件后，需要把类、方法、常变量放到堆内存中，保存所有引用类型的真实信息，以方便执行器执行，堆内存**在逻辑上**分为三部分：  
Young Generation Space  	**新生区**  		Young/New  
Tenure generation space  	**养老区**  		Old/Tenure  
Permanent Space  			**永久区**  		Perm  
注: jdk1.8中没有永久区了, 改为了**元空间** (Meta Space)
2. (Java7之前) 一个JVM实例只存在一个堆内存，堆内存的大小是可以调节的。类加载器读取了类文件后，需要把类、方法、常变量放到堆内存中，保存所有引用类型的真实信息，以方便执行器执行。  
![](https://i.imgur.com/s0MfSaI.png)
3. **新生区**是类的诞生、成长、消亡的区域，一个类在这里产生，应用，最后被垃圾回收器收集，结束生命。新生区又分为两部分： 伊甸区（Eden space）和幸存者区（Survivor pace） ，所有的类都是在伊甸区被new出来的。幸存区有两个： 0区（Survivor 0 space）和1区（Survivor 1 space）。当伊甸园的空间用完时，程序又需要创建对象，JVM的垃圾回收器将对伊甸园区进行垃圾回收(Minor GC)，将伊甸园区中的不再被其他对象所引用的对象进行销毁。然后将伊甸园中的剩余对象移动到幸存 0区。若幸存 0区也满了，再对该区进行垃圾回收，然后移动到 1 区。那如果1 区也满了呢？再移动到养老区。若养老区也满了，那么这个时候将产生MajorGC（FullGC），进行养老区的内存清理。若**养老区**执行了**Full GC**之后发现依然无法进行对象的保存，就会产生OOM异常**OutOfMemoryError**。
4. 如果出现java.lang.OutOfMemoryError: Java heap space 异常，说明Java虚拟机的堆内存不够。原因有二：  
（1）Java虚拟机的堆内存设置不够，可以通过参数-Xms、-Xmx来调整。  
（2）代码中创建了大量大对象，并且长时间不能被垃圾收集器收集（存在被引用）。
5. MinorGC的过程 (复制->清空->互换)    
![](https://i.imgur.com/6dAw0GV.png)  
a. Eden、SurvivorFrom复制到 SurvivorTo，年龄+1  
首先，当**Eden区**满的时候会触发第一次GC，把还活着的对象拷贝到SurvivorFrom区，当Eden区再次触发GC的时候会扫描**Eden区和From区域**，对这两个区域进行垃圾回收，经过这次回收后还存活的对象，则直接复制到To区域（如果有对象的年龄已经达到了老年的标准，则赋值到老年代区），同时把这些对象的年龄+1  
b. 清空Eden、SurvivorFrom   
然后，清空Eden和SurvivorFrom中的对象，也即**复制之后有交换，谁空谁是to**  
c. SurvivorTo和SurvivorFrom互换  
最后，SurvivorTo和SurvivorFrom互换，原SurvivorTo成为下一次GC时的SurvivorFrom区。部分对象会在From和To区域中复制来复制去，如此交换15次 (由JVM参数MaxTenuringThreshold决定,这个参数默认是15)，最终如果还是存活，就存入到老年代 (98%的对象是临时对象)
6. 实际而言，方法区（Method Area）和堆一样，是各个线程共享的内存区域，它用于存储虚拟机加载的：类信息+普通常量+静态常量+编译器编译后的代码等等，虽然JVM规范将方法区描述为堆的一个逻辑部分，但它却还有一个别名叫做**Non-Heap(非堆)**，目的就是要和堆分开。
7. 对于HotSpot虚拟机，很多开发者习惯将方法区称之为“永久代(Parmanent Gen)” ，但严格本质上说两者不同，或者说使用永久代来实现方法区而已，永久代是方法区 (相当于是一个接口interface) 的一个实现，jdk1.7的版本中，已经将原本放在永久代的字符串常量池移走。
![](https://i.imgur.com/2V8WCT0.png)
8. **永久存储区**是一个常驻内存区域，用于存放JDK自身所携带的 Class,Interface 的元数据，也就是说它存储的是运行环境必须的类信息，被装载进此区域的数据是不会被垃圾回收器回收掉的，关闭 JVM 才会释放此区域所占用的内存。
9. 关于元空间:  
 - 在Java8中，永久代已经被移除，被一个称为**元空间**的区域所取代。元空间的本质和永久代类似。
 - 元空间与永久代之间最大的区别在于：永久带使用的JVM的堆内存，但是java8以后的元空间并不在虚拟机中而是使用**本机物理内存**。
 - 因此，默认情况下，元空间的大小仅受本地内存限制。类的元数据放入 native memory, 字符串池和类的静态变量放入java堆中，这样可以加载多少类的元数据就不再由MaxPermSize 控制, 而由系统的实际可用空间来控制。

### 参数调优 ###
1. gc模型  
**jdk7:** ![](https://i.imgur.com/X6VxUDR.png)  
**jdk8:** ![](https://i.imgur.com/bOpYvSR.png)
2. 堆基础参数, 如果要手动配置, 一般Xmx和Xms大小**相同**, 防止内存不稳定  
![](https://i.imgur.com/o2Szxp8.png)
3. GC收集日志信息  
![](https://i.imgur.com/9mgweu3.png)  
![](https://i.imgur.com/5PaO26l.png)  
![](https://i.imgur.com/6QYsv5H.png)

## GC算法 ##
### 整体概括 ###
1. GC的**分代收集算法**  
 - 次数上频繁收集Young区  
 - 次数上较少收集Old区  
 - 基本不动元空间
2. JVM在进行GC时，并非每次都对上面三个内存区域一起回收的，大部分时候回收的都是指新生代。因此GC按照回收的区域又分了两种类型，一种是普通GC（minor GC），一种是全局GC（major GC or Full GC）。  
3. Minor GC和Full GC的区别:  
 - 普通GC（minor GC）：只针对**新生代**区域的GC,指发生在新生代的垃圾收集动作，因为大多数Java对象存活率都不高，所以Minor GC非常频繁，一般回收速度也比较快。  
 - 全局GC（major GC or Full GC）：指发生在**老年代**的垃圾收集动作，出现了Major GC，经常会伴随至少一次的Minor GC（但并不是绝对的）。Major GC的速度一般要比Minor GC慢上10倍以上。
4. 显式调用`System.gc()`时，**不一定**会立刻执行GC，而是提醒或告诉虚拟机，希望进行一次垃圾回收，至于什么时候进行回收还是取决于虚拟机，而且也不能保证一定进行回收。

### 算法 ###
#### 引用计数法(Reference-Count) ####
![](https://i.imgur.com/K3pSHlg.png)

#### 复制算法(Copying) ####
1. **年轻代**中使用的是Minor GC，这种GC算法采用的是复制算法(Copying)  
2. 原理  
Minor GC会把Eden中的所有活的对象都移到Survivor区域中，如果Survivor区中放不下，那么剩下的活的对象就被移到Old  generation中，也即一旦收集后，Eden是就变成空的了。  
当对象在 Eden ( 包括一个 Survivor 区域，这里假设是 from 区域 ) 出生后，在经过一次 Minor GC 后，如果对象还存活，并且能够被另外一块 Survivor 区域所容纳( 上面已经假设为 from 区域，这里应为 to 区域，即 to 区域有足够的内存空间来存储 Eden 和 from 区域中存活的对象 )，则使用复制算法将这些仍然还存活的对象复制到另外一块 Survivor 区域 ( 即 to 区域 ) 中，然后清理所使用过的 Eden 以及 Survivor 区域 ( 即 from 区域 )，并且将这些对象的年龄设置为1，以后对象在 Survivor 区每熬过一次 Minor GC，就将对象的年龄 + 1，当对象的年龄达到某个值时 ( 默认是 **15** 岁，通过-XX:MaxTenuringThreshold 来设定参数)，这些对象就会成为老年代。  
-XX:MaxTenuringThreshold — 设置对象在新生代中存活的次数
3. 解释  
 - HotSpot JVM把年轻代分为了三部分：1个Eden区和2个Survivor区（分别叫from和to）。默认比例为**8:1:1**, 一般情况下，新创建的对象都会被分配到Eden区(一些大对象特殊处理),这些对象经过第一次Minor GC后，如果仍然存活，将会被移到Survivor区。对象在Survivor区中每熬过一次Minor GC，年龄就会增加1岁，当它的年龄增加到一定程度时，就会被移动到年老代中。因为年轻代中的对象基本都是朝生夕死的(90%以上)，所以在年轻代的垃圾回收算法使用的是复制算法，复制算法的基本思想就是将内存分为两块，每次只用其中一块，当这一块内存用完，就将还活着的对象复制到另外一块上面。复制算法**不会产生内存碎片**。  
 - 优点: 没碎片, 缺点: 耗空间  
![](https://i.imgur.com/h1xdx7v.png)
 - 在GC开始的时候，对象只会存在于Eden区和名为“From”的Survivor区，Survivor区“To”是空的。紧接着进行GC，Eden区中所有存活的对象都会被复制到“To”，而在“From”区中，仍存活的对象会根据他们的年龄值来决定去向。年龄达到一定值(年龄阈值，可以通过-XX:MaxTenuringThreshold来设置)的对象会被移动到年老代中，没有达到阈值的对象会被复制到“To”区域。经过这次GC后，Eden区和From区已经被清空。这个时候，“From”和“To”会交换他们的角色，也就是新的“To”就是上次GC前的“From”，新的“From”就是上次GC前的“To”。不管怎样，都会保证名为To的Survivor区域是空的。Minor GC会一直重复这样的过程，直到“To”区被填满，“To”区被填满之后，会将所有对象移动到年老代中。  
 - 图解  
![](https://i.imgur.com/ULTbOwF.png)  
 - 因为Eden区对象一般**存活率较低**，一般的，使用两块10%的内存作为空闲和活动区间，而另外80%的内存，则是用来给新建对象分配内存的。一旦发生GC，将10%的from活动区间与另外80%中存活的eden对象转移到10%的to空闲区间，接下来，将之前90%的内存全部释放，以此类推。
4. 劣势  
 - 它**浪费**了一半的内存，这太要命了。 
 - 如果对象的存活率很高，我们可以极端一点，假设是100%存活，那么我们需要将所有对象都复制一遍，并将所有引用地址重置一遍。复制这一工作所花费的时间，在对象存活率达到一定程度时，将会变的不可忽视。 所以从以上描述不难看出，复制算法要想使用，最起码对象的存活率要非常低才行，而且最重要的是，我们必须要克服50%内存的浪费。

#### 标记清除(Mark-Sweep) ####
1. **老年代**一般是由标记清除或者是标记清除与标记整理的混合实现
2. 原理  
 - 图解
![](https://i.imgur.com/42pnbKd.png)  
![](https://i.imgur.com/qILo8oW.png)
 - 用通俗的话解释一下标记清除算法，就是当程序运行期间，若可以使用的内存被耗尽的时候，GC线程就会被触发并将程序暂停，随后将要回收的对象标记一遍，最终统一回收这些对象，完成标记清理工作接下来便让应用程序恢复运行。
 - 主要进行两项工作，第一项则是**标记**，第二项则是**清除**。  
a. 标记：从引用根节点开始标记遍历所有的GC Roots，先标记出要回收的对象。 
b. 清除：遍历整个堆，把标记的对象清除。   
c. 缺点：此算法需要**暂停**整个应用，会产生**内存碎片** 
3. 劣势  
 - 首先，它的缺点就是**效率比较低**（递归与全堆对象遍历），而且在进行GC的时候，需要**停止**应用程序，这会导致用户体验非常差劲
 - 其次，主要的缺点则是这种方式清理出来的空闲内存是**不连续**的，这点不难理解，我们的死亡对象都是随即的出现在内存的各个角落的，现在把它们清除之后，内存的布局自然会乱七八糟。而为了应付这一点，JVM就不得不维持一个内存的空闲列表，这又是一种开销。而且在分配数组对象的时候，寻找连续的内存空间会不太好找。

#### 标记压缩(Mark-Compact) ####
1. **老年代**一般是由标记清除或者是标记清除与标记整理的混合实现
2. 原理  
 - 图解  
![](https://i.imgur.com/Hz3Oujk.png)
 - 在整理压缩阶段，不再对标记的对像做回收，而是通过所有存活对像都向一端移动，然后直接清除边界以外的内存。
 - 可以看到，标记的存活对象将会被整理，按照内存地址依次排列，而未被标记的内存会被清理掉。如此一来，当我们需要给新对象分配内存时，JVM只需要持有一个内存的起始地址即可，这比维护一个空闲列表显然少了许多开销。 
 - 标记/整理算法不仅可以弥补标记/清除算法当中，内存区域分散的缺点，也消除了复制算法当中，内存减半的高额代价
3. 劣势  
标记/整理算法唯一的缺点就是**效率也不高**，不仅要标记所有存活对象，还要整理所有存活对象的引用地址。从效率上来说，标记/整理算法要低于复制算法。
4. 标记清除压缩(Mark-Sweep-Compact)  
![](https://i.imgur.com/CYIgfFf.png)

#### 总结 ####
1. 三种算法比较  
 - 内存效率：复制算法 > 标记清除算法 > 标记整理算法（此处的效率只是简单的对比时间复杂度，实际情况不一定如此）。 
 - 内存整齐度：复制算法 = 标记整理算法 > 标记清除算法。 
 - 内存利用率：标记整理算法 = 标记清除算法 > 复制算法。
2. 可以看出，效率上来说，复制算法是当之无愧的老大，但是却浪费了太多内存，而为了尽量兼顾上面所提到的三个指标，标记/整理算法相对来说更平滑一些，但效率上依然不尽如人意，它比复制算法多了一个标记的阶段，又比标记/清除多了一个整理内存的过程
3. 没有最好的算法，只有最合适的算法。 ==========>  **分代收集算法**。
 - 年轻代 (Young Gen)   
年轻代特点是区域相对老年代较小，对像存活率低。  
这种情况**复制算法**的回收整理，速度是最快的。复制算法的效率只和当前存活对像大小有关，因而很适用于年轻代的回收。而复制算法内存利用率不高的问题，通过hotspot中的两个survivor的设计得到缓解。
 - 老年代 (Tenure Gen)  
老年代的特点是区域较大，对像存活率高。  
这种情况，存在大量存活率高的对像，复制算法明显变得不合适。一般是由**标记清除**或者是**标记清除与标记整理**的混合实现。
4. 老年代一般是由标记清除或者是标记清除与标记整理的混合实现。以hotspot中的**CMS回收器**为例，CMS是基于Mark-Sweep实现的，对于对像的回收效率很高，而对于碎片问题，CMS采用基于Mark-Compact算法的Serial Old回收器做为补偿措施：当内存回收不佳（碎片导致的Concurrent Mode Failure时），将采用Serial Old执行Full GC以达到对老年代内存的整理。  
a. Mark阶段的开销与存活对像的数量成正比，这点上说来，对于老年代，标记清除或者标记整理有一些不符，但可以通过多核/线程利用，对并发、并行的形式提标记效率。  
b. Sweep阶段的开销与所管理区域的大小形正相关，但Sweep“就地处决”的特点，回收的过程没有对像的移动。使其相对其它有对像移动步骤的回收算法，仍然是效率最好的。但是需要解决内存碎片问题。  
c. Compact阶段的开销与存活对像的数据成开比，如上一条所描述，对于大量对像的移动是很大开销的，做为老年代的第一选择并不合适。

#### 面试题 ####
- JVM内存模型以及分区，需要详细到每个区放什么
- 堆里面的分区：Eden，survival from to，老年代，各自的特点。
- GC的三种收集方法：标记清除、标记整理、复制算法的原理与特点，分别用在什么地方
- Minor GC与Full GC分别在什么时候发生


# P6面试题 #
2019/10/26 20:06:53 

## JMM ##
1. 特征: 可见性, 原子性, 有序性
2. 请你谈谈JMM: 多线程访问内存的规范
![](https://i.imgur.com/WuTuylK.png)  
![](https://i.imgur.com/lDT44F6.jpg)

### 可见性 ###
volatile: 是JVM提供的**轻量级**的同步机制
- 保证可见性
- 不保证原子性
- 禁止指令重排

### 原子性 ###
不可分割, 完整性; 也即某个线程正在做某个具体业务时, 中间不可以被加塞或者被分割, 需要整体完整; 要么同时成功, 要么同时失败

### 有序性 ###
![](https://i.imgur.com/0Mva8jH.png)

## 多线程单例模式 ##
1. DCL (Double Check Lock, 双端检锁机制)
```
public class SingletonDemo {
    private static volatile SingletonDemo instance = null;

    private SingletonDemo() {
        System.out.println(Thread.currentThread().getName() + "\t我是构造方法SingletonDemo()");
    }

    public static SingletonDemo getInstance() {
        if (instance == null) {
            synchronized (SingletonDemo.class) {
                if (instance == null) {
                    instance = new SingletonDemo();
                }
            }
        }
        return instance;
    }
}
```
![](https://i.imgur.com/oTqwKdL.png)   
![](https://i.imgur.com/rEITlXy.png)

2. 通过静态内部类, 构造延时加载的单例模式
 ```
class StaticSingleton {
    private StaticSingleton() {
        System.out.println("StaticSingleton is created");
    }

    private static class SingletonHolder {
        private static StaticSingleton instance = new StaticSingleton();
    }

    public static StaticSingleton getInstance() {
        return SingletonHolder.instance;
    }
}
 ```

## CAS ##
1. Campare And Swap, 比较并交换
2. 底层原理: Unsafe类 + 自旋锁

### Unsafe类 ###
1. AtomicInteger底层实现
 ```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
	...
}
 ```
2. rt.jar\sun\misc\Unsafe.class  
![](https://i.imgur.com/CxpBUzT.png)
3. getAndIncrement()方法底层实现
 ```
    /**
     * Atomically increments by one the current value.
     *
     * @return the previous value
     */
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
 ```
![](https://i.imgur.com/fI25TK8.png)  
![](https://i.imgur.com/uqo2NDj.png)
4. 小总结
![](https://i.imgur.com/nMQRi9d.png)
5. CAS缺点  
 - 循环时间长, 开销很大
![](https://i.imgur.com/V2chYZ0.png)
 - 只能保证一个共享变量的原子操作
![](https://i.imgur.com/EidgE1w.png)
 - 引出来ABA问题???

### ABA问题 ###
1. "狸猫换太子"  
![](https://i.imgur.com/BYPCM2v.png)
2. 解决ABA问题
 ```
public class AbaDemo {

    static AtomicReference<Integer> atomicReference = new AtomicReference<>(100);
    static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {

        System.out.println("===== 以下是ABA问题的产生 =====");
        new Thread(() -> {
            atomicReference.compareAndSet(100, 101);
            atomicReference.compareAndSet(101, 100);
        }, "t1").start();

        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println(atomicReference.compareAndSet(100, 2019) + "\t" + atomicReference.get()); //2019
        }, "t2").start();

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("===== 以下是ABA问题的解决=====");
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t第一次版本号: " + atomicStampedReference.getStamp()); //1
            //t3线程暂停1秒钟
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (Exception e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第二次版本号: " + atomicStampedReference.getStamp()); //2
            atomicStampedReference.compareAndSet(101, 100, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第三次版本号: " + atomicStampedReference.getStamp()); //3
        }, "t3").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t第一次版本号: " + stamp); //1
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (Exception e) {
                e.printStackTrace();
            }
            boolean b = atomicStampedReference.compareAndSet(100, 2019, stamp, stamp + 1);
            if (b) 
                System.out.println(Thread.currentThread().getName() + "\t最新版本号: " + atomicStampedReference.getStamp());
            else System.out.println("未成功!!!");
            System.out.println(Thread.currentThread().getName() + "\t最新值为: " + atomicStampedReference.getReference()); //100
        }, "t4").start();
    }
}
 ```

## Java锁 ##
### 公平锁和非公平锁 ###
1. 是什么  
![](https://i.imgur.com/BqgLhH4.png)
2. 区别  
![](https://i.imgur.com/BWm6Wk9.png)

### 可重入锁(递归锁) ###
1. ReentrantLock/Synchronized是典型的可重入锁  
![](https://i.imgur.com/UNGnG2T.png)
2. 可重入锁可以避免死锁(加锁几次就必须解锁几次)

### 自旋锁 ###
1. 是什么  
![](https://i.imgur.com/5AjGXq8.png)
2. 代码验证
 ```
public class SpinLockDemo {
    AtomicReference<Thread> atomicReference = new AtomicReference<>();

    public void myLock() {
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName() + "\tcomes in");
        while (!atomicReference.compareAndSet(null, thread)) {
        }
    }

    public void myUnlock() {
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread, null);
        System.out.println(Thread.currentThread().getName() + "\tinvocks myUnlock()");
    }

    public static void main(String[] args) {
        SpinLockDemo spinLockDemo = new SpinLockDemo();

        new Thread(() -> {
            spinLockDemo.myLock();
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (Exception e) {
                e.printStackTrace();
            }
            spinLockDemo.myUnlock();
        }, "AA").start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (Exception e) {
            e.printStackTrace();
        }

        new Thread(() -> {
            spinLockDemo.myLock();
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (Exception e) {
                e.printStackTrace();
            }
            spinLockDemo.myUnlock();
        }, "BB").start();
    }
}
 ```

### 读写锁 ###
1. 独占锁(写锁)/共享锁(读锁)/互斥锁  
![](https://i.imgur.com/QER7nhC.png)
2. 代码验证
 ```
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        MyCache myCache = new MyCache();

        for (int i = 0; i < 5; i++) {
            final Integer tmpInt = i;
            new Thread(() -> {
                myCache.put(tmpInt.toString(), tmpInt.toString());
            }, String.valueOf(i)).start();
        }

        for (int i = 0; i < 5; i++) {
            final Integer tmpInt = i;
            new Thread(() -> {
                myCache.get(tmpInt.toString());
            }, String.valueOf(i)).start();
        }
    }
}
class MyCache {
    private volatile Map<String, Object> map = new HashMap<>();
    private ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public void put(String key, Object value) {
        rwLock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t正在写入: (" + key + ", " + value + ")");
            try {
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (Exception e) {
                e.printStackTrace();
            }
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "\t写入完成");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public void get(String key) {
        rwLock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t正在读取: " + key);
            try {
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (Exception e) {
                e.printStackTrace();
            }
            Object result = map.get(key);
            System.out.println(Thread.currentThread().getName() + "\t读取完成: " + key + " -> " + result);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rwLock.readLock().unlock();
        }
    }
}
 ```

## 其他并发API ##
### CountDownLatch ###
1. 是什么  
![](https://i.imgur.com/HEp8hvA.png)
2. 代码演示
```
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "国, 被灭亡");
                countDownLatch.countDown();
            }, CountryEnum.foreach_CountryEnum(i)).start();
        }
        countDownLatch.await();
        System.out.println("秦帝国, 一统华夏");
    }
}
.
enum CountryEnum {
    ONE(1, "齐"),
    TWO(2, "楚"),
    THREE(3, "燕"),
    FOUR(4, "赵"),
    FIVE(5, "魏"),
    SIX(6, "韩");

    @Getter
    private Integer retCode;
    @Getter
    private String retMessage;

    CountryEnum(Integer retCode, String retMessage) {
        this.retCode = retCode;
        this.retMessage = retMessage;
    }

    public static String foreach_CountryEnum(int index) {
        CountryEnum[] values = CountryEnum.values();
        for (CountryEnum value : values) 
            if (value.getRetCode() == index) return value.getRetMessage();
        return null;
    }
}
```

### CyclicBarrier ###
CountDownLatch做加法, CyclicBarrier做减法
![](https://i.imgur.com/DY70kBX.png)

### Semaphore ###
CountDownLatch做加法, CyclicBarrier做减法, Semaphore可以加也可以减

## 阻塞队列 ##
1. 概念  
![](https://i.imgur.com/JXrFAm0.png)  
![](https://i.imgur.com/IbNUOqU.png)
2. 优势  
![](https://i.imgur.com/FcdkN4Y.png)
3. BlockingQueue实现类(Queue的子接口)  
![](https://i.imgur.com/A8ZRKTA.png)
4. 常用API  
![](https://i.imgur.com/Z6Ojj8n.png)  
![](https://i.imgur.com/TZOJaaK.png)
5. SynchronousQueue, 队列中只有一个元素, 消费一个生产一个
 ```
public class SynchronousQueueDemo {
    public static void main(String[] args) {
        BlockingQueue<String> blockingQueue = new SynchronousQueue<>();

        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "\tput 1");
                blockingQueue.put("1");
                System.out.println(Thread.currentThread().getName() + "\tput 2");
                blockingQueue.put("2");
                System.out.println(Thread.currentThread().getName() + "\tput 3");
                blockingQueue.put("3");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "AAA").start();

        new Thread(() -> {
            try {
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "\t"+blockingQueue.take());
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "\t"+blockingQueue.take());
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "\t" + blockingQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "BBB").start();
    }
}
 ```

### 用阻塞队列实现生产者-消费者模型 ###
```
public class ProdConsumerDemo {
    public static void main(String[] args) {
        MyResource myResource = new MyResource(new ArrayBlockingQueue<>(10));

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t生产线程启动");
            try {
                myResource.produce();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "Produce").start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t消费线程启动");
            try {
                myResource.consume();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "Consume").start();

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("\n\n5秒钟时间到, 工作结束");
        myResource.stop();
    }
}

class MyResource {
    private volatile boolean FLAG = true; //默认开启, 进行生产 + 消费
    private AtomicInteger atomicInteger = new AtomicInteger(); //多线程场景最好不要用i++
    private BlockingQueue<String> blockingQueue; //只传接口不传实现类, 便于复用

    public MyResource(BlockingQueue<String> blockingQueue) {
        this.blockingQueue = blockingQueue;
        System.out.println(blockingQueue.getClass().getName());
    }

    public void produce() throws Exception {
        String data;
        boolean retValue;
        while (FLAG) { //用while判断条件, 不用if, 防止虚假唤醒
            data = atomicInteger.incrementAndGet() + "";
            retValue = blockingQueue.offer(data, 2L, TimeUnit.SECONDS);
            if (retValue) System.out.println(Thread.currentThread().getName() + "\t插入队列" + data + "成功");
            else System.out.println(Thread.currentThread().getName() + "\t插入队列" + data + "失败");
            TimeUnit.SECONDS.sleep(1);
        }
        System.out.println(Thread.currentThread().getName() + "\tFLAG=false, 生产动作结束");
    }

    public void consume() throws Exception {
        String result;
        while (FLAG) {
            result = blockingQueue.poll(2L, TimeUnit.SECONDS);
            if (result == null || result.equalsIgnoreCase("")) {
                FLAG = false;
                System.out.println(Thread.currentThread().getName() + "\t超过2s没有取到, 消费退出");
                return;
            }
            System.out.println(Thread.currentThread().getName() + "\t消费队列" + result + "成功");
        }
    }

    public void stop() {
        this.FLAG = false;
    }
}
```

## Synchronized和Lock的区别 ##
![](https://i.imgur.com/0jWTTkP.png)

## Callable接口 ##
```
public class Callable07 {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        FutureTask<Integer> futureTask = new FutureTask<>(new MyThread());
        new Thread(futureTask, "A").start();
        //不管起几个线程, 一个FutureTask只会被计算一次
        new Thread(futureTask, "B").start();
        
        //要求获得Callable线程的计算结果, 如果没有计算完成就要去强求, 会造成阻塞直到计算完成, 建议放在最后
        Integer result = futureTask.get();
        System.out.println("**********");
        System.out.println(result);
    }
}

class MyThread implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("come into call() method");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (Exception e) {
            e.printStackTrace();
        }
        //有返回值
        return 1024;
    }
}
```

## 线程池 ##
1. 优势  
![](https://i.imgur.com/sul1J3Y.png)
2. 常用的3种方式(jdk8之前)
`Executors.newFixedThreadPool(5); //1池5线程` -- 执行长期的任务, 性能好很多)  
`Executors.newSingleThreadExecutor(); //1池1线程` -- 一个任务一个任务执行的场景  
`Executors.newCachedThreadPool(); //1池n线程` -- 执行很多短期异步的小程序或者负载比较轻的服务器

### 线程池底层实现 ###
1. ThreadPoolExecutor的7个参数
 ```
/**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * 定义: 线程池中的常驻核心线程数
	 * 1) 在创建了线程池后, 当有请求任务来之后, 就会安排池中的线程去执行请求任务, 近似理解为"今日当值线程"
	 * 2) 当线程池中的线程数目达到corePoolSize后, 就会把到达的任务放到缓存队列当中
     * 
     * @param maximumPoolSize the maximum number of threads to allow in the pool
     * 定义: 线程池能够容纳同时执行的最大线程数, 其值必须大于等于1
     * 
     * @param keepAliveTime when the number of threads is greater than the core, this is the maximum time that excess idle threads will wait for new tasks before terminating.
     * 定义: 多余的空闲线程的存活时间
     * 
     * @param unit the time unit for the {@code keepAliveTime} argument
     * 定义: keepAliveTime的单位
     * 
     * @param workQueue the queue to use for holding tasks before they are executed.  This queue will hold only the {@code Runnable} tasks submitted by the {@code execute} method.
     * 定义: 任务队列, 被提交但尚未被执行的任务
     * 
     * @param threadFactory the factory to use when the executor creates a new thread
     * 定义: 生成线程池中的工作线程的线程工厂, 用于创建线程, 一般用默认即可
     * 
     * @param handler the handler to use when execution is blocked because the thread bounds and queue capacities are reached
     * 定义: 拒绝策略, 表示当队列满了并且工作线程大于等于线程池的最大线程数(maximumPoolSize)
     * 
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
 ```
2. 图解  
![](https://i.imgur.com/L21oW01.png)
3. 工作原理  
 - 图解:  
![](https://i.imgur.com/R5Nuz1N.png)  
 - 文字描述:  
![](https://i.imgur.com/JHMfSkC.png)
4. 拒绝策略
 - 是什么  
![](https://i.imgur.com/iI8xhM3.png)
 - JDK内置的拒绝策略(均实现了RejectedExecutionHandler接口)  
a. AbortPolicy(默认): 直接抛出RejectedExecutionException异常阻止系统正常运行  
b. CallerRunsPolicy: "调用者运行"一种调节机制, 该策略既不会抛弃任务, 也不会抛出异常, 而是将某些任务回退到调用者, 从而降低新任务的流量  
c. DiscardOldestPolicy: 抛弃队列中等待最久的任务, 然后把当前任务加入队列中, 尝试再次提交当前任务  
d. DiscardPolicy: 直接丢弃任务, 不予任何处理也不抛出异常; 如果允许任务丢失, 这是最好的一种方法

### 自定义线程池 ###
1. 不能使用jdk自带的线程池, 要自定义创建
> 【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这
样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。
说明：Executors 返回的线程池对象的弊端如下：  
1） FixedThreadPool 和 SingleThreadPool：  
允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。  
2） CachedThreadPool：  
允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。  
-- 阿里巴巴Java开发手册
2. 手写线程池
 ```
public class ThreadPoolDemo {
    public static void main(String[] args) {
        ExecutorService threadPool = new ThreadPoolExecutor(2, 5, 1L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(3), Executors.defaultThreadFactory(), new ThreadPoolExecutor.DiscardOldestPolicy());
        try {
            //最大为maximumPoolSize + BlockingQueue.size
            for (int i = 0; i < 9; i++)
                threadPool.execute(() -> System.out.println(Thread.currentThread().getName() + "\t办理业务"));
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            threadPool.shutdown();
        }
    }
}
 ```
3. 线程池参数配置  
 - CPU密集型  
![](https://i.imgur.com/nE4lxlR.png)   
 - IO密集型  
a. 第一种  
![](https://i.imgur.com/m8s5lmI.png)    
b. 第二种  
![](https://i.imgur.com/5Li4v5x.png)

## 死锁 ##
1. 是什么  
![](https://i.imgur.com/J3KY7iM.png)
2. 代码演示
 ```
public class DeadLockDemo {
    public static void main(String[] args) {
        String lockA = "lockA";
        String lockB = "lockB";
        new Thread(new HoldLockThread(lockA, lockB), "AAA").start();
        new Thread(new HoldLockThread(lockB, lockA), "BBB").start();
    }
}
class HoldLockThread implements Runnable {
    private String lockA;
    private String lockB;

    public HoldLockThread(String lockA, String lockB) {
        this.lockA = lockA;
        this.lockB = lockB;
    }

    @Override
    public void run() {
        synchronized (lockA) {
            System.out.println(Thread.currentThread().getName() + "\t自己持有: " + lockA + "\t尝试获得: " + lockB);
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (Exception e) {
                e.printStackTrace();
            }
            synchronized (lockB) {
                System.out.println(Thread.currentThread().getName() + "\t自己持有: " + lockB + "\t尝试获得: " + lockA);
            }
        }
    }
}
 ```
3. 解决: 
 ```
1) 在idea的终端, jps -l, 查看进程号
2) jstack (进程号), 生成问题报告 
 ```

## GC Roots ##
1. 什么是"垃圾"? 简单来说就是内存中已经**不再被使用到**的空间就是垃圾
2. 如何判断一个对象是否可以被回收?  
a. 引用计数法(现已不使用)    
![](https://i.imgur.com/EhppcCQ.png)  
b. 枚举根节点做可达性分析(根搜索路径)    
![](https://i.imgur.com/9vZKNY0.png)
 - case, 引用不可达的对象将会被回收  
![](https://i.imgur.com/mthJYw9.png)
 - Java中可以作为GC Roots的对象  
> 虚拟机栈(栈帧中的局部变量区, 也叫做局部变量表)中引用的对象  
> 方法区中的类静态属性引用的对象  
> 方法区中常量引用的对象  
> 本地方法栈中native方法(JNI)引用的对象

## JVM参数 ##
1. JVM参数类型  
 - 标配参数: -version, -help等, 几乎不发生变化的参数  
 - X参数  
a. -Xint: 解释执行  
b. -Xcomp: 第一次使用就编译成本地代码  
c. -Xmixed: 混合模式  
 - XX参数  
a. Boolean类型: -XX:+或者-某个属性值 ("+"表示开启, "-"表示关闭)  
b. KV设值类型: -XX:属性key=属性value  
c. jinfo举例, 如何查看当前运行程序的配置
 ```
jps -l   ->    java进程编号
jinfo -flag 具体参数 java进程编号   ->   查看某个参数
jinfo -flags java进程编号   ->   查看全部参数
 ```
d. 坑题: 如何解释-Xms和-Xmx?  
 ```
这两个参数仍为**XX参数**:  
-Xms == -XX:InitialHeapSize  
-Xmx == -XX:MaxHeapSize
 ```  
2. 查看初始默认值  
 - `java -XX:+PrintFlagsInitial` -> 查看初始参数
 - `java -XX:+PrintFlagsFinal -version` -> 查看修改后的参数 (:=表示被修改过的参数, =表示初)始值
 - `java -XX:+PrintCommandLineFlags -version` -> 查看常用参数(尤其是看最后一个参数: 垃圾回收器)
3. 常用参数
 - -Xms: 初始堆内存大小, 默认为物理内存1/64; 等价于-XX:InitialHeapSize
 - -Xmx: 最大分配堆内存,默认为物理内存的1/4; 等价于-XX:MaxHeapSize
 - -Xss: 设置单个线程栈的大小, 一般默认为512k~1024k (取决于jdk版本和平台, 64位linux + jdk8 = 1024k, 若为0则表示默认值); 等价于-XX:ThreadStackSize
 - -Xmn: 设置年轻代大小
 - -XX:MetaspaceSize: 设置元空间大小  
![](https://i.imgur.com/UwoSQgA.png)
 - -XX:+PrintGCDetails: 输出详细GC收集日志信息(要求会看GC日志)
 - -XX:SurvivorRatio: 设置新生代中eden和s0/s1空间的比例, 默认-XX:SurvivorRatio=8 (s0和s1相同)
 - -XX:NewRatio: 配置老年代与年轻代在堆结构的占比, 默认-XX:NewRatio=2, 新生代1老年代2
 - -XX:MaxTenuringThreshold: 设置垃圾最大年龄 (必须在0到15之间)

## 强软弱虚 ##
1. 强引用(默认): 死了都不收  
![](https://i.imgur.com/GboZyTK.png)
2. 软引用: 内存足够的前提下不收, 内存不够就收, 尽量避免OOM  
![](https://i.imgur.com/GVK8eaH.png)
3. 弱引用: 不管内存够不够, 只要GC就收  
![](https://i.imgur.com/jjVuHvU.png)
4. 软引用和弱引用的适用场景
 - MyBatis底层大量使用软引用和弱引用
 - 维护缓存系统  
![](https://i.imgur.com/JrpSm3A.png)
 - WeakHashMap
> Hash table based implementation of the Map interface, with weak keys. An entry in a WeakHashMap will automatically be removed when its key is no longer in ordinary use. More precisely, the presence of a mapping for a given key will not prevent the key from being discarded by the garbage collector, that is, made finalizable, finalized, and then reclaimed. When a key has been discarded its entry is effectively removed from the map, so this class behaves somewhat differently from other Map implementations. 
5. 引用队列  
![](https://i.imgur.com/vcnedyz.png)
6. 虚引用简介: 类似Spring的后置通知  
![](https://i.imgur.com/VpRpawT.png)
7. 总结  
![](https://i.imgur.com/o0R7d93.png)  
![](https://i.imgur.com/eQTJLx8.png)

## OOM ##
1. java.lang.StackOverflowError  
 - 代码案例
 ```
public class StackOverflowErrorDemo {
    public static void main(String[] args) {
        stackOverflowError();
    }

    private static void stackOverflowError() {
        stackOverflowError();
    }
}
 ```
 - 爆栈是**错误**, 不是异常  
![](https://i.imgur.com/imCFUr2.png)
2. java.lang.OutOfMemoryError: Java heap space
 ```
public class JavaHeapSpaceDemo {
    public static void main(String[] args) {
        String s = "dolly";
        while (true) {
            s += new Random().nextInt(11111111);
            s.intern(); //也是错误
        }
    }
}
 ```
3. java.lang.OutOfMemoryError: GC overhead limit exceeded  
![](https://i.imgur.com/AMcyOd6.png)  
 ```
public class GcOverheadDemo {
    //-Xms10m -Xmx10m -XX:+PrintGCDetails -XX:MaxDirectMemorySize=5m
    public static void main(String[] args) {
        int i = 0;
        List<String> list = new ArrayList<>();
        try {
            while (true) list.add(String.valueOf(++i).intern());
        } catch (Exception e) {
            System.out.println("===== i: " + i + " =====");
            e.printStackTrace();
            throw e;
        }
    }
}
 ```
4. java.lang.OutOfMemoryError: Direct buffer memory  

5. java.lang.OutOfMemoryError: Unable to create new native thread
6. java.lang.OutOfMemoryError: Metaspace