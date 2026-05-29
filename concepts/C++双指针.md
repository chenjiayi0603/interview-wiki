# C++ 双指针

> 双指针（Two Pointers）是一种常用的算法技巧，通过使用两个指针在数组或链表中协同移动来解决问题，通常分为左右指针和快慢指针两类。

See also: [[C++算法精选合并版]], [[C++手写代码模板]], [[C++高频面试问题]]

[src: raw/ingested/2技术/算法/C++算法精选合并版-二、双指针.md]

## 2.1 左右指针

| 题目 | 题目说明 | 思路 | 链接 |
| :-- | :-- | :-- | :-- |
| 两数之和（有序） | 升序数组中找两数之和等于 target，返回下标 | 左右夹逼，和大了右移、和小了左移 | - |
| 三数之和 | 数组中找所有和为 0 的不重复三元组 | 固定一端+双指针，找和为 0 的三元组 | [3sum](https://leetcode-cn.com/problems/3sum) |
| 盛水最多的容器 | n 条垂线，选两条与 x 轴围成容器，求最大盛水量 | 矮边移动，面积=min(h)×宽 | [container-with-most-water](https://leetcode-cn.com/problems/container-with-most-water) |
| 回文串判断 | 判断字符串正反读是否相同，忽略大小写和非字母数字 | 左右向中间，忽略非字母数字 | [valid-palindrome](https://leetcode-cn.com/problems/valid-palindrome) |
| 接雨水 | 柱状图高度数组，求能接多少雨水 | 双指针+维护左右最大高度，较小端可存水 | [trapping-rain-water](https://leetcode.cn/problems/trapping-rain-water/description/) |
| 最接近的三数之和 | 找和最接近 target 的三元组 | 固定一端+双指针夹逼，维护最小 diff | [3sum-closest](https://leetcode-cn.com/problems/3sum-closest) |
| 搜索二维矩阵 II | 每行每列升序的矩阵中找 target | 从左下角开始，小则右移、大则上移 | [search-a-2d-matrix-ii](https://leetcode-cn.com/problems/search-a-2d-matrix-ii/) |
| 最短无序连续子数组 | 找最短子数组，排序后整个数组有序 | 从左找 right（小于左边 max），从右找 left（大于右边 min） | [shortest-unsorted-continuous-subarray](https://leetcode-cn.com/problems/shortest-unsorted-continuous-subarray) |

### 两数之和（有序）

升序数组中找两数之和等于 target，返回下标。

**力扣**：[two-sum-ii-input-array-is-sorted](https://leetcode-cn.com/problems/two-sum-ii-input-array-is-sorted)

```cpp
// 两数之和（有序）：左右夹逼，和大了右移，和小了左移
vector<int> twoSum(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l < r) {
        int s = nums[l] + nums[r];
        if (s == target) return {l, r};
        s < target ? l++ : r--;
    }
    return {-1, -1};
}
```

### 三数之和

数组中找所有和为 0 的不重复三元组 [a,b,c]。

**力扣**：[3sum](https://leetcode-cn.com/problems/3sum)

```cpp
// 三数之和：固定一端 i，双指针 l、r 在 [i+1,n-1] 内夹逼
vector<vector<int>> threeSum(vector<int>& nums) {
    vector<vector<int>> res;
    sort(nums.begin(), nums.end());
    for (int i = 0; i < (int)nums.size() - 2; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;
        int l = i + 1, r = nums.size() - 1;
        while (l < r) {
            int s = nums[i] + nums[l] + nums[r];
            if (s == 0) {
                res.push_back({nums[i], nums[l++], nums[r--]});
                while (l < r && nums[l] == nums[l-1]) l++;
                while (l < r && nums[r] == nums[r+1]) r--;
            } else s < 0 ? l++ : r--;
        }
    }
    return res;
}
```

### 盛水最多的容器

n 条垂线高度 h[i]，选两条与 x 轴围成容器，求最大盛水量。

**力扣**：[container-with-most-water](https://leetcode-cn.com/problems/container-with-most-water)

```cpp
// 盛水最多的容器：面积=min(h[l],h[r])*(r-l)，矮边移动才可能增大
int maxArea(vector<int>& h) {
    int l = 0, r = h.size() - 1, res = 0;
    while (l < r) {
        res = max(res, min(h[l], h[r]) * (r - l));
        h[l] < h[r] ? l++ : r--;
    }
    return res;
}
```

### 接雨水

柱状图高度数组，每格宽度 1，求能接多少雨水。

**力扣**：[trapping-rain-water](https://leetcode.cn/problems/trapping-rain-water)

```cpp
// 接雨水：移动较小端，该端可存水=该端最大高度-当前高度
int trap(vector<int>& height) {
    int l = 0, r = height.size() - 1, lm = 0, rm = 0, res = 0;
    while (l < r) {
        if (height[l] < height[r]) {
            lm = max(lm, height[l]);
            res += lm - height[l++];
        } else {
            rm = max(rm, height[r]);
            res += rm - height[r--];
        }
    }
    return res;
}
```

### 最接近的三数之和

找和最接近 target 的三元组，返回该和。

**力扣**：[3sum-closest](https://leetcode-cn.com/problems/3sum-closest)

```cpp
int threeSumClosest(vector<int>& nums, int target) {
    if (nums.size() < 3) return 0;
    sort(nums.begin(), nums.end());
    int res = 0, diff = INT_MAX;
    for (int i = 0; i < (int)nums.size() - 2; i++) {
        int l = i + 1, r = nums.size() - 1;
        while (l < r) {
            int s = nums[i] + nums[l] + nums[r];
            if (s == target) return s;
            if (abs(target - s) < diff) { res = s; diff = abs(target - s); }
            s < target ? l++ : r--;
        }
    }
    return res;
}
```

### 搜索二维矩阵 II

每行从左到右、每列从上到下升序，判断 target 是否存在。

**力扣**：[search-a-2d-matrix-ii](https://leetcode-cn.com/problems/search-a-2d-matrix-ii)

```cpp
bool searchMatrix(vector<vector<int>>& matrix, int target) {
    if (matrix.empty() || matrix[0].empty()) return false;
    int x = matrix.size() - 1, y = 0;  // 左下角
    while (x >= 0 && y < matrix[0].size()) {
        if (matrix[x][y] < target) y++;
        else if (matrix[x][y] > target) x--;
        else return true;
    }
    return false;
}
```

### 最短无序连续子数组

找最短连续子数组，对其排序后整个数组升序，返回该子数组长度。

**力扣**：[shortest-unsorted-continuous-subarray](https://leetcode-cn.com/problems/shortest-unsorted-continuous-subarray)

```cpp
int findUnsortedSubarray(vector<int>& nums) {
    if (nums.empty()) return 0;
    int left = 0, right = -1;
    for (int i = 0, maxNum = nums[0]; i < nums.size(); i++) {
        if (nums[i] >= maxNum) maxNum = nums[i];
        else right = i;  // 从左往右：小于左边 max 的位置
    }
    for (int i = nums.size() - 1, minNum = nums.back(); i >= 0; i--) {
        if (nums[i] <= minNum) minNum = nums[i];
        else left = i;   // 从右往左：大于右边 min 的位置
    }
    return right - left + 1;
}
```

## 2.2 快慢指针

**说明**：slow 写指针、fast 读指针；链表判环用相遇法；环入口用重置快指针同速走。

### 删除有序数组重复项

升序数组中原地去重，使每个元素只出现一次，返回新长度。

**力扣**：[remove-duplicates-from-sorted-array](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array)

```cpp
// 删除有序数组重复项：slow 写指针，fast 读指针
int removeDuplicates(vector<int>& nums) {
    if (nums.empty()) return 0;
    int slow = 0;
    for (int fast = 1; fast < nums.size(); fast++)
        if (nums[fast] != nums[slow]) nums[++slow] = nums[fast];
    return slow + 1;
}
```

### 链表判环

判断链表是否存在环（尾节点指向前面某节点）。

**力扣**：[linked-list-cycle](https://leetcode-cn.com/problems/linked-list-cycle)

```cpp
// 链表判环：快慢指针，相遇则有环
bool hasCycle(ListNode* head) {
    ListNode *s = head, *f = head;
    while (f && f->next) {
        s = s->next; f = f->next->next;
        if (s == f) return true;
    }
    return false;
}
```

### 环形链表入口

若链表有环，返回环的入口节点；无环返回 null。

**力扣**：[linked-list-cycle-ii](https://leetcode-cn.com/problems/linked-list-cycle-ii)

```cpp
// 环形链表入口：相遇后，快指针回 head，同速走，再相遇即入口
ListNode* detectCycle(ListNode* head) {
    if (!head || !head->next) return nullptr;
    ListNode *s = head, *f = head;
    while (f && f->next) {
        s = s->next; f = f->next->next;
        if (s == f) break;
    }
    if (!f || !f->next) return nullptr;
    f = head;  // 重置快指针，同步走找入口
    while (f != s) { f = f->next; s = s->next; }
    return s;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-二、双指针.md]

## 补充：最接近的三数之和（腾讯精选练习50题）

给定一个包括 n 个整数的数组 nums 和 一个目标值 target。找出 nums 中的三个整数，使得它们的和与 target 最接近。返回这三个数的和。假定每组输入只存在唯一答案。

**力扣**：[3sum-closest](https://leetcode-cn.com/problems/3sum-closest)

示例：
输入：nums = [-1,2,1,-4], target = 1
输出：2
解释：与 target 最接近的和是 2 (-1 + 2 + 1 = 2) 。

提示：
- 3 <= nums.length <= 10^3
- -10^3 <= nums[i] <= 10^3
- -10^4 <= target <= 10^4

### 解法一（基础双指针）

```cpp
class Solution {
public:
    int threeSumClosest(vector<int>& nums, int target) {
        if (nums.size() < 3) return 0;
        sort(nums.begin(), nums.end());
        int res = 0, diff = INT_MAX;
        for (int i = 0; i < nums.size() - 2; ++i) {
            int left = i + 1, right = nums.size() - 1;
            while (left < right) {
                int s = nums[i] + nums[left] + nums[right];
                if (s == target) return s;
                int d = abs(target - s);
                if (d < diff) {
                    res = s;
                    diff = d;
                }
                if (s < target) ++left;
                else --right;
            }
        }
        return res;
    }
};
```

### 解法二（优化去重）

```cpp
class Solution {
public:
    int threeSumClosest(vector<int>& nums, int target) {
        if (nums.size() < 3) return 0;
        sort(nums.begin(), nums.end());
        int res = 0, diff = INT_MAX;
        for (int i = 0; i < nums.size() - 2; ++i) {
            int left = i + 1, right = nums.size() - 1;
            while (left < right) {
                int s = nums[i] + nums[left] + nums[right];
                if (s == target) return s;
                int d = abs(target - s);
                if (d < diff) {
                    res = s;
                    diff = d;
                }
                if (s < target) {
                    ++left;
                    while (left < right && nums[left] == nums[left - 1]) ++left;
                } else {
                    --right;
                    while (left < right && nums[right] == nums[right + 1]) --right;
                }
            }
        }
        return res;
    }
};
```

第二种性能要优化一些：
- 第一种 24 ms 9.4 MB Cpp
- 第二种 12 ms 10.2 MB Cpp

[src: raw/ingested/2技术/算法/cpp_leetcode_腾讯精选练习50题-[最接近的三数之和](https---leetcode-cn.com-problems-3sum-closest).md]