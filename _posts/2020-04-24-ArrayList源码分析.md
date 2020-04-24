---
layout: post
title: ArrayList源码分析
author: 不近视的猫
date: 2020-04-24
categories: blog
tags: []
description: ArrayList源码分析
---


`ArrayList`是我们较为常用的数据集合之一，其基本思路为使用数组去存储数据，若存储的数额大于原有的数组，就会进行扩容操作，而`ArrayList`的增删改查方法其实就是对于数组里面的数据进行增删查。

下面我们来分析下它里面常用方法的源码分析（基于 JDK 版本 1.8）

## 创建
### ArrayList();

```
    public ArrayList() {
    // private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    // 0 容量数组
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

直接给`ArrayList`里面的数据数组赋予一个 0 容量的数组。

### ArrayList(int initialCapacity)

```
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
        //	private static final Object[] EMPTY_ELEMENTDATA = {};
        // 0 容量数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
    

```

- 若`initialCapacity`大于 0，则创建一个`initialCapacity`容量大的数组。
- 若`initialCapacity`等于 0，则赋予一个 0 容量的数组。
- 否则，直接抛出`IllegalArgumentException`，提醒用户传入的`initialCapacity`的值是错误的。

### ArrayList(Collection<? extends E> c)

```
    public ArrayList(Collection<? extends E> c) {
    	//elementData为Object[]类型，c.toArray()可能返回的是其它类型的数组，
    	//例如String[]类型，只不过向上转型，使用elementData进行引用
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // 由于是向上转型，所以，本质上还是原来类型的数组，
            //例如使用String[0]去存储Object的内容，就会报java.lang.ArrayStoreException，
            //所以要进行二次判断，使用真正的Object[]去存储Collection里面的数据
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 赋予一个 0 容量的数组
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

## 添加

### add(E e)
```
    public boolean add(E e) {
    	//确保当前数组能够容纳所添加的数据，size为已存储的数据数
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
    	//获取需要存储当前数据的数目，最小值为10
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        	//private static final int DEFAULT_CAPACITY = 10;
        	//若为默认创建的数组，minCapacity的值为10
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
    	//modCount用于记录对于ArrayList操作过多少次
        modCount++;

        // 当需要存储数据的长度大于当前的数组长度，则进行扩容操作
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //这里其实相当于 oldCapacity*1.5
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 开始扩容操作，新建新的容量数组，并且把之前的值copy过去
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
            //数组容量的最大值为Integer.MAX_VALUE
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

下面梳理下逻辑：

- 计算当前存储数据所需的容量 A（已有的数据量+1）
- 判断容量 A 是否大于 10，若不大于 10，则 A 为 10。重点：**ArrayList 默认初始化的时候，是赋予一个 0 容量数组的，只有在 add 后才开始扩容为 10 容量**
- 判断容量 A 的大小是否已经超出当前数组的长度
	- 否：直接在数组的容量 A 下标的位置上存储数据 
	- 是：进行扩容操作，默认的扩容的大小为数组大小的 1.5 倍，在数组的容量 A 下标的位置上存储数据
		
### add(int index, E element)

```
    public void add(int index, E element) {
    	//判断是否越界
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
		//确保数组可以存储所要添加的数据
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //将index后面的数据往后偏移
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        //赋值
        elementData[index] = element;
        size++;
    }
```

### addAll(Collection<? extends E> c)

```
    public boolean addAll(Collection<? extends E> c) {
    	//将Collection数据转换为数组
        Object[] a = c.toArray();
        int numNew = a.length;
        //确保数组能够存储完全部数据
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //赋值
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
```

### addAll(int index, Collection<? extends E> c)

```
    public boolean addAll(int index, Collection<? extends E> c) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

		//偏移量，即index后面的数据都往后移numMoved位（包含index），
		//即中间空出一段位置，用于后面存储Collection里面的数据
        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);
		//将刚刚空出的位置补上
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```


## 删除

### remove(int index)

```
    public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
        	// 并不是直接把数组上的数据删除掉，而且直接把index背后的数据，全部往前偏移一位
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
         //把最后的一条数据置空
        elementData[--size] = null; 

        return oldValue;
    }
```


### remove(Object o)

```
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
            	//只删除第一位空值
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
        	//跟remove(int index)差不多，都是直接把数据往前偏移一位
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```

### removeRange(int fromIndex, int toIndex) 

```
	//移除一段范围的值，原理都差不多，就不重复说明
    protected void removeRange(int fromIndex, int toIndex) {
        if (toIndex < fromIndex) {
            throw new IndexOutOfBoundsException("toIndex < fromIndex");
        }

        modCount++;
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
```

### clear()

```
    public void clear() {
        modCount++;

        // clear to let GC do its work
        //直接把数组里面的值制空
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```


## 改

### set(int index, E e)

```
    public E set(int index, E element) {
    	//避免下标越界
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        E oldValue = (E) elementData[index];
        //值直接覆盖
        elementData[index] = element;
        return oldValue;
    }
```

## 查

### get(int index)

```
    public E get(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
		//直接把数组的值返回回来
        return (E) elementData[index];
    }
```

### contains(Object o)

```
    public boolean contains(Object o) {
    	//直接调用indexOf(Object o)，获取该值的下标所在，若该值不存在，则返回下标为-1
        return indexOf(o) >= 0;
    }
```

### indexOf(Object o)

```
    public int indexOf(Object o) {
        if (o == null) {
        	//获取第一个空值
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
        	//遍历数组，查找是否有改值
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

### lastIndexOf(Object o)

```
    public int lastIndexOf(Object o) {
    	//思路跟indexOf(Object o)一样，只是遍历的方向是倒序
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```



## 其他
### size()

```
    public int size() {
        return size;
    }
```

**注意：这里返回的并不是数组的容量大小，而是数组所存的值的多少**

### isEmpty()

```
    public boolean isEmpty() {
        return size == 0;
    }
```








