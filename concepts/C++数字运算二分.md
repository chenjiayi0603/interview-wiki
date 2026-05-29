# C++ 数字运算有关的二分

> 本文涵盖数字运算相关的二分查找算法，包括第一个错误的版本、x 的平方根、Pow(x, n)、有效的完全平方数等经典题目。

See also: [[C++双指针]], [[C++DFS回溯]], [[C++链表]]

## 第一个错误的版本

你是产品经理，目前正在带领一个团队开发新的产品。不幸的是，你的产品的最新版本没有通过质量检测。由于每个版本都是基于之前的版本开发的，所以错误的版本之后的所有版本都是错的。

假设你有 n 个版本 [1, 2, ..., n]，你想找出导致之后所有版本出错的第一个错误的版本。

你可以通过调用 bool isBadVersion(version) 接口来判断版本号 version 是否在单元测试中出错。实现一个函数来查找第一个错误的版本。你应该尽量减少对调用 API 的次数。

**力扣**：[first-bad-version](https://leetcode-cn.com/problems/first-bad-version)

示例 1：
输入：n = 5, bad = 4
输出：4
解释：
调用 isBadVersion(3) -> false 
调用 isBadVersion(5) -> true 
调用 isBadVersion(4) -> true
所以，4 是第一个错误的版本。

示例 2：
输入：n = 1, bad = 1
输出：1

提示：1 <= bad <= n <= 231 - 1

```cpp
class Solution {
public:
    int firstBadVersion(int n) {
        if (n < 1) return -1;
        int l = 1, r = n, mi = -1, bad = -1;
        while (l <= r) {
            mi = l + (r - l) / 2;
            if (isBadVersion(mi)) { bad = mi; r = mi - 1; }
            else l = mi + 1;
        }
        return bad;
    }
};
```

## x 的平方根

实现 int sqrt(int x) 函数。计算并返回 x 的平方根，其中 x 是非负整数。由于返回类型是整数，结果只保留整数的部分，小数部分将被舍去。

**力扣**：[sqrtx](https://leetcode-cn.com/problems/sqrtx)

示例 1：
输入: 4
输出: 2

示例 2：
输入: 8
输出: 2
说明: 8 的平方根是 2.82842..., 由于返回类型是整数，小数部分将被舍去。

```cpp
class Solution {
public:
    int mySqrt(int x) {
        if (x <= 1) return x;
        long l = 1, r = x;
        while (l < r) {
            long mid = l + (r - l + 1) / 2;
            if (mid * mid <= x) l = mid;
            else r = mid - 1;
        }
        return (int)l;
    }
};
```

## Pow(x, n)

实现 pow(x, n) ，即计算 x 的 n 次幂函数（即，x^n）。

**力扣**：[powx-n](https://leetcode-cn.com/problems/powx-n)

示例 1：
输入：x = 2.00000, n = 10
输出：1024.00000

示例 2：
输入：x = 2.10000, n = 3
输出：9.26100

示例 3：
输入：x = 2.00000, n = -2
输出：0.25000
解释：2-2 = 1/22 = 1/4 = 0.25

```cpp
class Solution {
public:
    double myPow(double x, int n) {
        double res(1.0);
        for (int k = abs(n); k != 0; k /= 2) {
            if (k % 2 == 1) {
                res *= x;
            }
            x *= x;
        }
        return n < 0 ? 1 / res : res;
    }
};
```

## 有效的完全平方数

给定一个正整数 num，编写一个函数，如果 num 是一个完全平方数，则返回 true，否则返回 false。

进阶：不要使用任何内置的库函数，如 sqrt。

**力扣**：[valid-perfect-square](https://leetcode-cn.com/problems/valid-perfect-square)

示例 1：
输入：num = 16
输出：true

示例 2：
输入：num = 14
输出：false

```cpp
class Solution {
public:
    bool isPerfectSquare(int num) {
        int l = 1, r = num;
        long mid, tmp;
        while (l <= r) {
            mid = l + (r - l) / 2;
            tmp = mid * mid;
            if (tmp == num) return true;
            else if (tmp < num) l = mid + 1;
            else r = mid - 1;
        }
        return false;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-数字运算有关的二分.md]