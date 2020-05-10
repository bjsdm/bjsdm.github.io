---
layout: post
title: LinkedList源码解析
author: 不近视的猫
date: 2020-05-10
categories: blog
tags: []
description: LinkedList源码解析
---



`ArrayList`与`LinkedList`的区别在于：

- `ArrayList`内部使用数组进行实现
- `LinkedList`内部实现方式：
	- 使用链表进行实现
	- 双向链表，即一个节点会记录前一个节点的位置和下个节点的位置
	- 记录首节点的位置以及尾结点的位置

若对于`ArrayList`的实现原理还未了解，可以先看看[ArrayList源码解析](https://blog.csdn.net/m0_46278918/article/details/105640871)。

下面我们来分析下`LinkedList`里面常用方法的源码分析

## 创建

### LinkedList()

```
    public LinkedList() {
    }
```

无任何操作

### LinkedList(Collection<? extends E> c)

```
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    public boolean addAll(Collection<? extends E> c) {
    	//size为当前的容量，默认添加到尾部
        return addAll(size, c);
    }

    public boolean addAll(int index, Collection<? extends E> c) {
    	//判断该下标是否越界
        checkPositionIndex(index);
		
        Object[] a = c.toArray();
        int numNew = a.length;
        //无数据，添加失败
        if (numNew == 0)
            return false;
		//pred为插入位置的前一个节点，succ为插入size下标位置的节点
        Node<E> pred, succ;
        if (index == size) {
            succ = null;
            pred = last;
        } else {
        	//获取下标位置的节点
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            //将Collection的每个数据都转换为一个节点
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
            	//设置首节点
                first = newNode;
            else
            	//将新节点插入尾部
                pred.next = newNode;
            //刷新新的尾结点
            pred = newNode;
        }

        if (succ == null) {
        	//直接记录尾结点
            last = pred;
        } else {
        	//这个适用于插入的数据在链表的中部，
        	//在新增节点的尾部接上之前size后面那块的数据（包括size下标）
            pred.next = succ;
            succ.prev = pred;
        }
		//更新链表长度
        size += numNew;
        //modCount用于记录更改次数
        modCount++;
        return true;
    }
```

```
	//获取下标节点
    Node<E> node(int index) {
        // assert isElementIndex(index);
		//首先，对于 index 值进行判断，判断在 size 的前一半还是在后一半
		//若是前一半，则通过头节点，一个个往后获取
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
        //若是后一半，则通过尾结点，一个个往前获取
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

```
	//判断下标的合理性
    private void checkPositionIndex(int index) {
        if (!isPositionIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private boolean isPositionIndex(int index) {
        return index >= 0 && index <= size;
    }

```

## 添加

### addFirst(E e)
将数据插入到首节点

```
    public void addFirst(E e) {
        linkFirst(e);
    }
    
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
        	//链表只有一个数据，首节点和尾结点为同一个节点
            last = newNode;
        else
        	//在新建节点的尾部插入之前的链表
            f.prev = newNode;
        size++;
        modCount++;
    }
```

### addLast(E e)
将数据插入到尾节点

```
    public void addLast(E e) {
        linkLast(e);
    }

    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
        	//将新建的节点插入到链表尾部
            l.next = newNode;
        size++;
        modCount++;
    }
```

### add(E e)
添加节点，直接调用`addLast(E e)`

```
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
```

### addAll(Collection<? extends E> c)
请查看`LinkedList(Collection<? extends E> c)`分析

### addAll(int index, Collection<? extends E> c)

请查看`LinkedList(Collection<? extends E> c)`分析

### add(int index, E element)
在链表中部插入数据

```
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
	//主体思路和addAll(int index, Collection<? extends E> c)一样
	//都是先拿到index下标位置的数据和它前一个位置的数据，然后再进行中间插入
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

## 删除

### removeFirst()
移除首部节点，并且返回首节点的值

```
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        //将首节点的内容以及其对下个节点的指向都置为null，便于资源回收
        f.item = null;
        f.next = null; // help GC
        //重新指向首节点，并且更新首节点或者尾结点的状态
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```

### removeLast()
移除尾结点，并且返回尾结点的值

```
    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }
	//原理跟unlinkFirst(Node<E> f) 类似，只是把对首节点的操作改为尾结点而已
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```

### remove(Object o)
删除某个值

```
    public boolean remove(Object o) {
    	//从首节点开始一直遍历，直到找到相对应的值，并将其移除
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
	//主要思路为：获取移除节点的上个节点和下个节点，将上节点之间指向下节点
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
		//判断要移除的节点是不是首节点
        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }
		//判断要移除的节点是不是尾结点
        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }

```

### remove(int index)
移除某个下标值

```
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
    //判断下标是否越界
    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

### remove()
默认移除首节点

```
    public E remove() {
        return removeFirst();
    }
```

## 改
###  set(int index, E element)
给 index 下标赋新值

```
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
```

## 查
### getFirst()
获取第一个节点值

```
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
```

### getLast()
获取最后一个节点值

```
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }
```

### get(int index)
获取`index`下标的节点值

```
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
```

###  indexOf(Object o)
获取值的下标（从首部开始遍历）

```
	//从头到尾依次遍历，若值相等，则返回相应下标
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```

### lastIndexOf(Object o)
获取值的下标（从尾部开始遍历）

```
    public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }
```

### contains(Object o)
是否包含某值

```
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }
```

## 其它

### size()

```
    public int size() {
        return size;
    }
```





