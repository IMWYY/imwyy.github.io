---
title: 你真的会二分查找吗？(结合leetcode)
date: 2018-05-05
tags: Algorithm
---

**转载请注明出处。**
写这篇文章的初衷是因为leetcode遇到了一个坑。我们先一起来看看。

## leetcode 34

Given an array of integers nums sorted in ascending order, find the starting and ending position of a given target value.

Your algorithm's runtime complexity must be in the order of O(log n).

If the target is not found in the array, return [-1, -1].

Example 1:Input: nums = [5,7,7,8,8,10], target = 8
Output: [3,4]

Example 2:Input: nums = [5,7,7,8,8,10], target = 6
Output: [-1,-1]

## 代码

要求算法复杂度为O(log n)。很显然，肯定是二分查找。二分查找到target然后在两边扩展找边界呗。

```java
public int[] searchRange(int[] nums, int target) {
    if (nums.length == 0) return new int[]{-1, -1};
    int left = 0, right = nums.length - 1, mid;

    while (left < right) {
        mid = (left + right)/ 2;

        if (nums[mid] < target) {
            left = mid;
        } else if (nums[mid] > target){
            right = mid;
        } else {
        	  left = mid;
        	  right = mid;
        	  break;
        }
    }

    if (nums[left] != target) {
        return new int[]{-1, -1};
    } else {
        while (left - 1 >= 0 && nums[left - 1] == target) left--;
        while (right + 1 < nums.length && nums[right + 1] == target) right++;
        return new int[]{left, right};
    }
}
```

然后拿个测试[5, 7, 7, 8, 8, 10]， target6 用例跑一下。WTF？死循环？？

## 问题在哪？

很困惑，之前一直这么写，没出过问题，怎么会死循环。那我就跟着代码走一遍看看。

数组[5, 7, 7, 8, 8, 10]， target 6

| left | right | mid  |
| :--: | :---: | :--: |
|  0   |   5   |  2   |
|  0   |   2   |  1   |
|  0   |   1   |  0   |
|  0   |   1   |  0   |
|  0   |   1   |  0   |

找到了问题。left=0，right=1， mid=0的时候，nums[left]=5, nums[right]=7,nums[mid]=5在如下代码里，会一直执行if的第一个分支，三个值都不会改变。

```java
if (nums[mid] <= target) {
    left = mid;
} else {
    right = mid;
}
```

而**left = mid;**才是罪魁祸首，如果这么写了，一旦target元素不在数组，那么就是死循环。所以应该改成**left = mid + 1;**。这样就顺利通过test case。总结起来，这个错误更是因为自己的疏忽。

## 还可能有什么问题？

1. mid的选取

	代码中我写的是`mid = (left + right)/ 2;`，但其实是有问题的，问题在于如果left和right都很大，那么两者相加是很可能溢出的。所以最好的写法应该是`mid = left + (right - left)/ 2;`

2. left和right怎么跳?

	这问题也是我最初出现死循环的原因。我把`left = mid`改成了`left = mid + 1`，为什么不把`right = mid`也改成`right = mid - 1`呢。在这里当然也是可以的，因为我只是想在数组里找到一个等于target的值的下标。

**验证一下我的结论**，我们来看一下jdk里面对于二分查找的实现。在`Arrays`这个类里面。

```java
// Like public version, but without range checks.
private static int binarySearch0(int[] a, int fromIndex, int toIndex, int key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        int midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
```

没错这里确实是，`low = mid + 1`和`high = mid - 1`。但是前面mid的选取好像和我说的有点冲突。

`int mid = (low + high) >>> 1`。这里>>>是指无符号右移，高位补0，而这里的low、high都是非负数，用这个应该只是一个防御性的编程。
>（计算机移位操作的效率要高很多，jdk里面很多用到了移位，如`ArrayList`、`HashMap`等等）

low和high直接相加，难道jdk没有考虑溢出的情况吗？其实不是，Java里数组的长度是有限制的，这个限制具体是由jvm可配置的（？）,我查看源码没有找到具体值是多少，至少当我执行`int[] test = new int[Integer.MAX_VALUE/2];`的时候，会报OutOfMemoryError错。既然这样也就不用考虑溢出问题，直接用移位操作反而更快。

## 更多情况

上面讨论的都是标准的二分查找，也就是在一个数组里找一个数的下标，找不到返回负数。其实还会经常遇到另一种，就是找到第一次出现该数的下标，或者最后一次出现该数的下标。显然jdk里的`binarySearch`并不能实现这个要求，java doc里面也有明说

> Searches the specified array of ints for the specified value using thebinary search algorithm. **If the array contains multiple elements with the specified value, there is no guarantee which one will be found.**

对于"查找第一次出现该数的下标"，我们可以理解成二分查找就是left不断往后走，找到目标位置的过程（最终结果肯定是left的值）。所以为了不出错，我们最好每次移动只改变left。

而对于“查找最后一次出现该数的下标”可以看作是“查找第一次出现该数的下标“的镜像，可以理解为后从往前进行二分查找（把数组反转就是求解第一种情况了），**所以在mid选取的时候，要反一反`mid = right - ((right-left)>> 1)`**

> 看到很多地方的写法都是`mid = left + ((right-left + 1)>> 1);`，这样反而不好理解。写成`mid = right - ((right-left)>> 1)`就好理解多了。

1. 二分查找第一次出现该数的下标

	```java
	while (left < right) {
	    mid = left + ((right-left)>> 1);

	    if (nums[mid] < target) {
	    // 如果中间值比target小，left移到mid+1
	        left = mid + 1;
	    } else {
	    // 如果中间值不比target小即可能等于，right移到mid
	        right = mid;
	    }
	 }
	```

2. 二分查找最后一次出现该数的下标

	```java
	while (left < right) {
	    mid = right - ((right-left)>> 1);//注意这里！！

	    if (nums[mid] <= target) {
	    // 如果中间值小于等于target即不比target大，left移到mid 因为left的目标是和target相等的最后一个下标
	        left = mid;
	    } else {
	    // 如果中间值比target大，right移到mid-1
	        right = mid -1 ;
	    }
	 }
	```

## 完善

仔细看看自己的解法，最后那个while循环有点问题，如果这个范围很大，那岂不是要循环整个数组了？

所以有了前面的铺垫，这个题目可以用更快更清晰的方法解决。**分别找到target的第一次和最后一次出现的位置即可**。最后附上代码

```java
public int[] searchRange(int[] nums, int target) {
    if (nums.length == 0) return new int[]{-1, -1};
    int left = 0, right = nums.length - 1, first;

    //找到第一次出现的位置
    while (left < right) {
        int mid = left + ((right - left) >> 1);
        if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }

    if (nums[left] != target) {
        return new int[]{-1, -1};
    }
    first = left;

    left = 0;
    right = nums.length - 1;

    //找到最后一次出现的位置
    while (left < right) {
        int mid = right - ((right - left) >> 1);
        if (nums[mid] <= target) {
            left = mid;
        } else {
            right = mid - 1;
        }
    }

    return new int[]{first, left};
}
```