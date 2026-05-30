# Go LeetCode 技巧：简单的二分循环

> 本文档收录了 LeetCode 中与二分查找相关的 Go 语言解题技巧，涵盖 Pow(x, n)、x 的平方根、第一个错误的版本、寻找两个有序数组的中位数、寻找旋转排序数组中的最小值（含重复元素）等经典题目。

See also: [[Go_LeetCode技巧]], [[Go语言语法快速入门]], [[Go常用数据结构]]

## Pow(x, n)

[题目链接](https://leetcode-cn.com/problems/powx-n)

迭代：
```go
func myPow(x float64 ,n int) float64 {
	res := float64(1)
	for k := int(math.Abs(float64(n)));k != 0 ;k /=2 {
		if (k % 2) == 1 {
			res *= x
		}
		x *= x
	}
	if n < 0 {
		return float64(1)/res
	}else{
		return res
	}
}
```

递归（时间复杂度 O(log n)，空间复杂度 O(1)）：
```go
func myPow(x float64, n int) float64 {
	if n == 0 {
		return 1
	}
	if n == 1 {
		return x
	}
	if n < 0 {
		n = -n
		x = 1 / x
	}
	tmp := myPow(x, n/2)
	if n % 2 == 0 {
		return tmp * tmp
	}
	return tmp * tmp * x
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-简单的二分循环.md]

## x 的平方根

[题目链接](https://leetcode-cn.com/problems/sqrtx)

实现 int sqrt(int x) 函数。计算并返回 x 的平方根，其中 x 是非负整数。由于返回类型是整数，结果只保留整数的部分，小数部分将被舍去。

示例 1：
输入: 4
输出: 2

示例 2：
输入: 8
输出: 2
说明: 8 的平方根是 2.82842..., 由于返回类型是整数，小数部分将被舍去。

```go
func mySqrt(x int) int {
	b,e,mid,res:= float64(0),float64(x),float64(1),float64(1)
	for math.Abs(x - res) > 0.000001 {
		mid = (b + e) /2
		res = mid * mid
		if res > x {
			e = mid 
		}else {
			b = mid 
		}
	}
	return int(mid)
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-简单的二分循环.md]

## 第一个错误的版本

[题目链接](https://leetcode-cn.com/problems/first-bad-version)

```go
/** 
 * Forward declaration of isBadVersion API.
 * @param   version   your guess about first bad version
 * @return 	      true if current version is bad 
 *		      false if current version is good
 * func isBadVersion(version int) bool;
 */

func firstBadVersion(n int) int {
     if n < 1 {return -1}
     l,r:= 1,n
     for l + 1 < r {
         mi := l + (r - l)/2
         if isBadVersion(mi) {
             r = mi
         }else{
             l = mi
         }
     } 
     if isBadVersion(l) {
         return l
     }else if isBadVersion(r) {
         return r
     }else{
         return -1
     }
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-简单的二分循环.md]

## 寻找两个有序数组的中位数

[题目链接](https://leetcode-cn.com/problems/median-of-two-sorted-arrays)

给定两个大小分别为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。请你找出并返回这两个正序数组的中位数。算法的时间复杂度应该为 O(log (m+n))。

示例 1：
输入：nums1 = [1,3], nums2 = [2]
输出：2.00000
解释：合并数组 = [1,2,3] ，中位数 2

示例 2：
输入：nums1 = [1,2], nums2 = [3,4]
输出：2.50000
解释：合并数组 = [1,2,3,4] ，中位数 (2 + 3) / 2 = 2.5

提示：
- nums1.length == m
- nums2.length == n
- 0 <= m <= 1000
- 0 <= n <= 1000
- 1 <= m + n <= 2000
- -10^6 <= nums1[i], nums2[i] <= 10^6

关键在于在nums1中一直寻找mid1，使mid1的左右两数跟mid2的左右两数是交错的（到边界也算），之后就用mid1、mid2两边4个数来算中位数。

二分查，复杂度 O(log (m+n))：
```go
func findMedianSortedArrays(nums1 []int, nums2 []int) float64 {
	if len(nums1) > len(nums2) {
		return findMedianSortedArrays(nums2, nums1)
	}
	left1, right1, length, mid1, mid2 := 0, len(nums1),len(nums1)+len(nums2), 0, 0
	for left1 <= right1 {
		mid1 = left1 + (right1-left1)>>1
		mid2 = (length + 1)/2 - mid1
		if mid1 > 0 && nums1[mid1-1] > nums2[mid2] {
			right1 = mid1 - 1
		} else if mid1 != len(nums1) && nums1[mid1] < nums2[mid2-1] {
			left1 = mid1 + 1
		} else {
			break
		}
	}
    var L,R int
    if mid1 == 0 {
        L = nums2[mid2-1]  
    }else if mid2 == 0 {
        L = nums1[mid1-1]
    }else{
        L = int(math.Max(float64(nums2[mid2-1]),float64(nums1[mid1-1])) )
    }
	if length & 1 == 1 {
		return float64(L)
	}
    if mid1 == len(nums1) {
        R = nums2[mid2]  
    }else if mid2 == len(nums2) {
        R = nums1[mid1]
    }else{
        R = int(math.Min(float64(nums2[mid2]),float64(nums1[mid1]))) 
    }
	return float64(L+R) / 2.0
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-简单的二分循环.md]

## 寻找旋转排序数组中的最小值

[题目链接](https://leetcode.cn/problems/find-minimum-in-rotated-sorted-array/)

已知一个长度为 n 的数组，预先按照升序排列，经由 1 到 n 次旋转后，得到输入数组。例如，原数组 nums = [0,1,2,4,5,6,7] 在变化后可能得到：
- 若旋转 4 次，则可以得到 [4,5,6,7,0,1,2]
- 若旋转 7 次，则可以得到 [0,1,2,4,5,6,7]

注意，数组 [a[0], a[1], a[2], ..., a[n-1]] 旋转一次的结果为数组 [a[n-1], a[0], a[1], a[2], ..., a[n-2]] 。

给你一个元素值互不相同的数组 nums ，它原来是一个升序排列的数组，并按上述情形进行了多次旋转。请你找出并返回数组中的最小元素。你必须设计一个时间复杂度为 O(log n) 的算法解决此问题。

示例 1：
输入：nums = [3,4,5,1,2]
输出：1
解释：原数组为 [1,2,3,4,5] ，旋转 3 次得到输入数组。

示例 2：
输入：nums = [4,5,6,7,0,1,2]
输出：0
解释：原数组为 [0,1,2,4,5,6,7] ，旋转 4 次得到输入数组。

示例 3：
输入：nums = [11,13,15,17]
输出：11
解释：原数组为 [11,13,15,17] ，旋转 4 次得到输入数组。

提示：
- n == nums.length
- 1 <= n <= 5000
- -5000 <= nums[i] <= 5000
- nums 中的所有整数互不相同
- nums 原来是一个升序排序的数组，并进行了 1 至 n 次旋转

```go
func findMin(nums []int) int {
	low, high := 0, len(nums)-1
	for low < high {
		if nums[low] < nums[high] {
			return nums[low]
		}
		mid := low + (high-low)>>1
		if nums[mid] >= nums[low] {
			low = mid + 1
		} else {
			high = mid
		}
	}
	return nums[low]
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-简单的二分循环.md]

## 寻找旋转排序数组中的最小值 II

[题目链接](https://leetcode.cn/problems/find-minimum-in-rotated-sorted-array-ii/)

已知一个长度为 n 的数组，预先按照升序排列，经由 1 到 n 次旋转后，得到输入数组。例如，原数组 nums = [0,1,4,4,5,6,7] 在变化后可能得到：
- 若旋转 4 次，则可以得到 [4,5,6,7,0,1,4]
- 若旋转 7 次，则可以得到 [0,1,4,4,5,6,7]

注意，数组 [a[0], a[1], a[2], ..., a[n-1]] 旋转一次的结果为数组 [a[n-1], a[0], a[1], a[2], ..., a[n-2]] 。

给你一个可能存在重复元素值的数组 nums ，它原来是一个升序排列的数组，并按上述情形进行了多次旋转。请你找出并返回数组中的最小元素。你必须尽可能减少整个过程的操作步骤。

示例 1：
输入：nums = [1,3,5]
输出：1

示例 2：
输入：nums = [2,2,2,0,1]
输出：0

提示：
- n == nums.length
- 1 <= n <= 5000
- -5000 <= nums[i] <= 5000
- nums 原来是一个升序排序的数组，并进行了 1 至 n 次旋转

进阶：这道题与寻找旋转排序数组中的最小值类似，但 nums 可能包含重复元素。允许重复会影响算法的时间复杂度吗？会如何影响，为什么？

```go
func findMin(nums []int) int {
    low, high := 0, len(nums)-1
	for low < high {
		if nums[low] < nums[high] {
			return nums[low]
		}
		mid := low + (high-low)>>1
		if nums[mid] > nums[low] {
			low = mid + 1
		} else if nums[mid] < nums[low] { 
			high = mid
		} else {
			for nums[mid] == nums[low] && mid >= low {low++}
        }
	}
	return nums[low]
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-简单的二分循环.md]