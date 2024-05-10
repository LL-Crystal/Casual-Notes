# Java ThreadLocal浅析

1、ThreadLocal概念

线程同步机制是多个线程共享同一个变量，而ThreadLocal是为每个线程创建一个单独的变量副本，每个线程都可以改变自己的变量副本而不影响其它线程所对应的副本。

>官方API上是这样介绍的：该类提供了线程局部(thread-local)变量。
这些变量不同于它们的普通对应物，因为访问某个变量（通过其 get 或 set 方法）的每个线程都有自己的局部变量，它独立于变量的初始化副本。
ThreadLocal实例通常是类中的 private static 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。
---

2、ThreadLocal的API

ThreadLocal定义了四个方法:

- get():返回此线程局部变量当前副本中的值
- set(T value):将线程局部变量当前副本中的值设置为指定值
- initialValue():返回此线程局部变量当前副本中的初始值
- remove():移除此线程局部变量当前副本中的值

>ThreadLocal还有一个特别重要的静态内部类ThreadLocalMap，该类才是实现线程隔离机制的关键。
get()、set()、remove()都是基于该内部类进行操作，ThreadLocalMap用键值对方式存储每个线程变量的副本，key为当前的ThreadLocal对象，value为对应线程的变量副本。
试想，每个线程都有自己的ThreadLocal对象，也就是都有自己的ThreadLocalMap，对自己的ThreadLocalMap操作，当然是互不影响的了，这就不存在线程安全问题了，所以ThreadLocal是以空间来交换安全性的解决思路。
---

3、源码分析

>ThreadLocalMap, ThreadLocalMap内部是利用Entry来进行key-value的存储的。
key就是ThreadLocal，value就是值，Entry继承WeakReference，所以Entry对应key的引用（ThreadLocal实例）是一个弱引用，方便GC回收。
---

```
static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

set(ThreadLocal key, Object value)

>这个set操作和集合Map解决散列冲突的方法不同，集合Map采用的是链地址法，这里采用的是开放定址法（线性探测）。
set()方法中的replaceStaleEntry()和cleanSomeSlots()，这两个方法可以清除掉key ==null的实例，防止内存泄漏。
---

在插入过程中，根据ThreadLocal对象的hash值，定位到table中的位置i，hash冲突解决过程如下：

- 如果当前位置是空的，那么正好，就初始化一个Entry对象放在位置i上；
- 不巧，位置i已经有Entry对象了，如果这个Entry对象的key正好是即将设置的key，那么重新设置Entry中的value；
- 很不巧，位置i的Entry对象，和即将设置的key没关系，那么只能找下一个空位置；这样的话，在get的时候，也会根据ThreadLocal对象的hash值，定位到table中的位置，然后判断该位置Entry对象中的key是否和get的key一致，如果不一致，就判断下一个位置。可以发现，set和get如果冲突严重的话，效率很低，因为ThreadLoalMap是Thread的一个属性，所以即使在自己的代码中控制了设置的元素个数，但还是不能控制其它代码的行为。

```
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    // 根据ThreadLocal的散列值，查找对应元素在数组中的位置
    int i = key.threadLocalHashCode & (len-1);
    // 采用线性探测法寻找合适位置
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // key存在，直接覆盖
        if (k == key) {
            e.value = value;
            return;
        }
        // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    //ThreadLocal对应的key实例不存在，new一个
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 清除陈旧的Entry(key == null的)
    // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

getEntry()

>由于采用了开放定址法，当前keu的散列值和元素在数组中的索引并不是一一对应的，
首先取一个猜测数（key的散列值），如果所对应的key是我们要找的元素，那么直接返回，否则调用getEntryAfterMiss
---

```
private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```

getEntryAfterMiss()

>这里一直在探测寻找下一个元素，知道找的元素的key是我们要找的。
这里当key==null时，调用expungeStaleEntry有利于GC的回收，用于防止内存泄漏。
---

```
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

4、存在问题, ThreadLocal为什么会内存泄漏

![thread_local](image/thread_local.png)

>ThreadLocalMap的key为ThreadLocal实例，他是一个弱引用，我们知道弱引用有利于GC的回收，当key == null时，GC就会回收这部分空间，
但value不一定能被回收，因为他和Current Thread之间还存在一个强引用的关系。
由于这个强引用的关系，会导致value无法回收，如果线程对象不消除这个强引用的关系，就可能会出现OOM。有些时候，我们调用ThreadLocalMap的remove()方法进行显式处理。
通常来讲，ThreadLocal 不会造成什么大问题，因为在 Thread 执行完毕的时候，所有的资源都会被自动释放掉。
唯一的问题就是——线程池, 如果我们在线程池中使用了 ThreadLocal ，由于我们通常无法直接控制线程池线程的创建与释放，也就意味着除非手动清除掉 ThreadLocal 中的资源， 否则，ThreadLocal 持有的资源可能永远无法释放，内存泄漏的隐患由此而生。
---

5、如何避免内存泄露

>既然已经发现有内存泄露的隐患，自然有应对的策略，在调用ThreadLocal的get()、set()可能会清除ThreadLocalMap中key为null的Entry对象，
这样对应的value就没有GC Roots可达了，下次GC的时候就可以被回收，当然如果调用remove方法，肯定会删除对应的Entry对象。
如果使用ThreadLocal的set方法之后，没有显示的调用remove方法，就有可能发生内存泄露，所以养成良好的编程习惯十分重要，使用完ThreadLocal之后，记得调用remove方法。
---

```
ThreadLocal<String> localName = new ThreadLocal();
try {
    localName.set("sdf");
    // 其它业务逻辑
} finally {
    localName.remove();
}
```

6、总结

- ThreadLocal不是用来解决共享变量的问题，也不是协调线程同步，他是为了方便各线程管理自己的状态而引用的一个机制。
- 每个ThreadLocal内部都有一个ThreadLocalMap,他保存的key是ThreadLocal的实例，他的值是当前线程的局部变量的副本的值。

