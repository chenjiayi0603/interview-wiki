# C++ 数组从两边往中间收拢

## 盛最多水的容器

给定一个长度为 n 的整数数组 height。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i])。找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。返回容器可以储存的最大水量。

**力扣**：[container-with-most-water](https://leetcode-cn.com/problems/container-with-most-water)

```cpp
class Solution {
public:
    int maxArea(vector<int>& height) {
        int left(0), right(height.size()-1);
        int res = 0;
        while(left < right){
            res = max(res, min(height[left], height[right])*(right-left));
            if(height[left] < height[right]) left++;
            else right--;
        }
        return res;
    }
};
```

## 最接近的三数之和

给定一个包括 n 个整数的数组 nums 和 一个目标值 target。找出 nums 中的三个整数，使得它们的和与 target 最接近。返回这三个数的和。

**力扣**：[3sum-closest](https://leetcode-cn.com/problems/3sum-closest)

```cpp
int threeSumClosest(vector<int>& nums, int target)
{
    if (nums.size() < 3) return 0;
    sort(nums.begin(),nums.end());
    int res(0),diff(INT_MAX);
    for(int i = 0;i < nums.size()-2;++i)
    {
        int left = i + 1,right = nums.size()-1;
        while(left < right)
        {
            int s = nums[i] + nums[left] + nums[right];
            if (s == target) return s;
            int d = abs(target - s);
            if (d < diff)
            {
                res = s;
                diff = d;
            }
            if (s < target) ++left;
            else --right;
        }
    }
    return res;
}
```

搜索过程的适度优化：

```cpp
int threeSumClosest(vector<int>& nums, int target)
{
    if (nums.size() < 3) return 0;
    sort(nums.begin(),nums.end());
    int res(0),diff(INT_MAX);
    for(int i = 0;i < nums.size()-2;++i)
    {
        int left = i + 1,right = nums.size()-1;
        while(left < right)
        {
            int s = nums[i] + nums[left] + nums[right];
            if (s == target) return s;
            int d = abs(target - s);
            if (d < diff)
            {
                res = s;
                diff = d;
            }
            if (s < target)
            {
                ++left;
                while (left < right && nums[left] == nums[left - 1]) ++left;
            }
            else
            {
                --right;
                while (left < right && nums[right] == nums[right + 1]) --right;
            }
        }
    }
    return res;
}
```

## 搜索二维矩阵 II

编写一个高效的算法来搜索 m x n 矩阵 matrix 中的一个目标值 target。该矩阵具有以下特性：每行的元素从左到右升序排列，每列的元素从上到下升序排列。

贪心：二维矩阵从左下角开始，往上或者往右搜索。

**力扣**：[search-a-2d-matrix-ii](https://leetcode-cn.com/problems/search-a-2d-matrix-ii)

```cpp
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        if (matrix.size() == 0 || matrix[0].size() == 0) return false;
        int m = matrix.size(), n = matrix[0].size();
        int x = m - 1, y = 0;
        for(; x >= 0 && y < n;)
        {
            if (matrix[x][y] < target) ++y;
            else if (matrix[x][y] > target) --x;
            else return true;
        }
        return false;
    }
};
```

## 最短无序连续子数组

给你一个整数数组 nums，你需要找出一个连续子数组，如果对这个子数组进行升序排序，那么整个数组都会变为升序排序。请你找出符合题意的最短子数组，并输出它的长度。

**力扣**：[shortest-unsorted-continuous-subarray](https://leetcode-cn.com/problems/shortest-unsorted-continuous-subarray)

```cpp
class Solution {
public:
    int findUnsortedSubarray(vector<int>& nums) {
        if (nums.size() == 0) return 0;
        int left = 0, right = -1;
        for (int i = 0, maxNum = nums[i]; i < nums.size(); ++i) {
            if (nums[i] >= maxNum) {
                maxNum = nums[i];
            } else {
                right = i;
            }
        }
        for (int i = nums.size()-1, minNum = nums[i]; i >= 0; --i) {
            if (nums[i] <= minNum) {
                minNum = nums[i];
            } else {
                left = i;
            }
        }
        return right - left + 1;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-数组从两边往中间收拢.md]