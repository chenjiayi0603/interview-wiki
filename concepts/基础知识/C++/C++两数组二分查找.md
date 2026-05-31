# C++ 两数组二分查找

## 寻找两个有序数组的中位数

[题目链接](https://leetcode-cn.com/problems/median-of-two-sorted-arrays)

给定两个大小分别为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。请你找出并返回这两个正序数组的中位数。

算法的时间复杂度应该为 O(log (m+n)) 。

示例 1：
输入：nums1 = [1,3], nums2 = [2]
输出：2.00000
解释：合并数组 = [1,2,3] ，中位数 2

示例 2：
输入：nums1 = [1,2], nums2 = [3,4]
输出：2.50000
解释：合并数组 = [1,2,3,4] ，中位数 (2 + 3) / 2 = 2.5

提示：
nums1.length == m
nums2.length == n
0 <= m <= 1000
0 <= n <= 1000
1 <= m + n <= 2000
-106 <= nums1[i], nums2[i] <= 106

关键在于在nums1中一直寻找mid1，使mid1的左右两数跟mid2的左右两数是交错的（到边界也算）,之后就用mid1、mid2两边4个数来算中位数

二分查,复杂度log(m + n)

```cpp
class Solution {
public:
    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
        if(nums1.size() > nums2.size()) return findMedianSortedArrays(nums2,nums1);//保证 mid2 的访问
        int length = nums1.size() + nums2.size();
        int left1 = 0, right1 = nums1.size(); 
        double result = 0;
        while (left1 <= right1)   //mid1二分查找
        {
            int mid1 = left1 + (right1 - left1)/2;
            int mid2 = (length+1)/2 - mid1;
            if(mid1 > 0 && nums1[mid1-1] > nums2[mid2]) right1 = mid1-1;
            else if(mid1 != nums1.size() && nums1[mid1] < nums2[mid2-1]) left1 = mid1+1;
            else{
                int L1 = mid1 > 0 ?nums1[mid1-1]:INT_MIN;
                int R1 = mid1 == nums1.size() ?INT_MAX:nums1[mid1];
                int L2 = mid2 > 0 ?nums2[mid2-1]:INT_MIN;
                int R2 = mid2 == nums2.size() ?INT_MAX:nums2[mid2];
                result = (length % 2 != 0) ? max(L1,L2):(max(L1,L2)+min(R1,R2))/2.0;
                break;
            }
        }
        return result;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-两数组的二分查找.md]