# Collections.unmodifiableList获取一个只读List

[toc]

#
# Collections.unmodifiableList获取一个只读List

## java.util.Collections


函数原型：`public static <T> List<T> unmodifiableList(List<? extends T> list)`


### 源码

```java
/**
* Returns an unmodifiable view of the specified list. This method allows
* modules to provide users with "read-only" access to internal
* lists. Query operations on the returned list "read through" to the
* specified list, and attempts to modify the returned list, whether
* direct or via its iterator, result in an
* <tt>UnsupportedOperationException</tt>.<p>
*
* The returned list will be serializable if the specified list
* is serializable. Similarly, the returned list will implement
* {@link RandomAccess} if the specified list does.
*
* @param <T> the class of the objects in the list
* @param list the list for which an unmodifiable view is to be returned.
* @return an unmodifiable view of the specified list.
*/
public static <T> List<T> unmodifiableList(List<? extends T> list) {
return (list instanceof RandomAccess ?
new UnmodifiableRandomAccessList<>(list) :
new UnmodifiableList<>(list));
}

```


`unmodifiableList`用来**获取一个只可读，不可写的指定的list**。如果试图去写入或者改变这个list，则会抛出一个`UnsupportedOperationException`异常。


### 原理

那么他具体是怎么实现的呢？可以看一下`UnmodifiableList`或者`UnmodifiableRandomAccessList`的代码：


```java

static class UnmodifiableList<E> extends UnmodifiableCollection<E>
implements List<E> {
private static final long serialVersionUID = -283967356065247728L;
final List<? extends E> list;
UnmodifiableList(List<? extends E> list) {
super(list);
this.list = list;
}
public boolean equals(Object o) {return o == this || list.equals(o);}
public int hashCode() {return list.hashCode();}
public E get(int index) {return list.get(index);}
public E set(int index, E element) {
throw new UnsupportedOperationException();
}
public void add(int index, E element) {
throw new UnsupportedOperationException();
}
public E remove(int index) {
throw new UnsupportedOperationException();
}
public int indexOf(Object o) {return list.indexOf(o);}
public int lastIndexOf(Object o) {return list.lastIndexOf(o);}
public boolean addAll(int index, Collection<? extends E> c) {
throw new UnsupportedOperationException();
}
@(实用开发技巧)Override
public void replaceAll(UnaryOperator<E> operator) {
throw new UnsupportedOperationException();
}
@Override
public void sort(Comparator<? super E> c) {
throw new UnsupportedOperationException();
}
public ListIterator<E> listIterator() {return listIterator(0);}
public ListIterator<E> listIterator(final int index) {
return new ListIterator<E>() {
private final ListIterator<? extends E> i
= list.listIterator(index);
public boolean hasNext() {return i.hasNext();}
public E next() {return i.next();}
public boolean hasPrevious() {return i.hasPrevious();}
public E previous() {return i.previous();}
public int nextIndex() {return i.nextIndex();}
public int previousIndex() {return i.previousIndex();}
public void remove() {
throw new UnsupportedOperationException();
}
public void set(E e) {
throw new UnsupportedOperationException();
}
public void add(E e) {
throw new UnsupportedOperationException();
}
@Override
public void forEachRemaining(Consumer<? super E> action) {
i.forEachRemaining(action);
}
};
}
public List<E> subList(int fromIndex, int toIndex) {
return new UnmodifiableList<>(list.subList(fromIndex, toIndex));
}
/**
* UnmodifiableRandomAccessList instances are serialized as
* UnmodifiableList instances to allow them to be deserialized
* in pre-1.4 JREs (which do not have UnmodifiableRandomAccessList).
* This method inverts the transformation. As a beneficial
* side-effect, it also grafts the RandomAccess marker onto
* UnmodifiableList instances that were serialized in pre-1.4 JREs.
*
* Note: Unfortunately, UnmodifiableRandomAccessList instances
* serialized in 1.4.1 and deserialized in 1.4 will become
* UnmodifiableList instances, as this method was missing in 1.4.
*/
private Object readResolve() {
return (list instanceof RandomAccess
? new UnmodifiableRandomAccessList<>(list)
: this);
}
}

```


可以看到，实现很简单，除了读操作相关的方法之外，例如`remove,add,replaceAll`,之类的操作，都抛出了`UnsupportedOperationException`异常。



参考地址：http://studyai.site/2016/04/14/java%E6%8A%80%E5%B7%A7%EF%BC%8DCollections.unmodifiableList/
