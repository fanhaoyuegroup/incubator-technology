## 一、HashMap中的红黑树
我们都知道在jdk1.8中将HashMap的数据结构进行了优化，有了红黑树结构的加入，今天我们来看看源码，了解一下这个红黑树。
 
### 1.TreeNode()方法
```java
     static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
          TreeNode<K,V> parent;  // red-black tree links
          TreeNode<K,V> left;
          TreeNode<K,V> right;
          TreeNode<K,V> prev;    // needed to unlink next upon deletion
          //仿佛看到了红黑树的影子，默认为红色
          boolean red;
          TreeNode(int hash, K key, V val, Node<K,V> next) {
              super(hash, key, val, next);
          }
}
    
```

### 2.源码分析
在前几次的分享中,HashMap中还有个特别...的红黑树，今天来看下：

（1）树化

树化的条件： 当链表长度大于TREEIFY_THRESHOLD（默认值是8）
 ``` 
       源码中这样体现：
          if (binCount >= TREEIFY_THRESHOLD - 1) 
             treeifyBin(tab, hash);
```     
我们来看下树化的源码：
``` java
  //树化
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        //如果tab为空或者tab的数量小于树化的最小阈值64,进行"扩容"
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        //可进行树化，指定桶的第一个节点
        else if ((e = tab[index = (n - 1) & hash]) != null) {
        //红黑树的  头hd /  tl尾
            TreeNode<K,V> hd = null, tl = null;
            do {
            //将该节点转化为树节点
                TreeNode<K,V> p = replacementTreeNode(e, null);
                //尾指针为空
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            //1.Node->TreeNode
            //2.单向链表->双向链表
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
  
  
```
resize中调用的方法：当结点是树的时候，调用split()
``` java

    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
    //split()中调用：如果小于UNTREEIFY_THRESHOLD，调用untreeify(),解除树                 
    if (lc <= UNTREEIFY_THRESHOLD)
                        tab[index] = loHead.untreeify(map);
```
上面只是将链表的结构转化为树的结构，treeify()才是真正生成红黑树
 
``` java
     //
            final void treeify(Node<K,V>[] tab) {
                TreeNode<K,V> root = null;
                for (TreeNode<K,V> x = this, next; x != null; x = next) {
                    next = (TreeNode<K,V>)x.next;
                    x.left = x.right = null;
                    //如果根节点是null表示初次树化，确定根节点，是黑色
                    if (root == null) {
                        x.parent = null;
                        x.red = false;
                        root = x;
                    }
                    else {
                        K k = x.key;
                        int h = x.hash;
                        Class<?> kc = null;
                        //又来了一个循环，从根节点开始遍历
                        for (TreeNode<K,V> p = root;;) {
                            int dir, ph;
                            K pk = p.key;
                            //当前节点的hash值>指定值的hash值，标记为左边
                            if ((ph = p.hash) > h)
                                dir = -1;
                            //当前节点的hash值<指定值的hash值，标记为右边
                            else if (ph < h)
                                dir = 1;
                            //hash值相等 ，那么还要通过其他方式再进行比较：
                           1.如果当前链表节点的key实现了comparable接口，并且当前树节点和链表节点是相同Class的实例，那么通过comparable的方式再比较两者。
                           2.如果还是相等，最后再通过tieBreakOrder比较一次
                          
                            else if ((kc == null &&
                                      (kc = comparableClassFor(k)) == null) ||
                                     (dir = compareComparables(kc, k, pk)) == 0)
                                dir = tieBreakOrder(k, pk);
                            //记录当前的结点
                            TreeNode<K,V> xp = p;
                            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                                x.parent = xp;
                                if (dir <= 0)
                                    xp.left = x;
                                else
                                    xp.right = x;
                                    
                            //
                                root = balanceInsertion(root, x);
                                break;
                            }
                        }
                    }
                }
                moveRootToFront(tab, root);
            }
            
            //当前值 a   指定值  b
             static int tieBreakOrder(Object a, Object b) {
                        int d;
                        if (a == null || b == null ||
                            (d = a.getClass().getName().
                             compareTo(b.getClass().getName())) == 0)
                            d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                                 -1 : 1);
                        return d;
                    }

```
平衡操作：
下面进行红黑上色：

我们来复习一下红黑树的几个基本条件：
- 节点是红色或黑色
- 根节点是黑色
- 每个叶节点（NIL节点，空节点）是黑色的
- 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)
- 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点

``` java
             //
             static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                                TreeNode<K,V> x) {
                        x.red = true;
                        //x:儿子 xp:父亲 xpp:爷爷 xppl:爷爷左边  xppr:爷爷右边
                        for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                            //1.x结点为根节点
                            if ((xp = x.parent) == null) {
                                x.red = false;
                                return x;
                            }
                            //2.父亲是黑色的，或者爷爷结点为空
                            else if (!xp.red || (xpp = xp.parent) == null)
                                return root;
                             //3.父亲是爷爷的左边
                            if (xp == (xppl = xpp.left)) {
                                //3-1：xppr(叔叔)结点不是空，且为红色
                                if ((xppr = xpp.right) != null && xppr.red) {
                                    xppr.red = false; //叔叔为黑
                                    xp.red = false; //父亲为黑
                                    xpp.red = true; //爷爷为红
                                    x = xpp; //爷爷结点是下一个遍历的开始
                                    //此时X=60，当前遍历的结点是60
                                }
                                
                                //为什么将爷爷作为下一个遍历的开始？
                               // 当前节点、父节点、叔叔节点之间都不会违背红黑树特性，但祖父节点却不一定。
                               // 若此时，祖父节点是根节点，直接将祖父节点设为“黑色”，那就完全解决这个问题了；
                               // 若祖父节点不是根节点，那我们需要将“祖父节点”设为“新的当前节点”，接着对“新的当前节点”进行分析
                                 
                                else {
                                     //3-2:X为右孩子
                                    if (x == xp.right) {
                                    //左旋
                                        root = rotateLeft(root, x = xp);
                                         //此时X=40,当前遍历的结点是40
                                        xpp = (xp = x.parent) == null ? null : xp.parent;
                                    }
                                     //3-3:X为左孩子
                                    if (xp != null) {
                                        //将60变化成黑色
                                        xp.red = false;
                                        if (xpp != null) {   //80
                                            //将80至为红色
                                            xpp.red = true;
                                           //右旋
                                            root = rotateRight(root, xpp);
                                        }
                                    }
                                }
                            }   
                          
                             //父亲是爷爷的右边          
                            .....
                            }
                        }
                    }
``` 
补图说明以上代码中：

![红黑树示例图](img/TreeMap.png)

``` java
        //左旋   该例子中传入的值为（80,40）
        static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            if (p != null && (r = p.right) != null) {
                if ((rl = p.right = r.left) != null)
                    //将50的父亲指向p(40)
                    rl.parent = p;
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false;
                else if (pp.left == p)
                    //将60-》80的左孩子
                    pp.left = r;
                else
                    pp.right = r;
                //将40-》60的左孩子
                r.left = p;
                p.parent = r;
            }
            return root;
        }
        
 ```
 ![左旋](img/左旋.jpg)
       
 ``` java
        //右旋  该例子中传入的参数是（80,80）
        static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr; //l=p.left=60 p=80
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)
                    //将70-》80的左孩子
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    //确定root为60
                    (root = l).red = false;
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                //80->60的右孩子
                l.right = p;
                p.parent = l;
            }
            return root;
        }

```

 ![右旋](img/右旋.jpg)

解除树化
``` java
//解除就没有树化那么。。。
    原来的红黑树中也实现了双向链表，所以在这里就是将原来TreeNode->Node
 final Node<K,V> untreeify(HashMap<K,V> map) {
            Node<K,V> hd = null, tl = null;
            for (Node<K,V> q = this; q != null; q = q.next) {
                Node<K,V> p = map.replacementNode(q, null);
                if (tl == null)
                    hd = p;
                else
                    tl.next = p;
                tl = p;
            }
            return hd;
        }

```

红黑树原理和算法详细介绍：http://www.cnblogs.com/skywang12345/p/3245399.html

接下来是。。。。。。。

---------------------------------------------------------------------------------------
#### HashMap为什么要使用红黑树？
    JDK1.8以前，HashMap通过数组+链表实现，在1.8之后加入红黑树，那么使用红黑树肯定是为了解决1.8以前
    HashMap中的不足之处。那么以前的HashMap又有哪些不足？
   1. JDK1.8以前，HashMap通过数组+链表实现，即使哈希函数取得再好，也很难达到元素百分百均匀分布。
   2. 当HashMap中有大量的元素都放到同一个桶中，这个桶下面就会有一条特别长的链表，遍历的复杂度就是O(n),
       失去了HashMap的优势。
    通过使用红黑树的结构，来优化了上述情况，
    
     