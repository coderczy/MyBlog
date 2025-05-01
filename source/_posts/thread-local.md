---
title: 一文彻底搞懂ThreadLocal
date: 2025-05-01 00:06:47
tags: JAVA多线程
---
# 一文彻底搞懂ThreadLocal
## 前言
&emsp;&emsp;**ThreadLocal**里的变量是线程之间隔离的，但在线程内部是处处可以访问的。ThreadLocal相当于是给变量在每个线程中创建了独立的副本，每个线程可以访问自己内部的副本。

&emsp;&emsp;作为一名职场小白，看到项目代码中有好多地方都使用了ThreadLocal来传递上下文，心想之前背了好多关于ThreadLocal的八股，直接使用ThreadLocal传递参数，省的定义一堆方法入参逐层向下传递，不用白不用。但是，后来听组里大佬说不建议在业务代码中使用ThreadLocal，甚至应该禁止，因为使用不当会带来很严重的问题。回头再看看自己之前写的代码，貌似还真有问题（本文就不细说了）。。。

&emsp;&emsp;总之，今天决定写一篇关于ThreadLocal的文章，希望自己在写的过程中能彻底搞懂ThreadLocal，搞清楚它的使用会引发哪些风险。

## 开门见山
&emsp;&emsp;在网上找了很多讲解文章来对以前的八股进行回忆，在这开门见山的说明ThreadLocal的使用不当会带来两个严重问题： **内存泄露**和**数据污染**。为什么开门见山的说明，就是要把这两大风险记在脑子里，以免下次再滥用ThreadLocal。

## 庖丁解牛
&emsp;&emsp;接下来，我们分别详细分析一下ThreadLocal什么场景下会造成内存泄露和数据污染的问题。

### 1. 源码窥探
&emsp;&emsp;我想在揭露这些问题之前，我们还需要回顾一下ThreadLocal的内部原理。二话不说直接上源码，先看set方法。
```JAVA
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```
&emsp;&emsp;从set方法实现来看，首先会获取当前线程，接着通过当前线程获取一个ThreadLocalMap，获取到ThreadLocalMap的话直接将变量value放进map，对应的key即为当前ThreadLocal对象。没拿到ThreadLocalMap的话就新创建一个map。这里的关键问题ThreadLocalMap是什么样的map？和当前线程是什么关系？。我们先看看getMap和createMap方法。
```JAVA
/**
 * Get the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param  t the current thread
 * @return the map
 */
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

/**
 * Create the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param t the current thread
 * @param firstValue value for the initial entry of the map
 */
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
&emsp;&emsp;可以看到获取的ThreadLocalMap就是当前线程t的一个属性：threadLocals。这里我们就不难理解什么叫每个线程独有的副本了，因为它就是和每个线程相关联的属性。接着，我们看看ThreadLocalMap的数据结构：
```JAVA
/**
 * ThreadLocalMap is a customized hash map suitable only for
 * maintaining thread local values. No operations are exported
 * outside of the ThreadLocal class. The class is package private to
 * allow declaration of fields in class Thread.  To help deal with
 * very large and long-lived usages, the hash table entries use
 * WeakReferences for keys. However, since reference queues are not
 * used, stale entries are guaranteed to be removed only when
 * the table starts running out of space.
 */
static class ThreadLocalMap {

    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private Entry[] table;

    // .....................................
    /**
     * Construct a new map without a table.
     */
    private ThreadLocalMap() {
    }

    /**
     * Construct a new map initially containing (firstKey, firstValue).
     * ThreadLocalMaps are constructed lazily, so we only create
     * one when we have at least one entry to put in it.
     */
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }

    //...........................................................
}
```
&emsp;&emsp;可以看到ThreadLocalMap是定义在ThreadLocal里的一个静态内部类，只能java.lang包内可以访问，主要就是给java.lang.Thread使用。ThreadLocalMap中又定义了自己的Entry类型，它继承了**WeakReference（弱引用）**，Entry中有value属性。从构造方法来看，ThreadLocalMap的key采用了弱引用，这个很关键。
### 2. 弱引用玄机
&emsp;&emsp;ThreadLocalMap的key为什么采用弱引用呢。首先我们知道弱引用的特性就是：发生gc的时候，当一个对象没有在被其他强引用引用时，它就会被回收。为了更加清晰的说明这个问题，我们画一个引用关系图：
{% asset_img reference.png ThreadLocal引用关系图 %}

&emsp;&emsp;从图中我们可以看出，只要当前线程Thread不消亡，它就会一直持有ThreadLocalMap对象的引用。如果B失去对ThreadLocal对象的引用时，如果ThreadLocalMap的key是强引用的话，该ThreadLocal对象将一直无法回收，除非当前线程Thread被销毁。Thread如果是主线程，那ThreadLocal对象将持续整改运行周期，如果是线程池中的线程，短时间内也无法消亡。从而就会引发内存泄露。因此ThreadLocalMap的key采用弱引用就避免了这一问题。

### 3. 内存泄露
&emsp;&emsp;key为弱引用就可以高枕无忧了吗？实际上还会有内存泄露的风险。B失去了对ThreadLocal对象的引用后，在gc的过程中ThreadLocal对象被回收，此时对应ThreadLocalMap中Entry的key就变成了null值，但value中的变量依旧被强引用，无法被回收。由于key已经是null了，业务代码无法访问到value变量，这就出现了废弃变量无法回收的情况，从而导致内存泄露。如何彻底避免内存泄露呢？

&emsp;&emsp;将ThreadLocal变量定义成private static，这样就一直存在ThreadLocal的强引用，也就能保证任何时候都能通过ThreadLocal的弱引用访问到Entry的value值。且能保证ThreadLocal实例唯一，不会每次请求都创建一个新的ThreadLocal实例，这样就不会对同一线程变量反复往ThreadLocalMap中塞Entry。

### 4. 数据污染
&emsp;&emsp;这样就够了吗？不够的，这样虽然在一定程度上避免了内存泄露问题，但是还会有一个数据污染的问题。现在的项目在处理请求时一般都会用到线程池，线程池提供了线程复用，减少线程上创建销毁、下文切换的开销。如果第一次请求往线程中塞了一个数据变量，请求结束时没有清理，同时对应的处理线程没有销毁，而是直接放回了线程池。下一次有请求复用了该线程，它依然能够访问到上一次请求塞到ThreadLocalMap中的数据变量，这往往是不符合预期的，也就是我们说的数据污染问题。

&emsp;&emsp;所以呢，每次使用完ThreadLocal都调用它的remove()方法清除数据。

## 最终的建议
&emsp;&emsp;由于ThreadLocal的使用不当会导致严重的系统问题，为了降低风险，我们建议在基建层面可以适当使用ThreadLocal，而在业务层面代码中就不要使用ThreadLocal了，直接定义context对象，将参数层层传递即可，虽然麻烦，但是安全。如果一定要使用则切记如下两点：
1. 每次使用完ThreadLocal都调用它的remove()方法清除数据（放在finally代码块中，保证一定能执行到）。
2. 将ThreadLocal变量定义成private static，这样就一直存在ThreadLocal的强引用，也就能保证任何时候都能通过ThreadLocal的弱引用访问到Entry的value值，进而清除掉 。

