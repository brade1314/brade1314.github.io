---
layout:       post
title:        "ArrayList扩容机制"
subtitle:     "ArrayList源码解析"
date:         2021-07-26 18:30:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
- JDK 源码
- ArrayList
---


# ArrayList扩容机制

### 构造器
> 提供了3个构造方法，默认无参构造方法较常用。

```java

    /**
     * 给定初始容量构造方法
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 无参构造方法，空对象数组
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        Object[] a = c.toArray();
        if ((size = a.length) != 0) {
            if (c.getClass() == ArrayList.class) {
                elementData = a;
            } else {
                elementData = Arrays.copyOf(a, size, Object[].class);
            }
        } else {
            // replace with empty array.
            elementData = EMPTY_ELEMENTDATA;
        }
    }

```

### `add` 方法。

```java

    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    /**
     * 在此指定位置插入指定元素列表。 移动当前在该位置的元素（如果有）
     * 和右边的任何后续元素（在它们的索引上加一）。
     *
     * @param index 要插入指定元素的索引
     * @param element 要插入的元素
     * @throws IndexOutOfBoundsException
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }

```

###  `grow()` 扩容方法。

```java

    /**
     * 默认初始化容量
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 计算容量，第1个元素添加时，走 if 分支，ArrayList 容量设置为 DEFAULT_CAPACITY（10）
     */
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

    /**
     * 确保内部容量：根据给定的最小容量进行容量扩增
     */
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    /**
     * 判断是否扩容
     */
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    /**
     *  要分配的数组的最大值
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 扩容：当增加的第 11 个元素 > 默认容量 10 时，
     * 新容量为 10 + (10/2) = 15，即扩容为原来的 1.5 倍
     * @param minCapacity 最小容量
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        // 旧容量 
        int oldCapacity = elementData.length;
        // 新容量，采用位移计算，右移1位，相当于/2
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 是否大于分配的数组的最大值，一般达不到这个量
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

```
