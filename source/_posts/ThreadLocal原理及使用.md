---
title: ThreadLocal原理及使用
date: 2017-08-21 17:11:10
tags:
    - ThreadLocal
categories:
    - Java并发
---
近期看Java并发编程，总是会提到ThreadLocal，介绍时只是轻描淡写的说可以理解为一个< Thread, Value>的Map，但实际上并不是这样实现的，于是就一直想看看ThreadLocal在JDK中到底是如何实现的。大概看了一下源码中一些比较重要的类和方法，基本上了解了实现原理。
<!--more-->
## 如何使用
ThreadLocal的初衷是将线程所要使用的局部变量由线程自己保存，同时在读取和写入上又比较方便。想要理解ThreadLocal的作用，不妨先假设如果没有ThreadLocal，我们如何实现上述的功能。

直接在线程中定义一个我们需要的变量是不可取的。因为在多线程的环境下这个变量是一个共享变量。那么还有一种方法是将变量定义在方法内部，这样变量就会在栈中被分配，而栈帧是线程私有的，可以做到只有自己读取到自己的局部变量，但是这样带来的问题是方法之间传递参数会变得很麻烦，因为要通过参数将变量传递过去。所以Java实现了ThreadLocal来优雅的实现这个功能。

先用一个例子看一下ThreadLocal是如何使用的：
```
public class HostHolder {
    //在多线程环境下，每个线程对应一个用户，因此将用户的信息保存在ThreadLocal中
    private static ThreadLocal<User> users = new ThreadLocal<>();

    public User getUser() {
        return users.get();
    }

    public void setUser(User user) {
        users.set(user);
    }

    public void clear() {
        users.remove();
    }
}
```

ThreadLocal常用的几个方法：
```
//获取ThreadLocal中存储的变量
get()
//将想要保存的值放入ThreadLocal中
set()
//清除保存的变量
remove()
//如果没有调用set()方法保存值，那么返回一个初始值，可以通过继承覆写此方法定义自己的初始值函数
initialValue()
```
知道这些函数，基本就可以使用ThreadLocal来进行编程了，接下来我们根据[此处](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8-b132/java/lang/ThreadLocal.java#ThreadLocal)的源码分析其实现。

## 实现原理
我们按照其方法来依次深入分析，先看其get()方法：
```
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    //根据当前线程获取对应map
    ThreadLocalMap map = getMap(t);
    //如果map存在，且查找的key对应的Entry存在，则查找其value
    if (map != null) {
        //注意此处我们的key使用的是this,也就是当前的ThreadLocal引用
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T) e.value;
            return result;
        }
    }
    //否则调用该方法返回一个初始值
    return setInitialValue();
}
```
代码所执行的逻辑看注释可基本了解，通过代码我们可以知道ThreadLocal中的值都是保存在一个ThreadLocalMap中的，接下来再看一下getMap(Thread t)函数的实现：
```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
这个函数相当简单，即返回线程的成员变量threadLocals，查看Thread类可以看到threadLocals是一个ThreadMap，默认为null。那么说明我们初始化一个ThreadLocalMap的代码就在setInitialValue()方法中了：
```
private T setInitialValue() {
    //获得一个初始值作为默认值
    T value = initialValue();
    //依旧是根据线程t获取map
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        //获取失败，则creatMap
        createMap(t, value);
    return value;
}
```
createMap中创建一个ThreadLocalMap很简单，就是调用其构造函数并赋值给成员变量threadLocals：
```
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
至此，我们再来看一下ThreadLocalMap究竟是如何实现的：
```
static class  ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private static final int INITIAL_CAPACITY = 16;
    private Entry[] table;
    private int size = 0;
    private int threshold; // Default to 0
    
    //构造函数1
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
    
    //构造函数2
    private   ThreadLocalMap(ThreadLocalMap parentMap) {
        Entry[] parentTable = parentMap.table;
        int len = parentTable.length;
        setThreshold(len);
        table = new Entry[len];

        for (int j = 0; j < len; j++) {
            Entry e = parentTable[j];
            if (e != null) {
                @SuppressWarnings("unchecked")
                ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                if (key != null) {
                    Object value = key.childValue(e.value);
                    Entry c = new Entry(key, value);
                    int h = key.threadLocalHashCode & (len - 1);
                    while (table[h] != null)
                        h = nextIndex(h, len);
                    table[h] = c;
                    size++;
                }
            }
        }
    }
    /*
    一些内部方法
    */
}
```
通过上面的代码，我们知道此Map内定义了一个Entry类，而这个类就是一个K-V实体。其中Key是ThreadLocal，Value就是我们所要保存的值；通过持有一个Entry数组来实现变量的保存。

通过代码我们还知道，Map的默认大小是16，变量保存的位置是通过与运算得出的。ThreadLocalMap有两个构造函数，通过第二个构造函数我们可以知道一个线程的ThreadLocal变量其子线程也是可以读取到的，实现的方法就是将父线程中的Map都复制到自己的Map中来。

同时根据其他的代码我们可以知道ThreadLocalMap是使用线性探查法来保存元素的，即如果哈希冲突时接着线性查找下一个是否有空间，直到插入成功。鉴于篇幅，就不分析ThreadLocalMap中的其他方法了。

还有set()方法源码也贴一下：
```
public void  set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
和get()方法几乎相同，不再赘述。

总结一下，当我们使用ThreadLocal时，首先获取到现在的线程，在线程中放着一个ThreadLocalMap，我们的数据都以 < ThreadLocal, Value> 的方式存储在其中，为了节约内存空间，我们没有使用拉链法而是使用了线性探查法，保证空间利用率较高，查找效率可能略差于拉链法。
