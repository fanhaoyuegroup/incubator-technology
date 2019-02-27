#坑死我的remove之ConcurrentModificationException
####先上段代码
````
    List<String> list = new ArrayList<>();
    Collections.addAll(list,"str1","str2","str3","str4","str5");
    list.forEach(str->{
        if("str1".equals(str))
        list.remove(str);
    });
````
本来想删除"str1"，结果喜提ConcurrentModificationException
接下来就看看，我喜提ConcurrentModificationException的道路上，ArrayList作出了哪些努力与贡献
首先：
````
 public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);//判空
        final int expectedModCount = modCount;//预期计算=计数，modCount敲黑板，要考
        @SuppressWarnings("unchecked")
        final E[] elementData = (E[]) this.elementData;
        final int size = this.size;
        //遍历的时候，判断了 modCount==expectedModCount
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            action.accept(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();//举报，就是它。
        }
    }
````
这里我们已经看到ConcurrentModificationException了，但是，我是怎么走到这一步的呢？
上面if里面条件：当modCount != expectedModCount时，抛出异常，也就是说modCount发生了改变？！！！
那他是啥时候改变的，为啥改变了/fd，再往下走走看。我们顺利的拿到了第一个数据，并且返回了，那么
一定是remove搞得鬼咯。看看remove
```
public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);//o!=null，执行fastRemove
                    return true;
                }
        }
        return false;
    }
```
纵观上面的代码，跟modCount没有关系！那真凶肯定是藏在fastRemove里了
```
 private void fastRemove(int index) {
        modCount++;//！！！！
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
modCount++啊啊啊，为什么要++，看看modCount作何解释
````
   /**
     * The number of times this list has been <i>structurally modified</i>.
     * Structural modifications are those that change the size of the
     * list, or otherwise perturb it in such a fashion that iterations in
     * progress may yield incorrect results.
     *
     * <p>This field is used by the iterator and list iterator implementation
     * returned by the {@code iterator} and {@code listIterator} methods.
     * If the value of this field changes unexpectedly, the iterator (or list
     * iterator) will throw a {@code ConcurrentModificationException} in
     * response to the {@code next}, {@code remove}, {@code previous},
     * {@code set} or {@code add} operations.  This provides
     * <i>fail-fast</i> behavior, rather than non-deterministic behavior in
     * the face of concurrent modification during iteration.
     *
     * <p><b>Use of this field by subclasses is optional.</b> If a subclass
     * wishes to provide fail-fast iterators (and list iterators), then it
     * merely has to increment this field in its {@code add(int, E)} and
     * {@code remove(int)} methods (and any other methods that it overrides
     * that result in structural modifications to the list).  A single call to
     * {@code add(int, E)} or {@code remove(int)} must add no more than
     * one to this field, or the iterators (and list iterators) will throw
     * bogus {@code ConcurrentModificationExceptions}.  If an implementation
     * does not wish to provide fail-fast iterators, this field may be
     * ignored.
     */
````
@_@The number of times this list has been structurally modified(以下来自谷歌翻译：此列表已被结构修改的次数)
成功破案，真相只有一个：当list发生结构的更改时，modCount都+1，以防止正在进行中的迭代产生不正确的结果。



