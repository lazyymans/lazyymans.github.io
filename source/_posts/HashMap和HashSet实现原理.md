---
title: HashMap和HashSet实现原理
date: 2018-05-03 16:50:57
tags: HashMap HashSet
categories: 源码分析
---

## HashMap分析

#### 前言HashMap的数据结构：

`HashMap`的数据结构是：数组 + 单向链表 的形式。

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/hash1.png?raw=true)

#### 1、我们先从`new HashMap();`这里开始分析HashMap。进入源码分析

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    private static final long serialVersionUID = 362498820763181265L;

    /**
     * The default initial capacity - MUST be a power of two.
     * 初始化时默认的HashMap容量, 必须为2的次幂值。
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     * 默认的HashMap最大的容量, 用以限制参数及表格最大容量, 必须是<= 1<<30的2的次幂。
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     * 默认加载因子, 在构造的时候没有给定的情况下
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     * 由链表转换成树的阈值TREEIFY_THRESHOLD（链表转树）
     * 一个桶中bin（箱子）的存储方式由链表转换成树的阈值。即当桶中bin的数量超过TREEIFY_THRESHOLD时使用树来代替链表。默认值是8 
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     * 由树转换成链表的阈值UNTREEIFY_THRESHOLD（树转链表）
     * 当执行resize操作时，当桶中bin的数量少于UNTREEIFY_THRESHOLD时使用链表来代替树。默认值是6 
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     * 当桶中的bin被树化时最小的hash表容量。（如果没有达到这个阈值，即hash表容量小于MIN_TREEIFY_CAPACITY，当桶中bin的数量太多时会执行resize扩容操作）这个MIN_TREEIFY_CAPACITY的值至少是TREEIFY_THRESHOLD的4倍。
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    /**
     * The next size value at which to resize (capacity * load factor).
     * 表示当HashMap的size大于threshold时会执行resize操作
     * @serial
     */
    int threshold;
    
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     * HashMap 存放的Node<K,V>[] 
     */
    transient Node<K,V>[] table;
}
```

这里我们来看一下 `HashMap`的四个构造方法

```java
/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity 	初始容量值
 * @param  loadFactor      the load factor			加载因子值
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
  //如果传入的容量大小小于0，抛出IllegalArgumentException 异常
  if (initialCapacity < 0)
    throw new IllegalArgumentException("Illegal initial capacity: " +
                                       initialCapacity);
  //如果传入的容量大小大于最大的容量，则修改成最大的容量
  if (initialCapacity > MAXIMUM_CAPACITY)
    initialCapacity = MAXIMUM_CAPACITY;
  //加载因子，小于0 或者 不是Float，抛出IllegalArgumentException 异常
  if (loadFactor <= 0 || Float.isNaN(loadFactor))
    throw new IllegalArgumentException("Illegal load factor: " +
                                       loadFactor);
  this.loadFactor = loadFactor;
  this.threshold = tableSizeFor(initialCapacity);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and the default load factor (0.75).
 *
 * @param  initialCapacity the initial capacity.	初始容量值
 * @throws IllegalArgumentException if the initial capacity is negative.
 */
public HashMap(int initialCapacity) {
  this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 */
public HashMap() {
  this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/**
 * Constructs a new <tt>HashMap</tt> with the same mappings as the
 * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
 * default load factor (0.75) and an initial capacity sufficient to
 * hold the mappings in the specified <tt>Map</tt>.
 *
 * @param   m the map whose mappings are to be placed in this map
 * @throws  NullPointerException if the specified map is null
 */
public HashMap(Map<? extends K, ? extends V> m) {
  this.loadFactor = DEFAULT_LOAD_FACTOR;
  putMapEntries(m, false);
}
```

我们重点关注前面三个，通常我们直接用的是`new HashMap();`这一个构造器。可以看到这个构造器只有初始化loadFactor（加载因子） 这个变量。在来看 `HashMap(int initialCapacity)`，此构造函数调用` HashMap(int initialCapacity, float loadFactor)`这一个构造函数，我们来看一下这个构造函数的最后一行代码，看下他的实现`this.threshold = tableSizeFor(initialCapacity);`，这里也就是设置`HashMap`扩容的闸阀。当后续操作不断的增加`HashMap`的容量时，超过`this.threshold`值时，`HashMap`进行扩容。

```java
/**
 * Returns a power of two size for the given target capacity.
 * 输出不小于cap的第一个2的n次幂，作为threshold，比如cap = 3，就变成了4
 */
static final int tableSizeFor(int cap) {
  int n = cap - 1;
  n |= n >>> 1;
  n |= n >>> 2;
  n |= n >>> 4;
  n |= n >>> 8;
  n |= n >>> 16;
  return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

#### 2、`HashMap`存放值

我们再来看`HashMap.put()`方法，跟着里面的注释进行分析

```java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
 *         (A <tt>null</tt> return can also indicate that the map
 *         previously associated <tt>null</tt> with <tt>key</tt>.)
 */
public V put(K key, V value) {
  //hash(key) 做hash处理，得到key对应的hash值
  return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
  int h;
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

/**
 * Implements Map.put and related methods
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
  //这里table 通常情况下第一次put值时为null，除非在创建HashMap 的时候使用的是 HashMap(Map<? extends K, ? extends V> m)这一个构造器。
  if ((tab = table) == null || (n = tab.length) == 0)
    //初始化table 或者给table 扩容
    n = (tab = resize()).length;
  //i = (n - 1) & hash：根据数组的大小，以及key 的hash 值，确定放入在tab数组中的位置。
  //判断tab数组中，该位置是否为空。 
  //(n - 1) & hash 决定当前的key 在tab 中的位置，这里也是为什么我们的容量必须是2的幂 (n - 1) & hash 相当于 hash % (n-1) ，得到的值必定在 0 - (n-1) 之间，n又等于tab[] 的size，(n - 1) & hash运算效率又极其之高。
  if ((p = tab[i = (n - 1) & hash]) == null)
    //为空，创建一个新的Node<key, value>，并赋值给tab[i]
    tab[i] = newNode(hash, key, value, null);
  else {
    //不为空处理
    Node<K,V> e; K k;
    //判断tab[i] 中 key 是否与put 进来的key 相同，这里使用hash和equals来进行比对
    if (p.hash == hash &&
        ((k = p.key) == key || (key != null && key.equals(k))))
      //相同的情况下将p(tab[i]) 赋值给 e 即 tab[i]->p e->p
      e = p;
    else if (p instanceof TreeNode)
      //tab[i] 中 key 是与put 进来的key 不相同，且此时我们的Node 链表转化为 TreeNode链表了
      e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
    else {
      //tab[i] 中 key 是与put 进来的key 不相同
      for (int binCount = 0; ; ++binCount) {
        //将p(tab[i])的下一个节点Node 赋值给e，判断e是不是为null
        if ((e = p.next) == null) {
          //e == null 创建Node，将p(tab[i])指向新的Node（put 进来的key、value），并且结束循环
          p.next = newNode(hash, key, value, null);
          //如果此时新加入的元素，造成了链表过大，将链表Node 转化为 TreeNode
          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
          break;
        }
        //判断e(p.next)的 key 是与put 进来的key 相同，相同则结束循环，不同将p -> e，再次循环判断
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          break;
        // p -> e
        p = e;
      }
    }
    //判断e != null，即此时put 的值，在原来的tab 中已经存在，这里也就是将新的value 替换原有的value。
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
    //tab 扩容
    resize();
  afterNodeInsertion(evict);
  return null;
}

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
  if (oldTab != null) {
    //将旧表中的数据复制给新表中
    for (int j = 0; j < oldCap; ++j) {
      Node<K,V> e;
      if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        if (e.next == null)
          newTab[e.hash & (newCap - 1)] = e;
        else if (e instanceof TreeNode)
          ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
        else { // preserve order
          Node<K,V> loHead = null, loTail = null;
          Node<K,V> hiHead = null, hiTail = null;
          Node<K,V> next;
          do {
            next = e.next;
            if ((e.hash & oldCap) == 0) {
              if (loTail == null)
                loHead = e;
              else
                loTail.next = e;
              loTail = e;
            }
            else {
              if (hiTail == null)
                hiHead = e;
              else
                hiTail.next = e;
              hiTail = e;
            }
          } while ((e = next) != null);
          if (loTail != null) {
            loTail.next = null;
            newTab[j] = loHead;
          }
          if (hiTail != null) {
            hiTail.next = null;
            newTab[j + oldCap] = hiHead;
          }
        }
      }
    }
  }
  return newTab;
}
```

可以看到，这里的分析的很多，那我们中点来看这一段代码：

```java
//不为空处理
Node<K,V> e; K k;
//判断tab[i] 中 key 是否与put 进来的key 相同，这里使用hash和equals来进行比对
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
  //相同的情况下将p(tab[i]) 赋值给 e 即 tab[i]->p e->p
  e = p;
else if (p instanceof TreeNode)
  //tab[i] 中 key 是与put 进来的key 不相同，且此时我们的Node 链表转化为 TreeNode链表了
  e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
else {
  //tab[i] 中 key 是与put 进来的key 不相同
  for (int binCount = 0; ; ++binCount) {
    //将p(tab[i])的下一个节点Node 赋值给e，判断e是不是为null
    if ((e = p.next) == null) {
      //e == null 创建Node，将p.next(tab[i].next)指向新的Node（put 进来的key、value），并且结束循环，这里其实就是把p.next(tab[i].next)指向新的Node
      p.next = newNode(hash, key, value, null);
      //如果此时新加入的元素，造成了链表过大，将链表Node 转化为 TreeNode
      if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
        treeifyBin(tab, hash);
      break;
    }
    //判断e(p.next)的 key 是与put 进来的key 相同，相同则结束循环，不同将p -> e，再次循环判断
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
      break;
    // p -> e
    p = e;
  }
  
  if (e != null) { // existing mapping for key
    V oldValue = e.value;
    if (!onlyIfAbsent || oldValue == null)
      e.value = value;
    afterNodeAccess(e);
    return oldValue;
  }
```

p -> tab[i]，我们put 一个值到tab[i] 位置上，此时tab[i] 已经存在值。

当我们put 一个值时，首先会判断 put 的 key 是否与tab[i] 中的key 相同。相同 e -> p，跳出`if else`，向下执行。不相同，判断当前的p 是否为 TreeNode类型，不为TreeNode 类型，执行`else`，进入`for`循环。重点来了。

1. `if ((e = p.next) == null)`首先将`p.next`赋给`e`并判断是否为null，如果为空（此时tab[i]中，有且仅有一个Node），创建一个新的Node 赋值给`p.next`，在判断此时 tab[i]链表中的Node 数量是否超过闸阀 TREEIFY_THRESHOLD，超过则把Node类型  转换为 TreeNode类型。跳出循环。（因为`e = p.next`为null，所以`if (e != null) `下面不再执行）

2. `if ((e = p.next) == null)`首先将`p.next`赋给`e`并判断是否为null，如果不为空（此时tab[i]中，存在2个以上Node或TreeNode），我们看`if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))`，因为我们之前只检验了我们put 进来的值 与tab[i] 上的这个值是否相同，那后面的链表中，所有的值都要进行检验是否相同。如果相同，直接跳出循环，进入`if (e != null) `继续执行。将新put 进来的 value 替换 oldValue。如果校验不相同，则`p = e`，继续进行循环，如果所有的Node 的key 与新put 的key 都不同的话，则把新put  的值放在链表结尾。


当put 完之后，进行判断`if (++size > threshold)`，是否需要对HashMap 进行扩容。

#### 3、`HashMap`取值

我们再来看`HashMap.get()`方法，跟着里面的注释进行分析

```java
/**
 * Returns the value to which the specified key is mapped,
 * or {@code null} if this map contains no mapping for the key.
 *
 * <p>More formally, if this map contains a mapping from a key
 * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
 * key.equals(k))}, then this method returns {@code v}; otherwise
 * it returns {@code null}.  (There can be at most one such mapping.)
 *
 * <p>A return value of {@code null} does not <i>necessarily</i>
 * indicate that the map contains no mapping for the key; it's also
 * possible that the map explicitly maps the key to {@code null}.
 * The {@link #containsKey containsKey} operation may be used to
 * distinguish these two cases.
 *
 * @see #put(Object, Object)
 */
public V get(Object key) {
  Node<K,V> e;
  return (e = getNode(hash(key), key)) == null ? null : e.value;
}

/**
 * Implements Map.get and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @return the node, or null if none
 */
final Node<K,V> getNode(int hash, Object key) {
  Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
  //判断 table 是否为 null，或者table.length 是不是大于0，并且tab[(n - 1) & hash] 对应的值不能为null
  if ((tab = table) != null && (n = tab.length) > 0 &&
      (first = tab[(n - 1) & hash]) != null) {
    //判断第一个 Node(first) 的 key 是否与 传入的 key 相同，相同直接返回 first
    if (first.hash == hash && // always check first node
        ((k = first.key) == key || (key != null && key.equals(k))))
      return first;
    //判断下一个Node 是否为 null
    if ((e = first.next) != null) {
      //判断是否为TreeNode 类型
      if (first instanceof TreeNode)
        return ((TreeNode<K,V>)first).getTreeNode(hash, key);
      do {
        //执行循环 判断当前的Node(e) 的 key 是否与 传入的 key 相同，相同直接返回 e，不相同，判断(e = e.next) != null。继续执行，如果最后没有找到，直接返回null
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          return e;
      } while ((e = e.next) != null);
    }
  }
  return null;
}
```

这里我们看以看到`HashMap.get()`的思路跟`HashMap.put()`的思路是一样的。都是通过比较 key 的 hash值 和 equals 来看 可以是否 key 相同。首先找到key 的hash 在table[] 中的位置，匹对table[i] 的key 是否与 get 的 key相同，不相同，寻找table[i] 的下一个节点Node，对整个链表进行遍历。找到则取出，没有找到继续遍历，直到整个链表遍历完成，若遍历完成之后还未找到，则返回null。

#### 4、HashMap的工作原理

HashMap是基于hashing的原理，我们使用put(key, value)存储对象到HashMap中，使用get(key)从HashMap中获取对象。当我们给put()方法传递键和值时，我们先对键调用hashCode()方法，返回的hashCode用于找到table位置来储存Entry对象。 

```java
//put 中加入新Node 
newNode(hash, key, value, null);

Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

#### 5、HashMap注意事项

1. 减少"碰撞"发生，经过我们的源码分析。一旦发生key的 hash值碰撞，就会在"碰撞"位置开始对整个链表遍历，链表的遍历会带来效率的影响。通常我们使用String，Interger这样的wrapper类作为HashMap 的key。或者使用不可变的、声明作final的对象，并且采用合适的equals()和hashCode()方法，来减少碰撞的发生。

2. 避免扩容，在`resize()`方法中，我们可以看下是怎么扩容的。关键代码：

   ```java
   if (oldCap > 0) {
       if (oldCap >= MAXIMUM_CAPACITY) {
           threshold = Integer.MAX_VALUE;
           return oldTab;
       }
       //通常执行这里，newCap = oldCap << 1 也就是扩大一倍原来的容量
       else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)
           newThr = oldThr << 1; // double threshold
   }
   Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
   ```

   这个开销是非常大的，在table[] 数组中扩容。扩容之后，还要把oldTab 中的数据复制到 newTab中。在我们使用HashMap 的时候尽可能的估算map 的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容 。

3. 不能在多线程中使用HashMap，HashMap是非线程安全的。在扩容的时候，HashMap可能造成死循环。

4. 负载因子可以进行修改，但不建议去修改，因为这个是经过科学计算得出的一个值，除非特殊情况下。

## HashSet分析

上面我们分析了HashMap，为什么我们先分析HashMap 呢？直接进入主题，这里我们列出部分代码：

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {
    
    private transient HashMap<E,Object> map;

	// Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * default initial capacity (16) and load factor (0.75).
     */
    public HashSet() {
        map = new HashMap<>();
    }

    /**
     * Constructs a new set containing the elements in the specified
     * collection.  The <tt>HashMap</tt> is created with default load factor
     * (0.75) and an initial capacity sufficient to contain the elements in
     * the specified collection.
     *
     * @param c the collection whose elements are to be placed into this set
     * @throws NullPointerException if the specified collection is null
     */
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * the specified initial capacity and the specified load factor.
     *
     * @param      initialCapacity   the initial capacity of the hash map
     * @param      loadFactor        the load factor of the hash map
     * @throws     IllegalArgumentException if the initial capacity is less
     *             than zero, or if the load factor is nonpositive
     */
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    /**
         * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
         * the specified initial capacity and default load factor (0.75).
         *
         * @param      initialCapacity   the initial capacity of the hash table
         * @throws     IllegalArgumentException if the initial capacity is less
         *             than zero
         */
    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    /**
     * Constructs a new, empty linked hash set.  (This package private
     * constructor is only used by LinkedHashSet.) The backing
     * HashMap instance is a LinkedHashMap with the specified initial
     * capacity and the specified load factor.
     *
     * @param      initialCapacity   the initial capacity of the hash map
     * @param      loadFactor        the load factor of the hash map
     * @param      dummy             ignored (distinguishes this
     *             constructor from other int, float constructor.)
     * @throws     IllegalArgumentException if the initial capacity is less
     *             than zero, or if the load factor is nonpositive
     */
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
}
```

可以看到，我们的HashSet在构造的时候，都会实例一个HashMap。

再来看`HashSet.add`方法：

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

可以看到实际上是`map.put(e, PRESENT)`，用的是HashMap 的put 方法。对象作为key，`new Object()`作为值存放到map 中。

由此可知，HashSet 的原理跟HashMap 的原理是差不多的。

HashSet 取值：

```java
/** 
 * 返回对此set中元素进行迭代的迭代器。返回元素的顺序并不是特定的。 
 *  
 * 底层实际调用底层HashMap的keySet来返回所有的key。 
 * 可见HashSet中的元素，只是存放在了底层HashMap的key上， 
 * value使用一个static final的Object对象标识。 
 * @return 对此set中元素进行迭代的Iterator。 
 */  
public Iterator<E> iterator() {  
    return map.keySet().iterator();  
}
//通过迭代器遍历获取值
```

HsahSet是基于HashMap 实现的，默认构造函数是构建一个初始容量为16，负载因子为0.75 的HashMap。封装了一个 HashMap 对象来存储所有的集合元素，所有放入 HashSet 中的集合元素实际上由 HashMap 的 key 来保存，而 HashMap 的 value 则存储了一个 PRESENT，它是一个静态的 Object 对象。 

当我们试图把某个类的对象当成 HashMap的 key，或试图将这个类的对象放入 HashSet 中保存时，重写该类的equals(Object obj)方法和 hashCode() 方法很重要，而且这两个方法的返回值必须保持一致：当该类的两个的 hashCode() 返回值相同时，它们通过 equals() 方法比较也应该返回 true。通常来说，所有参与计算 hashCode() 返回值的关键属性，都应该用于作为 equals() 比较的标准。 