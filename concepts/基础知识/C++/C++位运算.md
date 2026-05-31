# C++ 位运算

> 本文涵盖位运算的经典题型与 C++ 实现，包括异或找单数、摩尔投票找众数、2 的幂判断、数字范围按位与、数字的补数等。

See also: [[C++算法精选合并版]], [[C++手写代码模板]], [[C++高频面试问题]]

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]

## 只出现一次的数字

数组中除某数外均出现两次，找只出现一次的数。

**力扣**：[single-number](https://leetcode-cn.com/problems/single-number)

```cpp
// 只出现一次的数字：a^a=0，全部异或即得单数
int singleNumber(vector<int>& nums) {
    int res = 0;
    for (int x : nums) res ^= x;
    return res;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]

## 多数元素

数组中存在出现次数 > n/2 的元素，找出该众数。

**力扣**：[majority-element](https://leetcode-cn.com/problems/majority-element)

```cpp
// 多数元素：摩尔投票，不同则抵消，最后剩的即众数
int majorityElement(vector<int>& nums) {
    int cnt = 0, cand = 0;
    for (int x : nums) {
        if (cnt == 0) { cand = x; cnt = 1; }
        else cnt += (x == cand ? 1 : -1);
    }
    return cand;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]

## 2 的幂

判断整数 n 是否为 2 的幂（n>0 且二进制仅一个 1）。

```cpp
// 2的幂：n > 0 && !(n & (n - 1))
// 取最低的1：a & (-a)
// 去掉最低的1：n = n & (n - 1)
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]

## 只出现一次的数字 III

恰好两个元素出现一次，其余两次，找出这两个数。

**力扣**：[single-number-iii](https://leetcode-cn.com/problems/single-number-iii)

```cpp
vector<int> singleNumber(vector<int>& nums) {
    long x = 0;
    for (int n : nums) x ^= n;
    long bit = x & (-x);  // 取最低的 1 区分两数
    vector<int> res(2, 0);
    for (int n : nums)
        (n & bit) ? res[1] ^= n : res[0] ^= n;
    return res;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]

## 只出现一次的数字 II

恰好一个元素出现一次，其余三次。

**力扣**：[single-number-ii](https://leetcode-cn.com/problems/single-number-ii)

```cpp
int singleNumber(vector<int>& nums) {
    int one = 0, two = 0;
    for (int x : nums) {
        two |= one & x;
        one ^= x;
        int three = one & two;
        one &= ~three;
        two &= ~three;
    }
    return one;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]

## 数字范围按位与

[left, right] 内所有数按位与的结果。

**力扣**：[bitwise-and-of-numbers-range](https://leetcode-cn.com/problems/bitwise-and-of-numbers-range)

```cpp
int rangeBitwiseAnd(int m, int n) {
    while (n > m) n &= n - 1;  // 抹掉低位 1
    return n;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]

## 数字的补数

对二进制取反（0 变 1，1 变 0）后转十进制。

**力扣**：[number-complement](https://leetcode-cn.com/problems/number-complement)

```cpp
int findComplement(int num) {
    int c = 0;
    for (int n = 0; num > 0; num >>= 1, n++)
        if (!(num & 1)) c |= 1 << n;
    return c;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十二、位运算.md]