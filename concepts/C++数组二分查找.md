# C++ 数组二分查找

> 本文涵盖数组二分查找相关技巧：循环右移后求 k、搜索旋转排序数组（无重复/有重复）。

See also: [[C++双指针]], [[C++算法精选合并版]], [[C++手写代码模板]]

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-数组二分查找.md]

## 循环右移了k位

有一升序数组，现循环右移了k位（k未知），求k的值。

```cpp
#include <iostream>
#include <vector>
using namespace std;
int Fun(vector<int>& nums) {
    if (nums.empty() || nums[0] < nums.back()) return 0;
    int l = 0, r = nums.size() - 1;
    while (l + 1 < r) {
        int mid = l + (r - l) / 2;
        if (nums[l] < nums[mid]) l = mid;
        else if (nums[l] > nums[mid]) r = mid;
        else l++;
    }
    return l + 1;
}
int main() {
    vector<int> nums = {4,5,6,7,8,8,8,8,10,1,2,3,4};
    cout << Fun(nums) << endl;
}
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-数组二分查找.md]

## 搜索旋转排序数组

假设按照升序排序的数组在预先未知的某个点上进行了旋转。
( 例如，数组 [0,1,2,4,5,6,7] 可能变为 [4,5,6,7,0,1,2] )。

搜索一个给定的目标值，如果数组中存在这个目标值，则返回它的索引，否则返回 -1 。
你可以假设数组中不存在重复的元素。
你的算法时间复杂度必须是 O(log n) 级别。

**力扣**：[search-in-rotated-sorted-array](https://leetcode-cn.com/problems/search-in-rotated-sorted-array)

示例 1:
输入: nums = [4,5,6,7,0,1,2], target = 0
输出: 4
示例 2:
输入: nums = [4,5,6,7,0,1,2], target = 3
输出: -1

```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int l = 0, r = nums.size() - 1;
        while (l <= r) {
            int mi = l + (r - l) / 2;
            if (nums[mi] == target) return mi;
            if (nums[l] <= nums[mi]) {
                if (nums[l] <= target && target < nums[mi]) r = mi - 1;
                else l = mi + 1;
            } else {
                if (nums[mi] < target && target <= nums[r]) l = mi + 1;
                else r = mi - 1;
            }
        }
        return -1;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-数组二分查找.md]

## 搜索旋转排序数组 II

已知存在一个按非降序排列的整数数组 nums ，数组中的值不必互不相同。

在传递给函数之前，nums 在预先未知的某个下标 k（0 <= k < nums.length）上进行了 旋转 ，使数组变为 [nums[k], nums[k+1], ..., nums[n-1], nums[0], nums[1], ..., nums[k-1]]（下标 从 0 开始 计数）。例如， [0,1,2,4,4,4,5,6,6,7] 在下标 5 处经旋转后可能变为 [4,5,6,6,7,0,1,2,4,4] 。

给你 旋转后 的数组 nums 和一个整数 target ，请你编写一个函数来判断给定的目标值是否存在于数组中。如果 nums 中存在这个目标值 target ，则返回 true ，否则返回 false 。

你必须尽可能减少整个操作步骤。

**力扣**：[search-in-rotated-sorted-array-ii](https://leetcode-cn.com/problems/search-in-rotated-sorted-array-ii)

示例 1：
输入：nums = [2,5,6,0,0,1,2], target = 0
输出：true

示例 2：
输入：nums = [2,5,6,0,0,1,2], target = 3
输出：false

提示：
1 <= nums.length <= 5000
-104 <= nums[i] <= 104
题目数据保证 nums 在预先未知的某个下标上进行了旋转
-104 <= target <= 104

进阶：
这是 搜索旋转排序数组 的延伸题目，本题中的 nums 可能包含重复元素。
这会影响到程序的时间复杂度吗？会有怎样的影响，为什么？

```cpp
class Solution {
public:
    bool search(vector<int>& nums, int target) {
        int l = 0, r = nums.size() - 1;
        while (l <= r) {
            int mi = l + (r - l) / 2;
            if (nums[mi] == target) return true;
            if (nums[l] < nums[mi]) {// 左半有序
                if (nums[l] <= target && target < nums[mi]) r = mi - 1;
                else l = mi + 1;
            } else if (nums[l] > nums[mi]) {// 右半有序
                if (nums[mi] < target && target <= nums[r]) l = mi + 1;
                else r = mi - 1;
            } else ++l;  // nums[l]==nums[mi] 有重复，无法二分，收缩左边界
        }
        return false;
    }
};
```
/*
假设原数组是{1,2,3,3,3,3,3}，那么旋转之后有可能是{3,3,3,3,3,1,2}，或者{3,1,2,3,3,3,3}，这样的我们判断左边缘和中心的时候都是3，如果我们要寻找1或者2，我们并不知道应该跳向哪一半。解决的办法只能是对边缘移动一步，直到边缘和中间不在相等或者相遇，这就导致了会有不能切去一半的可能。所以最坏情况（比如全部都是一个元素，或者只有一个元素不同于其他元素，而他就在最后一个）就会出现每次移动一步，总共是n步，算法的时间复杂度变成O(n)
*/

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-数组二分查找.md]