---
layout: post
title: Java ThreadLocal类
category: 学习
tags: study
keywords:
description:
---

### ThreadLocal
用来支持 _*线程本地*_ 变量, 即每一个线程都拥有它自己的一个变量,线程相互之间不能相互访问.

### ThreadLocal对象和Thread线程的关系

![](/assets/picture/2016-08-08_threadLocal.png)

每个线程所持有的所有线程本地变量,其实都是放在当前线程中的((ThreadLocal.ThreadLocalMap)Thread.threadLocals).而ThreadLocal对象就像一个包装器, 只是负责更新或删除当前线程中的变量. 从Tread.threadLocals中获取或者删除本地变量时,是直接以当前ThreadLocal对象作为key进行操作的.

在Thread.threadLocals中, `ThreadLocal`对象的HashCode由上一个`ThreadLocal`对象的HashCode值加上 0x61c88647 , 据说这样可生成最优散列值.....

> The difference between successively generated hash codes - turns implicit sequential thread-local IDs into near-optimally spread multiplicative hash values for power-of-two-sized tables.

> `private static final int HASH_INCREMENT = 0x61c88647`;

### 本地 变量set过程

        public void set(T value) {
          // 获取当前线程所持有的所有本地变量集合,若集合为空,则新建
                Thread t = Thread.currentThread();
                ThreadLocalMap map = getMap(t);
                if (map != null)
                    map.set(this, value);
                else
                    createMap(t, value);
            }

ThreadLocalMap中使用 _*线性探测法*_ 解决Hash冲突.
在map.set方法中,若ThreadLocal对象已在hash表中,则覆盖,否则找到第一个为空的槽位,放置对象.

疑问:看循环逻辑中并未体现出对循环次数的限制, 那难道就不会出现hash表已满的情况导致for循环无 法退出么?

答案是,hash表的默认扩容策略为超过hash表容量的2/3,则进行扩容,所以hash表中一定会存在空的槽位,不会出现循环无法退出的情况.

在做扩容前,还会先剔除ThreadLocal对象已被垃圾回收的entity(ThreadLocal作为key,并且继承自WeakReference).而且策略为只扫描 log2(table.length)-1 次.看注释说这样做不仅简单,快速,而且据说效果不错. :-)

>@param n scan control: <tt>log2(n)</tt> cells are scanned,
          unless a stale entry is found, in which case
         <tt>log2(table.length)-1</tt> additional cells are scanned.
          When called from insertions, this parameter is the number
          of elements, but when from replaceStaleEntry, it is the
          table length. (Note: all this could be changed to be either
          more or less aggressive by weighting n instead of just
          using straight log n. But this version is simple, fast, and
          seems to work well.)

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


### get过程

        public T get() {
                Thread t = Thread.currentThread();
                ThreadLocalMap map = getMap(t);
                if (map != null) {
                    ThreadLocalMap.Entry e = map.getEntry(this);
                    if (e != null)
                        return (T)e.value;
                }
                return setInitialValue();
            }

和set()一样,也是先获取当前线程所持有的所有本地变量,然后获取当前ThreadLocal对象在这个线程中的本地变量.
如果Thread的本地变量集合为空,会有一个初始化的过程,在setInitialValue()中会为当前线程新建一个集合,并调用
`protected T initialValue()`获取当前ThreadLocal的初始值(声明为protected,可由子类指定初始值).

        private T setInitialValue() {
                T value = initialValue();
                Thread t = Thread.currentThread();
                ThreadLocalMap map = getMap(t);
                if (map != null)
                    map.set(this, value);
                else
                    createMap(t, value);
                return value;
            }

### InheritableThreadLocal
`InheritableThreadLocal`和`ThreadLocal`类似,也是保存线程本地变量的, 差别是 `InheritableThreadLocal`中的变量可从父线程中继承过来, 即在创建新线程时,会将当前线程中的`InheritableThreadLocal`中的所有变量,复制一份到新创建线程的 `inheritableThreadLocals`中去.所以如果修改父线程中的变量,是不影响子线程中的本地变量的复制代码如下:


        /**
                 * Construct a new map including all Inheritable ThreadLocals
                 * from given parent map. Called only by createInheritedMap.
                 *
                 * @param parentMap the map associated with parent thread.
                 */
                private ThreadLocalMap(ThreadLocalMap parentMap) {
                    Entry[] parentTable = parentMap.table;
                    int len = parentTable.length;
                    setThreshold(len);
                    table = new Entry[len];

                    for (int j = 0; j < len; j++) {
                        Entry e = parentTable[j];
                        if (e != null) {
                            ThreadLocal key = e.get();
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
