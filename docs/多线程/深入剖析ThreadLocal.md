## 前言
朋友们在遇到线程安全问题的时候，大多数情况下可能会使用synchronized关键字，每次只允许一个线程进入锁定的方法或代码块，这样就可以保证操作的原子性，保证对公共资源的修改不会出现莫名其妙的问题。这种加锁的机制，在并发量小的情况下还好，如果并发量较大时，会有大量的线程等待同一个对象锁，会造成系统吞吐量直线下降。

JDK的开发者可能也考虑到使用synchronized的弊端，于是出现了volatile 和 ThreadLocal等另外的思路解决线程安全问题。volatile它所修饰的变量不保留拷贝，直接访问主内存，主要用于一写多读的场景。ThreadLocal是给每一个线程都创建变量的副本，保证每个线程访问都是自己的副本，相互隔离，就不会出现线程安全问题，这种方式其实用空间换时间的做法。其他的内容以后有空再讨论，今天我们重点聊一下 ThreadLocal。

接下来，我们将从以下几个方面介绍ThreadLocal

如何使用ThreadLocal？
- ThreadLocal的工作原理
- ThreadLocal源码解析
- ThreadLocal有哪些坑

## 1.如何使用ThreadLocal？

在使用ThreadLocal之前我们先一起看个例子
```java
/**
 * 不安全线程场景
 *
 * @author sue
 * @date 2020/8/12 21:21
 */
public class TestThread {

    private int count = 0;

    public void calc() {
        count++;
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {
        TestThread testThread = new TestThread();
        for (int i = 0; i < 20; i++) {
            new ThreadA(i, testThread).start();
        }
        Thread.sleep(200);
        System.out.println("realCount:" + testThread.getCount());
    }
}

class ThreadA extends Thread {

    private int i;
    private TestThread testThread;

    ThreadA(int i, TestThread testThread) {
        this.i = i;
        this.testThread = testThread;
    }

    public void run() {
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        testThread.calc();
        System.out.println("i:" + i + ",count:" + testThread.getCount());
    }
}
```
运行结果：
```java
i:8,count:8
i:7,count:8
i:11,count:10
i:4,count:11
i:13,count:12
i:2,count:8
i:0,count:8
i:9,count:8
i:3,count:8
i:1,count:8
i:5,count:8
i:6,count:8
i:12,count:11
i:10,count:9
i:14,count:15
i:18,count:17
i:15,count:18
i:17,count:16
i:16,count:15
i:19,count:18
realCount:18
```
我们可以看到，realCount最终出现错误，预计的结果应该是20，实际情况却是18，出现了线程安全问题。

接下来，把程序改成ThreadLocal运行结果会怎样？
```java
/**
 * ThreadLocal场景
 *
 * @author sue
 * @date 2020/8/12 21:21
 */
public class TestThreadLocal {

    private ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

    public void calc() {
        threadLocal.set(getCount() + 1);
    }

    public int getCount() {
        Integer integer = threadLocal.get();
        return integer != null ? integer : 0;
    }

    public static void main(String[] args) throws InterruptedException {
        TestThreadLocal testThreadLocal = new TestThreadLocal();
        for (int i = 0; i < 20; i++) {
            new ThreadB(i, testThreadLocal).start();
        }
        Thread.sleep(200);
        System.out.println("realCount:" + testThreadLocal.getCount());
    }

}

class ThreadB extends Thread {

    private int i;
    private TestThreadLocal testThreadLocal;

    ThreadB(int i, TestThreadLocal testThreadLocal) {
        this.i = i;
        this.testThreadLocal = testThreadLocal;
    }

    public void run() {
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        testThreadLocal.calc();
        System.out.println("i:" + i + ",count:" + testThreadLocal.getCount());
    }
}
```
运行结果：
```java
i:6,count:1
i:10,count:1
i:3,count:1
i:0,count:1
i:7,count:1
i:11,count:1
i:9,count:1
i:5,count:1
i:8,count:1
i:1,count:1
i:4,count:1
i:2,count:1
i:13,count:1
i:15,count:1
i:14,count:1
i:19,count:1
i:18,count:1
i:17,count:1
i:12,count:1
i:16,count:1
realCount:0
```
我们可以看到，跟之前的例子运行结果差别很大，首先现在count全部都是1，之前count有8，10，11，12等很多值。其次realCount之前是18，现在的realCount却是0。为什么会造成这样的差异呢？

## 2.ThreadLocal的工作原理

先看看示例1中的情况
![](https://pic.imgdb.cn/item/611539565132923bf87a720c.jpg)

我们可以看到多个线程可以同时访问公共资源count，当某个线程在执行count++的时候，可能其他的线程正好同时也执行count++。但由于多个线程变量count的不可见性，会导致另外的线程拿到旧的count值+1，这样就出现了realCount预计是20，但是实际上是18的数据问题。

再看看示例2中的情况：
![](https://pic.imgdb.cn/item/611539665132923bf87a9654.jpg)

如图所示，往大的方向上说，ThreadLocal会给每一个线程都创建变量的副本，保证每个线程访问都是自己的副本，相互隔离。

往小的方向上说，每个线程内部都有一个threadLocalMap，每个threadLocalMap里面都包含了一个entry数组，而entry是由threadLocal和数据（这里指的是count）组成的。这样一来，每个线程都拥有自己专属的变量count。

示例2中线程1调用calc方法时，会先调用的getCount方法，由于第一次调用threadLocal.get()返回是空的，所以getCount返回值是0。这样threadLocal.set(getCount() + 1);就变成了threadLocal.set(0 + 1);它会给线程1中threadLocal的数据值设置成1。线程2再调用calc方法，同样会先调用getCount方法，由于第一次调用threadLocal.get()返回是空的，所以getCount返回值也是0。

这样threadLocal.set(getCount() + 1);会给线程2中threadLocal的数据值也设置成1。。。。。。最后每个线程的threadLocal中的数据值都是1。

还有，示例2中打印出来的realCount为什么是0呢？

因为testThreadLocal.getCount()是在主线程中调用的，其他的线程改变只会影响自己的副本，不会影响原始变量，count初始值是0，所以最后还是0。

## 3.ThreadLocal源码解析

在介绍ThreadLocal之前，让我们一起先看看Thread类
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```
可以看到Thread类中定义了一个叫threadLocals的成员变量，它的类型是ThreadLocal.ThreadLocalMap。很明显ThreadLocalMap是ThreadLocal的内部类，验证了我在图中画的内容，每个线程都有一个ThreadLocalMap对象。

我们再重点看看ThreadLocalMap
```java
static class ThreadLocalMap {

      static class Entry extends WeakReference<ThreadLocal<?>> {
          /** The value associated with this ThreadLocal. */
          Object value;

          Entry(ThreadLocal<?> k, Object v) {
              super(k);
              value = v;
          }
      }

      /**
       * The initial capacity -- MUST be a power of two.
       */
      private static final int INITIAL_CAPACITY = 16;

      /**
       * The table, resized as necessary.
       * table.length MUST always be a power of two.
       */
      private Entry[] table;

      /**
       * The number of entries in the table.
       */
      private int size = 0;

      /**
       * The next size value at which to resize.
       */
      private int threshold; // Default to 0

      /**
       * Set the resize threshold to maintain at worst a 2/3 load factor.
       */
      private void setThreshold(int len) {
          threshold = len * 2 / 3;
      }

       ........省略
  }
```  
由于该方法太长了，我在这里省略了部分内容。从以上代码可以看到ThreadLocalMap里面包含了一个叫table的数组，它的类型是Entry，Entry是WeakReference（弱引用）的子类，Entry又包含了 ThreadLocal变量 和Object的value ，其中ThreadLocal变量做为WeakReference的referent。

接下来，我们再回到ThreadLocal类，常用的其实就下面四个方法：get()， initialValue()，set(T value) 和 remove()，接下来我们会逐一介绍。

首先看看get()方法
```java
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程中的ThreadLocalMap对象
    ThreadLocalMap map = getMap(t);
    //如果可以查询到数据
    if (map != null) {
        //从ThreadLocalMap中获取entry对象
        ThreadLocalMap.Entry e = map.getEntry(this);
        //如果entry存在
        if (e != null) {
            @SuppressWarnings("unchecked")
            //获取entry中的值
            T result = (T)e.value;
            //返回获取到的值
            return result;
        }
    }
    //调用初始化方法
    return setInitialValue();
}
```
其中的getMap方法

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
再简单不过了，直接返回的是当前线程的成员变量threadLocals

再看看getEntry方法
```java
private Entry getEntry(ThreadLocal<?> key) {
    //threadLocalHashCode是key的hash值
    //key.threadLocalHashCode & (table.length - 1)，
    //相当于threadLocalHashCode对table.length - 1的取余操作，
    //这样可以保证数组的下表在0到table.length - 1之间。
    int i = key.threadLocalHashCode & (table.length - 1);
    //获取下标对应的entry
    Entry e = table[i];
    //如果entry不为空，并且从弱引用中获取到的值(threadLocal) 和 key相同 
    if (e != null && e.get() == key)
        //返回获取到的entry
        return e;
    else
       //如果没有获取到entry或者e.get()获取不到数据，则清理空数据
        return getEntryAfterMiss(key, i, e);
}
我之前说过entry是WeakReference的子类，那么e.get()方法会调用：

 public T get() {
      return this.referent;
  }
``  
返回的是一个引用，这个引用就是构造器传入的threadLocal对象。

getEntryAfterMiss里面有说明逻辑呢？
```java
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
该方法里面会调用expungeStaleEntry方法，后面我们会重点介绍的。

再看看setInitialValue方法
```java
protected T initialValue() {

return null;  
}

private T setInitialValue() {
    //调用用户自定义的initialValue方法，默认值是null
    T value = initialValue();
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程中的ThreadLocalMap，跟之前一样
    ThreadLocalMap map = getMap(t);
    //如果ThreadLocalMap不为空，
    if (map != null)
        //则覆盖key为当前threadLocal的值
        map.set(this, value);
    else
       //否则创建新的ThreadLocalMap
        createMap(t, value);
    //返回用户自定义的值    
    return value;
}
```
当中的initialValue()方法，就是我们要介绍的第二个方法
```java
  protected T initialValue() {
      return null;
  }
```  
我们可以看到该方法只有一个空实现，等着用户的子类重写之后重新实现。

接下来重点看看threadLocalMap的set方法
```java
private void set(ThreadLocal<?> key, Object value) {
    //将table数组赋值给新数组tab
    Entry[] tab = table;
    //获取数组长度
    int len = tab.length;
    //跟之前一样计算数组中的下表
    int i = key.threadLocalHashCode & (len-1);

    //循环变量tab获取entry
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        //获取entry中的threadLocal对象 
        ThreadLocal<?> k = e.get();
        //如果threadLocal对象不为空，并且等于key
        if (k == key) {
            //覆盖已有数据
            e.value = value;
            //返回
            return;
        }
        //如果threadLocal对象为空
        if (k == null) {
            //创建一个新的entry赋值给已有key
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    //如果key不在已有数据中，则创建一个新的entry
    tab[i] = new Entry(key, value);
    //长度+1
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
replaceStaleEntry方法也会调用expungeStaleEntry方法。

再看看setInitialValue方法中的createMap方法
```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
代码很简单，就是new了一个ThreadLocalMap对象。

好了，到这来get() 和 initialValue() 方法介绍完了。

下面介绍set(T value) 方法
```java
public void set(T value) {
    //获取当前线程，都是一样的套路
    Thread t = Thread.currentThread();
    //根据当前线程获取当中的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //ThreadLocalMap不为空，则调用之前介绍过的ThreadLocalMap的set方法
    if (map != null)
        map.set(this, value);
    else
       //如果ThreadLocalMap为空，则创建一个对象，之前也介绍过
        createMap(t, value);
}
```
so easy

最后，看看remove()方法
```java
public void remove() {
     //还是那个套路，不过简化了一下
     //先获取当前线程，再获取线程中的ThreadLocalMap对象
     ThreadLocalMap m = getMap(Thread.currentThread());
     //如果ThreadLocalMap不为空
     if (m != null)
         //删除数据
         m.remove(this);
 }
``` 
这个方法的关键就在于ThreadLocalMap类的remove方法
```java
private void remove(ThreadLocal<?> key) {
    //将table数组赋值给新数组tab
    Entry[] tab = table;
    //获取数组长度
    int len = tab.length;
    //跟之前一样计算数组中的下表
    int i = key.threadLocalHashCode & (len-1);
    //循环变量从下表i之后不为空的entry
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        //如果可以获取到threadLocal并且值等于key 
        if (e.get() == key) {
            //清空引用
            e.clear();
            //处理threadLocal为空但是value不为空的entry
            expungeStaleEntry(i);
            return;
        }
    }
}
```
其中的clear方法，也很简单，只是把引用设置为null，即清空引用
```java
public void clear() {
    this.referent = null;
}
```
我们可以看到get()、set(T value) 和 remove()方法，都会调用expungeStaleEntry方法，我们接下来重点看一下expungeStaleEntry方法
```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    //将位置staleSlot对应的entry中的value设置为null，有助于垃圾回收
    tab[staleSlot].value = null;
    //将位置staleSlot对应的entry设置为null，有助于垃圾回收
    tab[staleSlot] = null;
    //数组大小-1
    size--;

    Entry e;
    int i;
    //变量staleSlot之后entry不为空的数据
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        //获取当前位置的entry中对应的threadLocal 
        ThreadLocal<?> k = e.get();
        //threadLocal为空，说明是脏数据
        if (k == null) {
            //value设置为null，有助于垃圾回收
            e.value = null;
            //当前位置的entry设置为null
            tab[i] = null;
            //数组大小-1
            size--;
        } else {
            //重新计算位置
            int h = k.threadLocalHashCode & (len - 1);
            //如果h和i不相等，说明存在hash冲突
            //现在它前面的脏Entry被清理
            //该Entry需要向前移动，防止下次get()或set()的时候
            //再次因散列冲突而查找到null值
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
该方法首先清除当前位置的脏Entry，然后向后遍历直到table[i]==null。在遍历的过程中如果再次遇到脏Entry就会清理。如果没有遇到就会重新变量当前遇到的Entry，如果重新散列得到的下标h与当前下标i不一致，说明该Entry被放入Entry数组的时候发生了散列冲突（其位置通过再散列被向后偏移了），现在其前面的脏Entry已经被清除，所以当前Entry应该向前移动，补上空位置。否则下次调用set()或get()方法查找该Entry的时候会查找到位于其之前的null值。

为什么要做这样的清除？

我们知道entry对象里面包含了threadLocal和value，threadLocal是WeakReference（弱引用）的referent。每次垃圾回收期触发GC的时候，都会回收WeakReference的referent，会将referent设置为null。那么table数组中就会存在很多threadLocal = null 但是 value不为空的entry，这种entry的存在是没有任何实际价值的。这种数据通过getEntry是获取不到值，因为它里面有if (e != null && e.get() == key)这句判断。

为什么要使用WeakReference（弱引用）？

如果使用强引用，ThreadLocal在用户进程不再被引用，但是只要线程不结束，在ThreadLocalMap中就还存在引用，无法被GC回收，会导致内存泄漏。如果用户线程耗时非常长，这个问题尤为明显。

另外在使用线程池技术的时候，由于线程不会被销毁，回收之后，下一次又会被重复利用，会导致ThreadLocal无法被释放，最终也会导致内存泄露问题。

## 4.ThreadLocal有哪些坑

### 内存泄露问题：

ThreadLocal即使使用了WeakReference（弱引用）也可能会存在内存泄露问题，因为 entry对象中只把key(即threadLocal对象)设置成了弱引用，但是value值没有。还是会存在下面的强依赖：

> Thread -> ThreaLocalMap -> Entry -> value

要解决这个问题就需要调用get()、set(T value) 或 remove()方法。但是 get()和set(T value) 方法是基于垃圾回收器把key回收之后的基础之上触发的数据清理。如果出现垃圾回收器回收不及时的情况，也一样有问题。

所以，最保险的做法是在使用完threadLocal之后，手动调用一下remove方法，从源码可以看到，该方法会把entry中的key(即threadLocal对象)和value一起清空。

### 线程安全问题：

可能有些朋友认为使用了threadLocal就不会出现线程安全问题了，其实是不对的。假如我们定义了一个static的变量count，多线程的情况下，threadLocal中的value需要修改并设置count的值，它一样有问题。

因为static的变量是多个线程共享的，不会再单独保存副本。

## 5.总结

1. 每个线程都有一个threadLocalMap对象，每个threadLocalMap里面都包含了一个entry数组，而entry是由key（即threadLocal）和value（数据）组成。
2. entry的key是弱引用，可以被垃圾回收器回收。
3. threadLocal最常用的这四个方法：get()， initialValue()，set(T value) 和 remove()，除了initialValue方法，其他的方法都会调用expungeStaleEntry方法做key==null的数据清理工作。
4. threadLocal可能存在内存泄露和线程安全问题，使用完之后，要手动调用remove方法。