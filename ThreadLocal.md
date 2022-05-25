# ThreadLocal

## 与synchronized的区别

ThreadLocal模式与synchronized关键字都用于处理多线程并发访问变量的问题, 不过两者处理问题的角度和思路不同。

|        | synchronized                                                 | ThreadLocal                                                  |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 原理   | 同步机制采用'以时间换空间'的方式, 只提供了一份变量,让不同的线程排队访问 | ThreadLocal采用'以空间换时间'的方式, 为每一个线程都提供了一份变量的副本,从而实现同时访问而相不干扰 |
| 侧重点 | 多个线程之间访问资源的同步                                   | 多线程中让每个线程之间的数据相互隔离                         |

## 数据结构

### 早期设计

每个`ThreadLocal`都创建一个`Map`，然后用线程作为`Map`的`key`，要存储的局部变量作为`Map`的`value`

![img](https://github.com/cbxbj/threadLocal/blob/master/img/001.png)

### 现在设计

在JDK8中 `ThreadLocal`的设计是：每个`Thread`维护一个`ThreadLocalMap`，这个Map的`key`是`ThreadLocal`实例本身，`value`才是真正要存储的值`Object`

![img](https://github.com/cbxbj/threadLocal/blob/master/img/002.png)

### 好处

1. 这样设计之后每个`Map`存储的`Entry`数量就会变少。因为之前的存储数量由`Thread`的数量决定，现在是由`ThreadLocal`的数量决定。在实际运用当中，往往ThreadLocal的数量要少于Thread的数量。

2. 当`Thread`销毁之后，对应的`ThreadLocalMap`也会随之销毁，能减少内存的使用。

## 详细结构

![img](https://github.com/cbxbj/threadLocal/blob/master/img/003.png)

`Thread`类有一个类型为`ThreadLocal.ThreadLocalMap`的实例变量`threadLocals`，也就是说每个线程有一个自己的`ThreadLocalMap`。

`ThreadLocalMap`有自己的独立实现，可以简单地将它的`key`视作`ThreadLocal`，`value`为代码中放入的值（实际上`key`并不是`ThreadLocal`本身，而是它的一个**弱引用**）。

每个线程在往`ThreadLocal`里放值的时候，都会往自己的`ThreadLocalMap`里存，读也是以`ThreadLocal`作为引用，在自己的`map`里找对应的`key`，从而实现了**线程隔离**。

`ThreadLocalMap`有点类似`HashMap`的结构，只是`HashMap`是由**数组+链表**实现的，而`ThreadLocalMap`中并没有**链表**结构。

我们还要注意`Entry`， 它的`key`是`ThreadLocal<?> k` ，继承自`WeakReference`， 也就是我们常说的弱引用类型。

### 扩展：引用类型

强引用、软引用、弱引用、虚引用、终结器引用

![img](https://github.com/cbxbj/threadLocal/blob/master/img/004.png)

#### 强引用

只要有一个GC ROOT强引用该对象,垃圾回收时，该对象就不会被垃圾回收

- 只有所有 GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收

#### 软引用（SoftReference）

当一个对象没有强引用，只有软引用，垃圾回收后发现内存仍不够就会发生该对象释放

- 仅有软引用引用该对象时，在垃圾回收后，内存仍不足时会再次出发垃圾回收，回收软引用 对象
- 可以配合引用队列来释放软引用自身

#### 弱引用（WeakReference）

当一个对象没有强引用，只有弱引用，只要发生垃圾回收就会把该对象回收

- 仅有弱引用引用该对象时，在垃圾回收时，无论内存是否充足，都会回收弱引用对象
- 可以配合引用队列来释放弱引用自身

**补充:**

- 软引用、弱引用还可配合**引用队列**一起工作
- 软弱引用,引用的对象被回收后,**自身也是对象**
- 会进入引用队列，进入引用队列的对象会被进一步处理释放(**自身被强引用引用**)

#### 虚引用（PhantomReference）

**虚引用和终结器引用必须配合引用队列使用(创建时会被关联一个引用队列)**

- 必须配合引用队列使用，主要配合 ByteBuffer 使用，被引用对象回收时，会将虚引用入队， 由 Reference Handler 线程调用虚引用相关方法释放直接内存

> 回忆:创建直接内存ByteBuffer.allocateDirect(1)时,会有一个Cleaner虚引用对象创建
>
> ByteBuffer被回收的时候Cleaner虚引用对象会进入引用队列，
>
> Reference Handler线程定时会查找虚引用队列里有没有对象,有的话就会调用虚引用的clean方法
>
> Cleaner的clean方法会调用unsafe的freeMemory方法释放直接内存

#### 终结器引用（FinalReference）

- 无需手动编码，但其内部配合引用队列使用，在垃圾回收时，终结器引用入队（被引用对象 暂时没有被回收），再由 Finalizer 线程通过终结器引用找到被引用对象并调用它的 finalize 方法，第二次 GC 时才能回收被引用对象

Object对象中有一个finalize()

```java
protected void finalize() throws Throwable { }
```

- 一个对象重写了finalize(),并且没有强引用引用该对象时
- 虚拟机会创建对应的终结器引用
- 当该对象被垃圾回收时，就会把对应的终结器引用加入引用队列(这时该方法没有被垃圾回收)
- 优先级很低的线程Finalize Handler线程查看引用队列中是否有终结器引用
- 有的话就会根据终结器引用找到该对象，并调用finalize()方法
- 调用后,下次垃圾回收就会把该对象回收掉

##  GC 之后 key 是否为 null？

![img](https://github.com/cbxbj/threadLocal/blob/master/img/005.png)

图示中：Heap中的ThreadLocal被2个对象引用 

1. Stack中threadLocalref强引用
2. Heap中Entry中的key弱引用

当Stack中的threadLocal弹栈后，Heap中的ThreadLocal就只剩下一个弱引用，下次GC就会为null

```java
public class ThreadLocalDemo {

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(ThreadLocalDemo::test);
        t.start();
        t.join();
    }

    private static void test()  {
        try {
            new ThreadLocal<>().set("def"); //关键点
            System.gc();
            Thread t = Thread.currentThread();
            Class<? extends Thread> clz = t.getClass();
            Field field = clz.getDeclaredField("threadLocals");
            field.setAccessible(true);
            Object ThreadLocalMap = field.get(t);
            Class<?> tlmClass = ThreadLocalMap.getClass();
            Field tableField = tlmClass.getDeclaredField("table");
            tableField.setAccessible(true);
            Object[] arr = (Object[]) tableField.get(ThreadLocalMap);
            for (Object o : arr) {
                if (o != null) {
                    Class<?> entryClass = o.getClass();
                    Field valueField = entryClass.getDeclaredField("value");
                    Field referenceField = entryClass.getSuperclass().getSuperclass().getDeclaredField("referent");
                    valueField.setAccessible(true);
                    referenceField.setAccessible(true);
                    System.out.printf("弱引用key:%s,值:%s%n", referenceField.get(o), valueField.get(o));
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

结果如下：

```java
弱引用key:null,值:def
```

**分析**：Heap中的ThreadLocal自始至终只被1个对象引用：Heap中Entry中的key弱引用，所以GC后为null

将上述代码中关键点改成

```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();
threadLocal.set("def");
```

结果如下：

```java
弱引用key:java.lang.ThreadLocal@4c0edf85,值:def
```

**分析**：Heap中的ThreadLocal有2个对象引用：Heap中Entry中的key弱引用、Stack中threadLocal强引用，所以GC后不为null

## ThreadLocal.set()方法源码详解

![img](https://github.com/cbxbj/threadLocal/blob/master/img/006.png)

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value); //map存在则set
    else
        createMap(t, value); //map不存在则创建map
}
```

### 分支一：createMap(t, value)

#### `ThreadLocalMap` 桶的默认值

```java
private static final int INITIAL_CAPACITY = 16;
```

#### `ThreadLocalMap` Hash 算法

```java
int i = key.threadLocalHashCode & (len-1);
```

`ThreadLocalMap`中`hash`算法很简单，这里`i`就是当前 key 在散列表中对应的数组下标位置。

这里最关键的就是`threadLocalHashCode`值的计算，`ThreadLocal`中有一个属性为`HASH_INCREMENT = 0x61c88647`

```java
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode = new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    static class ThreadLocalMap {
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
}
```

每当创建一个`ThreadLocal`对象，这个`ThreadLocal.nextHashCode` 这个值就会增长 `0x61c88647` 。

这个值很特殊，它是**斐波那契数** 也叫 **黄金分割数**。`hash`增量为 这个数字，带来的好处就是 `hash` **分布非常均匀**。

##### 扩展：Hash算法

2. HashMap
3. ConcurrentHashMap
4. Dict
5. Redis插槽
6. StrngTable

###### HashMap

桶的个数必须是2的n次方，默认起始大小16

```java
i = (n - 1) & ((h = key.hashCode()) ^ (h >>> 16))
```

###### ConcurrentHashMap

桶的个数必须是2的n次方，默认起始大小16

```java
static final int HASH_BITS = 0x7fffffff; //用于计算key的hash值时把hash值控制到integer的最大值以内

i = (n - 1) & ((h = key.hashCode() ^ (h >>> 16)) & HASH_BITS)
```

###### Dict

桶的个数 是2的n次方，起始大小4

```c
hashFunction(key) & sizemask //sizemask为桶大小减1
```

###### Redis插槽

桶的个数是16384，2的n次方，不可扩容

```c
getCRC16(key) & 0x3FFF; //0x3FFF = 16383
```

###### StringTable

1. 底层为hash表,桶个数越多,hash碰撞几率减少,查找速度变快

2. jdk1.8存储在Heap中

   > 为什么从方法区(永久代中)移到Heap中，感兴趣的同学自行查阅资料，此处不做过多讲解

3. 桶的个数默认：60013，不可扩容

   > -XX:StringTableSize=桶个数，该参数可更改桶的个数(范围：1009~2305843009213693951)

#### `ThreadLocalMap` 扩容阈值

```java
//The next size value at which to resize.  要调整大小的下一个大小值
private int threshold;

ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //略...
	setThreshold(INITIAL_CAPACITY);
}

private void setThreshold(int len) {
    threshold = len * 2 / 3; 
}
```

### 分支二：map.set(this, value)

#### `ThreadLocalMap` Hash 冲突

> **注明：** 下面所有示例图中，**绿色块**`Entry`代表**正常数据**，**灰色块**代表`Entry`的`key`值为`null`，**已被垃圾回收**。**白色块**表示`Entry`为`null`。

虽然`ThreadLocalMap`中使用了**黄金分割数**来作为`hash`计算因子，大大减少了`Hash`冲突的概率，但是仍然会存在冲突。

`HashMap`中解决冲突的方法是在数组上构造一个**链表**结构，冲突的数据挂载到链表上，如果链表长度超过一定数量则会转化成**红黑树**。

而 `ThreadLocalMap` 中并没有链表结构，所以这里不能使用 `HashMap` 解决冲突的方式了。

![img](https://github.com/cbxbj/threadLocal/blob/master/img/007.png)

如上图所示，如果我们插入一个`value=27`的数据，通过 `hash` 计算后应该落入槽位 4 中，而槽位 4 已经有了 `Entry` 数据。

此时就会线性向后查找，一直找到 `Entry` 为 `null` 的槽位才会停止查找，将当前元素放入此槽位中。

##### 扩展：Hash冲突

1. HashMap
2. ConcurrentHashMap
3. Dict

###### HashMap/ConcurrentHashMap

put时得到角标，获取该角标的Entry是否为空

- 空：直接插入
- 非空：依次遍历是否存在hashCode相同的key
  - 存在：比较equals看是否相等
    - 相等：覆盖
    - 不相等：尾插
  - 不存在：尾插

当Map中的其中一个链表的对象个数如果达到了8个，此时如果数组长度没有达到64，那么Map会扩容

当Map中的其中一个链表的对象个数如果达到了8个，此时如果数组长度大于等于64，那么Map会链表-->红黑树

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    
    static final int TREEIFY_THRESHOLD = 8;
    
    static final int MIN_TREEIFY_CAPACITY = 64;

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //略...
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        //略...
    }

    final void treeifyBin(Node<K,V>[] tab, int hash) {
        //略...
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();//扩容
        //略...
    }
}
```

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    
    static final int TREEIFY_THRESHOLD = 8;
    
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //略...
        if (binCount >= TREEIFY_THRESHOLD)
            treeifyBin(tab, i);
        //略...
    }
    
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        //略...
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);//扩容
        //略...
    }
}
```

###### Dict

数据结构：数组+链表

头插(单线程，不用担心并发死链)

#### 图示详解

往`ThreadLocalMap`中`set`数据（**新增**或者**更新**数据）分为好几种情况，针对不同的情况我们画图来说明。

**第一种情况：** 通过`hash`计算后的槽位对应的`Entry`数据为空：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/008.png)

这里直接将数据放到该槽位即可。

**第二种情况：** 槽位数据不为空，`key`值与当前`ThreadLocal`通过`hash`计算获取的`key`值一致：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/009.png)

这里直接更新该槽位的数据。

**第三种情况：** 槽位数据不为空，往后遍历过程中，在找到`Entry`为`null`的槽位之前，没有遇到`key`过期的`Entry`：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/010.png)

遍历散列数组，线性往后查找，如果找到`Entry`为`null`的槽位，则将数据放入该槽位中，或者往后遍历过程中，遇到了**key 值相等**的数据，直接更新即可。

**第四种情况：** 槽位数据不为空，往后遍历过程中，在找到`Entry`为`null`的槽位之前，遇到`key`过期的`Entry`，如下图，往后遍历过程中，遇到了`index=7`的槽位数据`Entry`的`key=null`：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/011.png)

散列数组下标为 7 位置对应的`Entry`数据`key`为`null`，表明此数据`key`值已经被垃圾回收掉了，此时就会执行`replaceStaleEntry()`方法，该方法含义是**替换过期数据的逻辑**，以**index=7**位起点开始遍历，进行探测式数据清理工作。

初始化探测式清理过期数据扫描的开始位置：`slotToExpunge = staleSlot = 7`

以当前`staleSlot`开始 向前迭代查找，找其他过期的数据，然后更新过期数据起始扫描下标`slotToExpunge`。`for`循环迭代，直到碰到`Entry`为`null`结束。

如果找到了过期的数据，继续向前迭代，直到遇到`Entry=null`的槽位才停止迭代，如下图所示，**slotToExpunge 被更新为 0**：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/012.png)

上面向前迭代的操作是为了更新探测清理过期数据的起始下标`slotToExpunge`的值，这个值在后面会讲解，它是用来判断当前过期槽位`staleSlot`之前是否还有过期元素。

接着开始以`staleSlot`位置(`index=7`)向后迭代，**如果找到了相同 key 值的 Entry 数据：**

![img](https://github.com/cbxbj/threadLocal/blob/master/img/013.png)

从当前节点`staleSlot`向后查找`key`值相等的`Entry`元素，找到后更新`Entry`的值并交换`staleSlot`元素的位置(`staleSlot`位置为过期元素)，更新`Entry`数据，然后开始进行过期`Entry`的清理工作，如下图所示：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/014.png)

向后遍历过程中，如果没有找到相同 key 值的 Entry 数据：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/015.png)

从当前节点`staleSlot`向后查找`key`值相等的`Entry`元素，直到`Entry`为`null`则停止寻找。通过上图可知，此时`table`中没有`key`值相同的`Entry`。

创建新的`Entry`，替换`table[stableSlot]`位置：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/016.png)

替换完成后也是进行过期元素清理工作，清理工作主要是有两个方法：`expungeStaleEntry()`和`cleanSomeSlots()`，具体细节后面会讲到

#### 源码详解

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

`for`循环遍历，向后查找，我们先看下`nextIndex()`、`prevIndex()`方法实现：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/017.png)

```java
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

接着看剩下`for`循环中的逻辑：

1. 遍历当前`key`值对应的桶中`Entry`数据为空，这说明散列数组这里没有数据冲突，跳出`for`循环，直接`set`数据到对应的桶中
2. 如果`key`值对应的桶中`Entry`数据不为空
   2.1 如果`k = key`，说明当前`set`操作是一个替换操作，做替换逻辑，直接返回
   2.2 如果`key = null`，说明当前桶位置的`Entry`是过期数据，执行`replaceStaleEntry()`方法(核心方法)，然后返回
3. `for`循环执行完毕，继续往下执行说明向后迭代的过程中遇到了`entry`为`null`的情况
   3.1 在`Entry`为`null`的桶中创建一个新的`Entry`对象
   3.2 执行`++size`操作
4. 调用`cleanSomeSlots()`做一次启发式清理工作，清理散列数组中`Entry`的`key`过期的数据
   4.1 如果清理工作完成后，未清理到任何数据，且`size`超过了阈值(数组长度的 2/3)，进行`rehash()`操作
   4.2 `rehash()`中会先进行一轮探测式清理，清理过期`key`，清理完成后如果**size >= threshold - threshold / 4**，就会执行真正的扩容逻辑(扩容逻辑往后看)

接着重点看下`replaceStaleEntry()`方法，`replaceStaleEntry()`方法提供替换过期数据的功能，我们可以对应上面**第四种情况**的原理图来再回顾下，具体代码如下：

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))

        if (e.get() == null)
            slotToExpunge = i;

    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {

        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

`slotToExpunge`表示开始探测式清理过期数据的开始下标，默认从当前的`staleSlot`开始。以当前的`staleSlot`开始，向前迭代查找，找到没有过期的数据，`for`循环一直碰到`Entry`为`null`才会结束。如果向前找到了过期数据，更新探测清理过期数据的开始下标为 i，即`slotToExpunge=i`

```java
for (int i = prevIndex(staleSlot, len);
     (e = tab[i]) != null;
     i = prevIndex(i, len)){

    if (e.get() == null){
        slotToExpunge = i;
    }
}
```

接着开始从`staleSlot`向后查找，也是碰到`Entry`为`null`的桶结束。 如果迭代过程中，**碰到 k == key**，这说明这里是替换逻辑，替换新数据并且交换当前`staleSlot`位置。如果`slotToExpunge == staleSlot`，这说明`replaceStaleEntry()`一开始向前查找过期数据时并未找到过期的`Entry`数据，接着向后查找过程中也未发现过期数据，修改开始探测式清理过期数据的下标为当前循环的 index，即`slotToExpunge = i`。最后调用`cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);`进行启发式过期数据清理。

```java
if (k == key) {
    e.value = value;

    tab[i] = tab[staleSlot];
    tab[staleSlot] = e;

    if (slotToExpunge == staleSlot)
        slotToExpunge = i;

    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    return;
}
```

`cleanSomeSlots()`和`expungeStaleEntry()`方法后面都会细讲，这两个是和清理相关的方法，一个是过期`key`相关`Entry`的启发式清理(`Heuristically scan`)，另一个是过期`key`相关`Entry`的探测式清理。

**如果 k != key**则会接着往下走，`k == null`说明当前遍历的`Entry`是一个过期数据，`slotToExpunge == staleSlot`说明，一开始的向前查找数据并未找到过期的`Entry`。如果条件成立，则更新`slotToExpunge` 为当前位置，这个前提是前驱节点扫描时未发现过期数据。

```java
if (k == null && slotToExpunge == staleSlot)
    slotToExpunge = i;
```

往后迭代的过程中如果没有找到`k == key`的数据，且碰到`Entry`为`null`的数据，则结束当前的迭代操作。此时说明这里是一个添加的逻辑，将新的数据添加到`table[staleSlot]` 对应的`slot`中。

```java
tab[staleSlot].value = null;
tab[staleSlot] = new Entry(key, value);
```

最后判断除了`staleSlot`以外，还发现了其他过期的`slot`数据，就要开启清理数据的逻辑：

```java
if (slotToExpunge != staleSlot)
    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
```

### `ThreadLocalMap`过期 key 的探测式清理流程

上面有提及`ThreadLocalMap`的两种过期`key`数据清理方式：**探测式清理**和**启发式清理**。

我们先讲下探测式清理，也就是`expungeStaleEntry`方法，遍历散列数组，从开始位置向后探测清理过期数据，将过期数据的`Entry`设置为`null`，沿途中碰到未过期的数据则将此数据`rehash`后重新在`table`数组中定位，如果定位的位置已经有了数据，则会将未过期的数据放到最靠近此位置的`Entry=null`的桶中，使`rehash`后的`Entry`数据距离正确的桶的位置更近一些。操作逻辑如下：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/018.png)

如上图，`set(27)` 经过 hash 计算后应该落到`index=4`的桶中，由于`index=4`桶已经有了数据，所以往后迭代最终数据放入到`index=7`的桶中，放入后一段时间后`index=5`中的`Entry`数据`key`变为了`null`

![img](https://github.com/cbxbj/threadLocal/blob/master/img/019.png)

如果再有其他数据`set`到`map`中，就会触发**探测式清理**操作。

如上图，执行**探测式清理**后，`index=5`的数据被清理掉，继续往后迭代，到`index=7`的元素时，经过`rehash`后发现该元素正确的`index=4`，而此位置已经有了数据，往后查找离`index=4`最近的`Entry=null`的节点(刚被探测式清理掉的数据：`index=5`)，找到后移动`index= 7`的数据到`index=5`中，此时桶的位置离正确的位置`index=4`更近了。

经过一轮探测式清理后，`key`过期的数据会被清理掉，没过期的数据经过`rehash`重定位后所处的桶位置理论上更接近`i= key.hashCode & (tab.len - 1)`的位置。这种优化会提高整个散列表查询性能。

接着看下`expungeStaleEntry()`具体流程，我们还是以先原理图后源码讲解的方式来一步步梳理：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/020.png)

我们假设`expungeStaleEntry(3)` 来调用此方法，如上图所示，我们可以看到`ThreadLocalMap`中`table`的数据情况，接着执行清理操作：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/021.png)

第一步是清空当前`staleSlot`位置的数据，`index=3`位置的`Entry`变成了`null`。然后接着往后探测：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/022.png)

执行完第二步后，index=4 的元素挪到 index=3 的槽位中。

继续往后迭代检查，碰到正常数据，计算该数据位置是否偏移，如果被偏移，则重新计算`slot`位置，目的是让正常数据尽可能存放在正确位置或离正确位置更近的位置

![img](https://github.com/cbxbj/threadLocal/blob/master/img/023.png)

在往后迭代的过程中碰到空的槽位，终止探测，这样一轮探测式清理工作就完成了，接着我们继续看看具体**实现源代码**：

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

这里我们还是以`staleSlot=3` 来做示例说明，首先是将`tab[staleSlot]`槽位的数据清空，然后设置`size--` 接着以`staleSlot`位置往后迭代，如果遇到`k==null`的过期数据，也是清空该槽位数据，然后`size--`

```java
ThreadLocal<?> k = e.get();

if (k == null) {
    e.value = null;
    tab[i] = null;
    size--;
}
```

如果`key`没有过期，重新计算当前`key`的下标位置是不是当前槽位下标位置，如果不是，那么说明产生了`hash`冲突，此时以新计算出来正确的槽位位置往后迭代，找到最近一个可以存放`entry`的位置。

```java
int h = k.threadLocalHashCode & (len - 1);
if (h != i) {
    tab[i] = null;

    while (tab[h] != null)
        h = nextIndex(h, len);

    tab[h] = e;
}
```

这里是处理正常的产生`Hash`冲突的数据，经过迭代后，有过`Hash`冲突数据的`Entry`位置会更靠近正确位置，这样的话，查询的时候 效率才会更高。

## `ThreadLocalMap`扩容机制

在`ThreadLocalMap.set()`方法的最后，如果执行完启发式清理工作后，未清理到任何数据，且当前散列数组中`Entry`的数量已经达到了列表的扩容阈值`(len*2/3)`，就开始执行`rehash()`逻辑：

```java
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
```

接着看下`rehash()`具体实现：

```java
private void rehash() {
    expungeStaleEntries();

    if (size >= threshold - threshold / 4)
        resize();
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

这里首先是会进行探测式清理工作，从`table`的起始位置往后清理，上面有分析清理的详细流程。清理完成之后，`table`中可能有一些`key`为`null`的`Entry`数据被清理掉，所以此时通过判断`size >= threshold - threshold / 4` 也就是`size >= threshold * 3/4` 来决定是否扩容。

我们还记得上面进行`rehash()`的阈值是`size >= threshold`，所以当面试官套路我们`ThreadLocalMap`扩容机制的时候 我们一定要说清楚这两个步骤：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/024.png)

接着看看具体的`resize()`方法，为了方便演示，我们以`oldTab.len=8`来举例：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/025.png)

扩容后的`tab`的大小为`oldLen * 2`，然后遍历老的散列表，重新计算`hash`位置，然后放到新的`tab`数组中，如果出现`hash`冲突则往后寻找最近的`entry`为`null`的槽位，遍历完成之后，`oldTab`中所有的`entry`数据都已经放入到新的`tab`中了。重新计算`tab`下次扩容的**阈值**，具体代码如下：

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null;
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

## `ThreadLocalMap.get()`详解

### `ThreadLocalMap.get()`图解

**第一种情况：** 通过查找`key`值计算出散列表中`slot`位置，然后该`slot`位置中的`Entry.key`和查找的`key`一致，则直接返回：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/026.png)

**第二种情况：** `slot`位置中的`Entry.key`和要查找的`key`不一致：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/027.png)

我们以`get(ThreadLocal1)`为例，通过`hash`计算后，正确的`slot`位置应该是 4，而`index=4`的槽位已经有了数据，且`key`值不等于`ThreadLocal1`，所以需要继续往后迭代查找。

迭代到`index=5`的数据时，此时`Entry.key=null`，触发一次探测式数据回收操作，执行`expungeStaleEntry()`方法，执行完后，`index 5,8`的数据都会被回收，而`index 6,7`的数据都会前移，此时继续往后迭代，到`index = 6`的时候即找到了`key`值相等的`Entry`数据，如下图所示：

![img](https://github.com/cbxbj/threadLocal/blob/master/img/028.png)

### `ThreadLocalMap.get()`源码详解

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

## `ThreadLocalMap`过期 key 的启发式清理流程

上面多次提及到`ThreadLocalMap`过期key的两种清理方式：**探测式清理(expungeStaleEntry())**、**启发式清理(cleanSomeSlots())**

探测式清理是以当前`Entry` 往后清理，遇到值为`null`则结束清理，属于**线性探测清理**。

而启发式清理被作者定义为：**Heuristically scan some cells looking for stale entries**.

![img](https://github.com/cbxbj/threadLocal/blob/master/img/029.png)

具体代码如下：

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

## `InheritableThreadLocal`

我们使用`ThreadLocal`的时候，在异步场景下是无法给子线程共享父线程中创建的线程副本数据的。

为了解决这个问题，JDK 中还有一个`InheritableThreadLocal`类，我们来看一个例子：

```java
public class InheritableThreadLocalDemo {
    public static void main(String[] args) {
        ThreadLocal<String> ThreadLocal = new ThreadLocal<>();
        ThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
        ThreadLocal.set("父类数据:threadLocal");
        inheritableThreadLocal.set("父类数据:inheritableThreadLocal");

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("子线程获取父类ThreadLocal数据：" + ThreadLocal.get());
                System.out.println("子线程获取父类inheritableThreadLocal数据：" + inheritableThreadLocal.get());
            }
        }).start();
    }
}
```

打印结果：

```java
子线程获取父类ThreadLocal数据：null
子线程获取父类inheritableThreadLocal数据：父类数据:inheritableThreadLocal
```

实现原理是子线程是通过在父线程中通过调用`new Thread()`方法来创建子线程，`Thread#init`方法在`Thread`的构造方法中被调用。在`init`方法中拷贝父线程数据到子线程中：

```java
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    this.stackSize = stackSize;
    tid = nextThreadID();
}
```

但`InheritableThreadLocal`仍然有缺陷，一般我们做异步化处理都是使用的线程池，而`InheritableThreadLocal`是在`new Thread`中的`init()`方法给赋值的，而线程池是线程复用的逻辑，所以这里会存在问题。

当然，有问题出现就会有解决问题的方案，阿里巴巴开源了一个`TransmittableThreadLocal`组件就可以解决这个问题，这里就不再延伸，感兴趣的可自行查阅资料。
