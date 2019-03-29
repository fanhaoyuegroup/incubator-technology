## 一、`二叉树` 和 `TreeMap`

1. 优点：插入快、树形结构清晰、灵活性和扩展性强，即可以变化各种高级树形结构
2. 遍历：
    * 前序遍历：规则（根节点->前序遍历左子树->前序遍历右子树）
    * 中序遍历：规则（中序遍历左子树->根节点->中序遍历右子树）遍历出来的数据一定有序
    * 后序遍历：规则（后序遍历左子树->后序遍历右子树->根节点）
    * 知其中二者，便可求其他一个遍历
### 1、`二叉排序树`

1.1 `性质`

1. 若它的左子树不空，则左子树上所有结点的值均小于它的根节点的值；
2. 它的右子树上所有结点的值均大于它的根节点的值；
3. 若它的右子树上所有结点的值均大于它的根节点的值；

1.2 `Node基本结构`

* left（左节点）
* right（右节点）
* data（数据域）

1.3 `缺点`：最坏情况会产生一边倒情况，与 `链表结构` 一样


### 2、`平衡二叉树`

2.1 `性质`

1. 构建树时要求任意节点的 `深度` 不得过深（子树高度相差不超过1），这棵树最终就是一棵`平衡二叉树`

2.2 `Node基本结构`

* left（左节点）
* right（右节点）
* data（数据域）
* height（高度）

2.3 `Tips`： `高度` 和 `深度` 需要说明一下，高度指的是当前节点到叶子节点的最长路径，例如叶子节点的高度就为0，而深度则是指从根节点到当前节点的最长路径，例如根节点的深度为0

2.4 `目的`：解决二叉排序树节点 `一边倒` 情况

2.5 此种树结构 `难点`：失衡点处理

1. 左左旋转（右旋）- 解决右子树失衡问题
2. 右右旋转（左旋）- 解决左子树失衡问题
3. 左右双旋（先左旋后右旋）- 解决第一次比较左子树大，第二次比较右子树大，一次旋转无法修复
4. 右左双旋（先右旋后左旋）- 解决第一次比较右子树大，第二次比较左子树大，一次旋转无法修复

### 3、`红黑树`

3.1 `性质`

1. 每个节点或者是黑色，或者是红色
2. 根节点是黑色。
3. 每个叶子节点（NIL）是黑色。
4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5. 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。
6. ![image](https://ss0.baidu.com/6ONWsjip0QIZ8tyhnq/it/u=2367683841,405774276&fm=170&s=01B6E83285A6E4A146D908E200005032&w=640&h=308&img.JPEG)

3.2 其本质是一颗二叉排序树，只是多了很多规则使得平衡，与平衡二叉树同级

3.3 `难点`

* 插入节点出现的情况 
* 删除节点出现的情况（远比插入复杂多很多）

3.4 `插入节点情况如下`（假设要插入的节点为 N，N 的父节点为 P，祖父节点为 G，叔叔节点为 U）

1. 插入的新节点 N 是红黑树的根节点，这种情况下，我们把节点 N 的颜色由红色变为黑色，性质2（根是黑色）被满足。同时 N 被染成黑色后，红黑树所有路径上的黑色节点数量增加一个，性质5（从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点）仍然被满足。
![image](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15152058354349.jpg)

2. N 的父节点是黑色，这种情况下，性质4（每个红色节点必须有两个黑色的子节点）和性质5没有受到影响，不需要调整。
![image](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15152058658313.jpg)

3. N 的父节点是红色（节点 P 为红色，其父节点必然为黑色），叔叔节点 U 也是红色。由于 P 和 N 均为红色，所有性质4被打破，此时需要进行调整。这种情况下，先将 P 和 U 的颜色染成黑色，再将 G 的颜色染成红色。此时经过 G 的路径上的黑色节点数量不变，性质5仍然满足。但需要注意的是 G 被染成红色后，可能会和它的父节点形成连续的红色节点，此时需要递归向上调整.
![image](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15152056319752.jpg)

4. N 的父节点为红色，叔叔节点为黑色。节点 N 是 P 的右孩子，且节点 P 是 G 的左孩子。此时先对节点 P 进行左旋，调整 N 与 P 的位置。接下来按照情况五进行处理，以恢复性质4。
![image](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15152057064115.jpg)

5. N 的父节点为红色，叔叔节点为黑色。N 是 P 的左孩子，且节点 P 是 G 的左孩子。此时对 G 进行右旋，调整 P 和 G 的位置，并互换颜色。经过这样的调整后，性质4被恢复，同时也未破坏性质5。
 ![image](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15152057476644.jpg)
 
6. `插入总结`:情况一和情况二比较简单，情况三、四、五稍复杂。但如果细心观察，会发现这三种情况的区别在于叔叔节点的颜色，如果叔叔节点为红色，直接变色即可。如果叔叔节点为黑色，则需要选选择，再交换颜色。当把这三种情况的图画在一起就区别就比较容易观察了，如下图
![image](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15152052502152.jpg)

3.5 `参考`
[红黑树详细分析](http://www.tianxiaobo.com/2018/01/11/%E7%BA%A2%E9%BB%91%E6%A0%91%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90/)


### 4、`TreeMap`

4.1 简介：`TreeMap` 底层基于 `红黑树` 实现，可保证在log(n)时间复杂度内完成 containsKey、get、put 和 remove 操作，效率很高。另一方面，由于 TreeMap 基于红黑树实现，这为 TreeMap 保持键的有序性打下了基础。总的来说，TreeMap 的核心是红黑树，其很多方法也是对红黑树增删查基础操作的一个包装。所以只要弄懂了红黑树，TreeMap 就没什么秘密了。

4.2 源码分析（查找、增加、删除）

1. `查找`：TreeMap基于红黑树实现，而红黑树是一种自平衡二叉查找树，所以 TreeMap 的查找操作流程和二叉查找树一致。二叉树的查找流程是这样的，先将目标值和根节点的值进行比较，如果目标值小于根节点的值，则再和根节点的左孩子进行比较。如果目标值大于根节点的值，则继续和根节点的右孩子比较。在查找过程中，如果目标值和二叉树中的某个节点值相等，则返回 true，否则返回 false。TreeMap 查找和此类似，只不过在 TreeMap 中，节点（Entry）存储的是键值对<k,v>。在查找过程中，比较的是键的大小，返回的是值，如果没找到，则返回null。TreeMap 中的查找方法是get，具体实现在getEntry方法中，相关源码如下：
```java
public V get(Object key) {
    Entry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    
    // 查找操作的核心逻辑就在这个 while 循环里
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

2. `插入`：当往 TreeMap 中放入新的键值对后，可能会破坏红黑树的性质。这里为了描述方便，把 Entry 称为节点。并把新插入的节点称为N，N 的父节点为P。P 的父节点为G，且 P 是 G 的左孩子。P 的兄弟节点为U。在往红黑树中插入新的节点 N 后（新节点为红色），会产生下面5种情况：
    1. N 是根节点
    2. N 的父节点是黑色
    3. N 的父节点是红色，叔叔节点也是红色
    4. N 的父节点是红色，叔叔节点是黑色，且 N 是 P 的右孩子
    5. N 的父节点是红色，叔叔节点是黑色，且 N 是 P 的左孩子
3. 上面5中情况中，情况2不会破坏红黑树性质，所以无需处理。情况1 会破坏红黑树性质2（根是黑色），情况3、4、和5会破坏红黑树性质4（每个红色节点必须有两个黑色的子节点）。这个时候就需要进行调整，以使红黑树重新恢复平衡。接下来分析一下插入操作相关源码

```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    // 1.如果根节点为 null，将新节点设为根节点
    if (t == null) {
        compare(key, key);
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        // 2.为 key 在红黑树找到合适的位置
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            //相等，做更新操作
            else
                return t.setValue(value);
        } while (t != null);
    } else {
        // 与上面代码逻辑类似，省略
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 3.将新节点链入红黑树中
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    // 4.插入新节点可能会破坏红黑树性质，这里修正一下
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}


/** 新增节点后对红黑树的调整方法 */ 
private void fixAfterInsertion(Entry<K,V> x) { 
    // 将新插入节点的颜色设置为红色 
    x. color = RED; 
    // while循环，保证新插入节点x不是根节点或者新插入节点x的父节点不是红色（这两种情况不需要调整） 
    while (x != null && x != root && x. parent.color == RED) { 
// 如果新插入节点x的父节点是祖父节点的左孩子 
        if (parentOf(x) == leftOf(parentOf (parentOf(x)))) { 
        // 取得新插入节点x的叔叔节点 
        Entry<K,V> y = rightOf(parentOf (parentOf(x))); 
        // 如果新插入x的父节点是红色-------------------① 
        if (colorOf(y) == RED) { 
            // 将x的父节点设置为黑色 
            setColor(parentOf (x), BLACK); 
            // 将x的叔叔节点设置为黑色 
            setColor(y, BLACK); 
            // 将x的祖父节点设置为红色 
            setColor(parentOf (parentOf(x)), RED); 
            // 将x指向祖父节点，如果x的祖父节点的父节点是红色，按照上面的步奏继续循环 
            x = parentOf(parentOf (x)); 
         } else { 
         // 如果新插入x的叔叔节点是黑色或缺少，且x的父节点是祖父节点的右孩子-------------------② 
         if (x == rightOf( parentOf(x))) { 
         // 左旋父节点 
            x = parentOf(x); rotateLeft(x); 
         } 
         // 如果新插入x的叔叔节点是黑色或缺少，且x的父节点是祖父节点的左孩子-------------------③ 
         // 将x的父节点设置为黑色 
         setColor(parentOf (x), BLACK); 
         // 将x的祖父节点设置为红色 
         setColor(parentOf (parentOf(x)), RED); 
         // 右旋x的祖父节点 
         rotateRight( parentOf(parentOf (x))); } 
         } else { 
         // 如果新插入节点x的父节点是祖父节点的右孩子，下面的步奏和上面的相似，只不过左旋右旋的区分，不在细讲 
         } 
         // 最后将根节点设置为黑色，不管当前是不是红色，反正根节点必须是黑色 
         root.color = BLACK; 
}
```

3. `删除`
...待完善
