# 排序

## 前言

算法的精髓是屏蔽了一切可能出现的意外情况，而不是解决一切可能出现的意外情况

## 限制

+ 主存中完成，内部排序
+ 借助硬盘，外部排序
+ 排序算法都需要考虑一个平均的情况，但是平均情况确实最难确认的 _（平均从来都不是二分之一）_
+ 下界 _(算法的最坏情况)_

## 注意点

+ 对于特殊情况的考虑有时候可以并入算法，有时候需要单独提出

## 排序算法

### 插入排序

+ 双for循环的时间复杂度是O(N^2)，一般也是我们所考虑的下限，最简单，最容易也是效率最低的算法
+ 换位，边比较边换位，每一位比较后符合条件是不可避免的
+ 时间复杂度 O(N^2)
+ 空间复杂度 

```java
    /**
     * 插入排序
     * @param data 待排序数据
     * @param <AnyType> 泛型
     */
    public static <AnyType extends Comparable<? super AnyType>> void insertionSort(AnyType[] data) {
        int i;
        for (int p = 1; p < data.length; p++) {
            AnyType tmp = data[p];
            for (i = p; i > 0 && tmp.compareTo(data[i - 1]) < 0; i--) {
                data[i] = data[i - 1];
            }

            data[i] = tmp;
        }
    }
```

### 希尔排序（缩减增量排序）

+ 增量序列的选择对性能影响很大
+ 增量序列的最后一个数必定为1，这样最后一次就是插入排序，但是并没有完全的插入排序的比较和换位
+ 增量序列极其重要 _有建议是N/2_
+ 希尔排序在发现后1000年才被证明
+ 时间复杂度N^3/2

_注意为什么交互空间开销大，参考零拷贝的问题_

```java
    /**
     * 希尔排序
     * @param data 待排序数据
     * @param <AnyType> 泛型
     */
    public static <AnyType extends Comparable<? super AnyType>> void shellSort(AnyType[] data) {
        int i;

        // 增量序列的构造
        for (int gap = data.length / 2; gap > 0; gap /= 2) {
            // 类似插入排序
            for (int p = gap; p < data.length; p++) {
                AnyType tmp = data[p];
                for (i = p; i >= gap && tmp.compareTo(data[i - gap]) < 0; i -= gap) {
                    data[i] = data[i - gap];
                }

                data[i] = tmp;
            }
        }
    }
```


### 堆排序

+ 空间换时间

```java
```

### 归并排序


最大回旋数 算法 待排序的数据预处理 很重要
