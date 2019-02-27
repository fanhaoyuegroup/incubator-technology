## 一、认识一下LinkedHashMap
 LinkedHashMap实际上就是在HashMap的基础上维护了一个双向链表。
 LinkedHashMap大多数的方法是直接继承HashMap，为了实现双向链表的相关功能覆写了相关的方法。
 LinkedHashMap示意图 ：
 ![LinkedHashMap结构示意图](img/LinkedHashMap01.jpg)
 
### 1.构造方法

``` 
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }

    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }


    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    
```
构造函数中有个 accessOrder：

元素的遍历顺序：
 
 false: 按照插入的顺序遍历  
true: 将访问过的元素放到最后）

来看下小demo，稍后来看看源码实现。
``` 
public class TestLinkedHashMap {
    public static void main(String[] args) {
        Boolean accessOrder = true;
        LinkedHashMap map = new LinkedHashMap(16,0.75f,accessOrder);
        map.put(1,"a");
        map.put(2,"b");
        map.put(3,"c");
        map.put(4,"d");
        map.put(5,"f");
        System.out.println(map);
        map.get(1);
        map.get(2);
        System.out.println(map);

    }
}
//打印的结果：
//{1=a, 2=b, 3=c, 4=d, 5=f}
//{3=c, 4=d, 5=f, 1=a, 2=b}

```
实现双向链表的结构
``` 
   static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

    private static final long serialVersionUID = 3801124242820219131L;

    
    transient LinkedHashMap.Entry<K,V> head;

    
    transient LinkedHashMap.Entry<K,V> tail;

   
    final boolean accessOrder;

```


### 2.源码分析
上次的源码分享中HashMap的put方法中除了红黑树，留下了一丢丢LinkedHashMap的影子，我们来看下
 ``` 
        final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {
             ......
             afterNodeAccess(e);      
             ......
               
             afterNodeInsertion(evict);
             ......
        }
        
       插一句，在put方法,LinkedHashMap的newNode()方法的重写：
        
         Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
                LinkedHashMap.Entry<K,V> p =
                    new LinkedHashMap.Entry<K,V>(hash, key, value, e);
                    
                linkNodeLast(p);
                return p;
            }
         
          // link at the end of list  将新节点放在链表的尾部
             private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
                 LinkedHashMap.Entry<K,V> last = tail;
                 tail = p;
                 //链表为空
                 if (last == null)
                     head = p;
                 else {
                 将新节点放在链表的尾部
                     p.before = last;
                     last.after = p;
                 }
             }


```     

LinkedHashMap中覆写了这几个方法：

（1）新增结点
``` 
// possibly remove eldest   新结点插入之后，删除最早插入的结点
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
    //删除最早的结点，默认返回false
      protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
            return false;
        }
 evict：默认传false
 false: 不实现
 true:  实现LRU(缓存淘汰算法，这个...)    

```
（2）删除结点
``` 
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        //1.先判断是否存在这个table
        if ((tab = table) != null && (n = tab.length) > 0 &&
            //2.判断头结点是否存在
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
            //3.判断是否是树结构
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                //4.标记要删除结点
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                    //删除单链表中的结点
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                //对LinkedHashMap的处理，重新组织双向链表结构
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }


//结点删除之后，重新组合双向链表关系
   void afterNodeRemoval(Node<K,V> e) { 
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        //说明为头结点
        if (b == null)
            head = a;
        else
            b.after = a;
            //说明为尾结点
        if (a == null)
            tail = b;
        else
            a.before = b;
    }

```
删除示意图：
![删除示意图](img/LinkedHashMap02.jpg)

（3）get方法，遍历顺序变更
``` 
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
            //accessOrder在面的小demo中看到，true/false不同
        if (accessOrder)
        //为true时的处理
            afterNodeAccess(e);
        return e.value;
    }

    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        //访问的结点不是尾结点
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            //头结点
            if (b == null)
                head = a;
            else
                b.after = a;
                //不是尾结点
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
                //将p结点放到最后
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

（4）containsValue()方法

```  
LinkedHashMap:
   public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
HashMap:
 public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```
根据这个方法可看出，LinkedHashMap与HashMap的结构差别，LinkedHashMap只要一个for循环遍历，她是一个双向链表，而HashMap是数组加链表的结构，所以是两个for循环遍历。

（5）迭代器
``` 

   LinkedHashMap: // Iterators

    abstract class LinkedHashIterator {
        LinkedHashMap.Entry<K,V> next;
        LinkedHashMap.Entry<K,V> current;
        int expectedModCount;

        LinkedHashIterator() {
            next = head;
            expectedModCount = modCount;
            current = null;
        }

        public final boolean() {
            return next != null;
        }

        final LinkedHashMap.Entry<K,V> nextNode() {
            LinkedHashMap.Entry<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            current = e;
            next = e.after;
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }

  HashMap: // iterators

    abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            //遍历到第一个不为空的
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
                 // 当前的桶遍历完了就开始遍历下一个桶
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }

```

接下来是。。。。。。。

     