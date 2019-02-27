### 一、HashSet

#### 1.源码小析

**（1）一些属性**

``` java
    s  tatic final long serialVersionUID = -5024744406713321676L;
    //HashSet的实现是由HashMap<E,Object>类型的map实现。所以放入hashset集合的元素均存进了map的key中
    private transient HashMap<E,Object> map;
    // Dummy value to associate with an Object in the backing Map
    //map中的每一个键值对<key,value>的value值均为该静态Object对象PRESENT
    private static final Object PRESENT = new Object();

```
**（2）构造函数**
``` java
    /** 
      * 默认的无参构造器，构造一个空的HashSet。 
      *  
      * 实际底层会初始化一个空的HashMap，并使用默认初始容量为16和加载因子0.75。 
      */  
    public HashSet() {
        map = new HashMap<>();
    }
    /** 
     * 构造一个包含指定collection中的元素的新set。 
     * 
     * 实际底层使用默认的加载因子0.75和足以包含指定 
     * collection中所有元素的初始容量来创建一个HashMap。 
     * @param c 其中的元素将存放在此set中的collection。 
     */  
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
    //对linkedHashSet的支持
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }

```

**（3）相关方法**

``` java

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
    public void clear() {
            map.clear();
    }
    ......
```

#### 2、不允许重复元素：
底层实现是HashMap，HashSet是通过将值保存到HashMap中的key中，value值存储的都是PRESENT的Object对象。在HashMap中存储相同的key时（hashCode()返回值相等，通过equals比较也返回true），key值不会有变化，value新值会将旧值覆盖，
#### 3、线程不安全
底层实现是HashMap,由前面的学习可知，HashMap是线程不安全的，jdk1.7中，多线程情况下HashMap在扩容时会造成死循环；jdk1.8中，多线程情况下HashMap在扩容时会造成数据丢失。故HashSet也是线程不安全的
替代方案：使用Collections.synchronizedSet

#### 4.HashSet和HashMap的比较
| HashSet      |	HashMap  |
|-----------|---------|
| HashSet实现了Set接口| HashMap实现了Map接口|	
| 存储对象| 存储键值对|	
|HashMap中使用键对象来计算hashcode值|HashSet使用成员对象来计算hashcode值，对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性，如果两个对象不同的话，那么返回false|

### 二、LinkedHashSet

少的可怜的源码：
```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    private static final long serialVersionUID = -2851667679971038690L;

    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }
    public LinkedHashSet() {
        super(16, .75f, true);
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

    @Override
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
    }
}

```

保证了遍历的顺序和插入的顺序一致

###小结
HashSet存储元素是无序且不等于访问顺序
LinkedHashSet存储元素是无序的，但是由于双向链表的存在，迭代时获取元素的顺序等于元素的添加顺序，注意这里不是访问顺序
HashSet的public类型构造函数均是采用HashMap实现，所以HashSet能够存储不重复的对象，包括NULL。
LinkedHashSet通过继承HashSet，采用LinkedhashMap进行实现，所以LinkedHashSet除了具有HashSet的功能外，还能保证元素按照加入顺序进行排序。
