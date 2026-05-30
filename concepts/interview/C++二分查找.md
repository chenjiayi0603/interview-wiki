# C++ 二分查找

> 本文涵盖二分查找的经典题型与 C++ 实现，包括基础二分、lower_bound、旋转数组、第一个错误版本等。

See also: [[C++算法精选合并版]], [[C++手写代码模板]], [[C++高频面试问题]]

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## 基础二分

升序数组中查找 target，返回下标，未找到返回 -1。

```cpp
// 基础二分：闭区间 [l,r]，未找到返回 -1
int binarySearch(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == target) return m;
        nums[m] < target ? l = m + 1 : r = m - 1;
    }
    return -1;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## lower_bound

升序数组中找第一个 >= target 的下标。

```cpp
// lower_bound：第一个 >= target 的下标，左闭右开 [l,r)
int lowerBound(vector<int>& nums, int target) {
    int l = 0, r = nums.size();
    while (l < r) {
        int m = l + (r - l) / 2;
        // 如果中值小于目标，左边界右移，否则右边界左移
        if (nums[m] < target) l = m + 1; else r = m;  
    }
    return l;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## 搜索旋转排序数组

升序数组在某点旋转后，查找 target 的下标。

**力扣**：[search-in-rotated-sorted-array](https://leetcode-cn.com/problems/search-in-rotated-sorted-array)

```cpp
// 搜索旋转排序数组：先判 mid 在左半还是右半有序段，再二分
int search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == target) return m;
        if (nums[m] < nums[r]) {
            if (nums[m] < target && target <= nums[r]) l = m + 1;
            else r = m - 1;
        } else {
            if (nums[l] <= target && target < nums[m]) r = m - 1;
            else l = m + 1;
        }
    }
    return -1;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## 第一个错误版本

版本号 1~n，isBadVersion(i) 表示 i 及之后全坏，找第一个坏的版本号。

**力扣**：[first-bad-version](https://leetcode-cn.com/problems/first-bad-version)

```cpp
// 第一个错误版本：二分找第一个 true，r 存候选
int firstBadVersion(int n) {
    int l = 1, r = n, bad = -1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (isBadVersion(m)) {
            bad = m;
            r = m - 1;
        } else {
            l = m + 1;
        }
    }
    return bad;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## 搜索旋转排序数组 II（有重复）

旋转数组可能含重复元素，判断 target 是否存在。

**力扣**：[search-in-rotated-sorted-array-ii](https://leetcode-cn.com/problems/search-in-rotated-sorted-array-ii)

```cpp
bool search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == target) return true;
        if (nums[l] < nums[m]) {
            if (nums[l] <= target && target < nums[m]) r = m - 1;
            else l = m + 1;
        } else if (nums[l] > nums[m]) {
            if (nums[m] < target && target <= nums[r]) l = m + 1;
            else r = m - 1;
        } else l++;  // nums[l]==nums[m] 无法二分，收缩
    }
    return false;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## x 的平方根

求非负整数 x 的平方根（向下取整）。

**力扣**：[sqrtx](https://leetcode-cn.com/problems/sqrtx)

```cpp
int mySqrt(int x) {
    if (x <= 1) return x;
    long l = 1, r = x;
    while (l < r) {
        long m = l + (r - l + 1) / 2;
        if (m * m <= x) l = m;
        else r = m - 1;
    }
    return (int)l;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## 有效的完全平方数

判断 num 是否为完全平方数。

**力扣**：[valid-perfect-square](https://leetcode-cn.com/problems/valid-perfect-square)

```cpp
bool isPerfectSquare(int num) {
    long l = 1, r = num;
    while (l <= r) {
        long m = l + (r - l) / 2, t = m * m;
        if (t == num) return true;
        t < num ? l = m + 1 : r = m - 1;
    }
    return false;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]

## 寻找两个有序数组的中位数

两升序数组，求合并后的中位数，要求 O(log(m+n))。

**力扣**：[median-of-two-sorted-arrays](https://leetcode-cn.com/problems/median-of-two-sorted-arrays)

```cpp
double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
    if (nums1.size() > nums2.size()) return findMedianSortedArrays(nums2, nums1);
    int len = nums1.size() + nums2.size();
    int l = 0, r = nums1.size();
    double res = 0;
    while (l <= r) {
        int m1 = l + (r - l) / 2;
        int m2 = (len + 1) / 2 - m1;
        if (m1 > 0 && m2 < nums2.size() && nums1[m1-1] > nums2[m2]) r = m1 - 1;
        else if (m1 < nums1.size() && m2 > 0 && nums1[m1] < nums2[m2-1]) l = m1 + 1;
        else {
            int L1 = m1 > 0 ? nums1[m1-1] : INT_MIN;
            int R1 = m1 < nums1.size() ? nums1[m1] : INT_MAX;
            int L2 = m2 > 0 ? nums2[m2-1] : INT_MIN;
            int R2 = m2 < nums2.size() ? nums2[m2] : INT_MAX;
            res = (len % 2) ? max(L1, L2) : (max(L1, L2) + min(R1, R2)) / 2.0;
            break;
        }
    }
    return res;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-八、二分查找.md]