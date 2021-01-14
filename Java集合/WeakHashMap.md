# WeakHashMap

- WeakHashMap 继承于AbstractMap，实现了Map接口。
- 和HashMap一样，WeakHashMap 也是一个散列表，它存储的内容也是键值对(key-value)映射，而且键和值都可以是null。
- 不过WeakHashMap的键是“弱键”。在 WeakHashMap 中，当系统发起GC后，key就会被回收了。更精确地说，对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的丢弃，这就使该键成为可终止的，被终止，然后被回收。某个键被终止时，它对应的键值对也就从映射中有效地移除了。

实现步骤：

1. 新建WeakHashMap，将“键值对”添加到WeakHashMap中。
   实际上，WeakHashMap是通过数组table保存Entry(键值对)；每一个Entry实际上是一个单向链表，即Entry是键值对链表。
2. 当某“弱键”不再被其它对象引用，并被GC回收时。在GC回收该“弱键”时，这个“弱键”也同时会被添加到ReferenceQueue(queue)队列中。
3. 当下一次我们需要操作WeakHashMap时，会先同步table和queue。table中保存了全部的键值对，而queue中保存被GC回收的键值对；同步它们，就是删除table中被GC回收的键值对。
   这就是“弱键”如何被自动从WeakHashMap中删除的步骤了。

### 源码分析

### Entry的定义

```java
//Entry里面没看到K的定义哦，跑哪里去了？
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
}

/**
key跑哪里去了？
看super(key,queue);
 |
 |  
 V
--------------- 
WeakReference.java里面哦。 */ 
public WeakReference(T referent) {
        super(referent);
    }
/**
看super(referent);    
---------------
 |
 |  
 V  
---------------

Reference.java里面哦。   */
private T referent;    
Reference(T referent) {
        this(referent, null);
    }
```

解说：可以看到key直接被定义为弱引用了。
定义为弱引用之后，GC就可以随意回收它了。呵呵

### expungeStaleEntries的定义

expungeStaleEntries监控queue队列，当key被回收时，queue里面会接受到GC发送过来的回收消息，expungeStaleEntries这个监控到queue不为空时，就把对应的value也置为空，这样value也就可以被回收了。

WeakHashMap和HashMap基本一样，所以不细看其他部分的源码，主要是这个expungeStaleEntries方法里面。

```java
//WeakHashMap里面定义了一个成员变量queue，GC在回收key之后会发送通知，key被回收了。发送的通知会放到queue。
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

//这个方法是干什么的呢？
//我们上面说了key是弱引用，但是value是强引用啊，所以这个方法的作用是回收value。
//如果key被回收了，则将value=null，这样就可以让GC把value也回收了。
private void expungeStaleEntries() {
        //queue里面的数据时GC主动往里面添加的，所以在WeakHashMap代码里面我们是看不到queue的赋值的。
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```

本文转自：<https://blog.csdn.net/piaoslowly/article/details/81909686>