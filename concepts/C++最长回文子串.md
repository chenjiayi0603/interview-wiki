# C++ 最长回文子串

## 问题描述

给定一个字符串 s，找到 s 中最长的回文子串。你可以假设 s 的最大长度为 1000。

示例 1：
输入: "babad"
输出: "bab"
注意: "aba" 也是一个有效答案。

示例 2：
输入: "cbbd"
输出: "bb"

## 解法一：中心扩展法

```cpp
class Solution {
public:
    string longestPalindrome(string s) {
        int start = 0, maxLen = 1, n = s.size();
        auto expand = [&](int l, int r) {
            while (l >= 0 && r < n && s[l] == s[r]) l--, r++;
            if (r - l - 1 > maxLen) start = l + 1, maxLen = r - l - 1;
        };
        for (int i = 0; i < n-1; ++i) expand(i, i), expand(i, i+1);
        return s.substr(start, maxLen);
    }
};
```

复杂度 O(N²)

## 解法二：动态规划

```cpp
class Solution {
public:
    string longestPalindrome(string s) {
        int n = s.size();
        if (n < 2)  return s;
        int maxL = 1, begin = 0, l, r, i;
        vector<vector<bool>> dp(n, vector<bool>(n, true));
        for (r = 1; r < n; r++) {
            for (l = 0; l < r; l++) {
                if (s[l] != s[r])
                    dp[l][r] = false;
                else if (r - l < 3)
                    dp[l][r] = true;
                else
                    dp[l][r] = dp[l + 1][r - 1];
                if (dp[l][r] && (r - l + 1 > maxL)) {
                    maxL = r - l + 1;
                    begin = l;
                }
            }
        }
        return s.substr(begin, maxL);
    }
};
```

复杂度 O(N²)，空间 O(N²)

## 性能对比

- 中心扩展法：通过 56 ms，6.8 MB Cpp
- 动态规划：通过 1200 ms，16.8 MB Cpp

[src: raw/ingested/2技术/算法/cpp_leetcode_腾讯精选练习50题-[最长回文子串](https---leetcode-cn.com-problems-longest-palindromi.md]