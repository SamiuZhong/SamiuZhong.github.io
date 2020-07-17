## 概述

1. HashMap的结构包括了初始数组、链表、红黑树
2. 插入数据的时候使用 pos=key%size 来进行插入数据
3. 当两个或两个以上的数据key相同值不同的时候（发生冲突），就会挂在初始化位置的链表后面
4. 当某个节点后出现过多的链表节点的时候，就会转换成红黑树以提高效率

## 源码分析

### 初始化和插入

创建一个HashMap并插入第一条数据

```java
Map<String,String> map = new HashMap<String,String>();
map.put("1000","test01");
```

看看它的构造函数长啥样

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; 
}
```

这里的loadFactor是一个负载因子。

那我们去看看put方法。

看put方法之前首先定义了一个Node节点和节点所组成的数组

```java
transient Node<K,V>[] table;

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    HashMap.Node<K,V> next;

    Node(int hash, K key, V value, HashMap.Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    ...
}
```

然后是put

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    HashMap.Node<K,V>[] tab; HashMap.Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)	//comment 1
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)	//comment 2
        tab[i] = newNode(hash, key, value, null);	//comment 3
    else {
        HashMap.Node<K,V> e; K k;
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))	//comment 4
            e = p;
        else if (p instanceof HashMap.TreeNode)	//comment 5
            e = ((HashMap.TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {	//comment 6
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) {
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

可以看到在comment 1处对tab进行了一个为空的判断，我们创建的时候肯定是为空的，所以就会进入判断，调用一个resize方法。那我们就去看看这个resize方法

```java
final HashMap.Node<K,V>[] resize() {
    HashMap.Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        ...
    }
    else if (oldThr > 0) 
        ...
    else {	//comment 1        
        newCap = DEFAULT_INITIAL_CAPACITY;	//16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);	//12
    }
    if (newThr == 0) {
        ...
    } 
    threshold = newThr;
    HashMap.Node<K,V>[] newTab = (HashMap.Node<K,V>[])new HashMap.Node[newCap];
    table = newTab;
    if (oldTab != null) {
    	...
    }    
    return newTab;
}
```

这里明显会走入comment 1处的判断，所以newCap会被赋值为DEFAULT_INITIAL_CAPACITY，那么这个DEFAULT_INITIAL_CAPACITY又是个啥？

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
```

我们可以看到DEFAULT_INITIAL_CAPACITY的值就是1 << 4，也就是16，那么显然newThr=0.75*16=12。好，然后把newTab返回，也就是长度为16的Node数组。

然后又回到了putVal方法的comment 2。

comment 2这里首先令下标i = (n - 1) & hash这样的一个值，这里的hash为上面put方法传过来的hash(key)，那我们去看看hash方法长什么样子

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这里的一顿操纵主要就是为了提高散列度。

继续回到上面的comment 2，判断这个节点为空的话就会进comment 3新建一个节点；

否则就进入else，在comment 4处判断是否key和hash值都相同；

不同的话现在comment 5判断是否是一个树节点，是的话添加进红黑树；

否则进comment 6挂载在链表后面。

### 扩容机制