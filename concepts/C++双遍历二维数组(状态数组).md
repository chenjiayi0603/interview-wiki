# C++ 双遍历二维数组（状态数组）

> 本文涵盖双遍历的二维数组（状态数组）在动态规划中的应用，包括括号生成、不同的子序列、最长回文子串、分割回文串 II、不同的二叉搜索树等经典题目。

See also: [[C++动态规划]], [[C++DFS回溯]]

## 括号生成

数字 n 代表生成括号的对数，生成所有可能的并且有效的括号组合。

**力扣**：[generate-parentheses](https://leetcode-cn.com/problems/generate-parentheses)

```cpp
class Solution {
public:
    vector<string> generateParenthesis(int n) {
        if (n <= 0) return {};
        vector<vector<string>> dp(n + 1);
        dp[0].push_back("");
        for (size_t i = 1; i <= n; i++) {
            for (size_t j = 0; j < i; j++) {
                for (string& str1 : dp[j]) {
                    for (string& str2 : dp[i - j - 1]) {
                        dp[i].push_back("(" + str1 + ")" + str2);
                    }
                }
            }
        }
        return dp[n];
    }
};
```

## 不同的子序列

给定一个字符串 s 和一个字符串 t，计算在 s 的子序列中 t 出现的个数。

**力扣**：[distinct-subsequences](https://leetcode-cn.com/problems/distinct-subsequences)

```cpp
class Solution {
public:
    int numDistinct(string s, string t) {
        vector<vector<long>> dp(s.size() + 1, vector<long>(t.size() + 1, 0));
        for (int i = 0; i <= s.size(); i++) dp[i][0] = 1;
        for (int i = 1; i <= s.size(); i++) {
            for (int j = 1; j <= t.size() && j <= i; j++) {
                if (s[i - 1] == t[j - 1]) {
                    dp[i][j] = dp[i - 1][j] + dp[i - 1][j - 1];
                } else {
                    dp[i][j] = dp[i - 1][j];
                }
            }
        }
        return dp[s.size()][t.size()];
    }
};
```

## 最长回文子串

给你一个字符串 s，找到 s 中最长的回文子串。

**力扣**：[longest-palindromic-substring](https://leetcode-cn.com/problems/longest-palindromic-substring/)

```cpp
class Solution {
public:
    string longestPalindrome(string s) {
        if (s.size() <= 1) return s;
        vector<vector<int>> dp(s.size(), vector<int>(s.size(), 0));
        int maxLen = 1, start = 0;
        for (int i = 0; i < s.size(); i++) {
            for (int j = 0; j <= i; ++j) {
                if (s[i] == s[j] && (i - j < 2 || dp[i-1][j+1])) {
                    dp[i][j] = 1;
                    if (i - j + 1 > maxLen) {
                        start = j;
                        maxLen = i - j + 1;
                    }
                }
            }
        }
        return s.substr(start, maxLen);
    }
};
```

## 分割回文串 II

给你一个字符串 s，请你将 s 分割成一些子串，使每个子串都是回文。返回符合要求的最少分割次数。

**力扣**：[palindrome-partitioning-ii](https://leetcode.cn/problems/palindrome-partitioning-ii/)

```cpp
class Solution {
public:
    int minCut(string s) {
        if (s.size() <= 1) return 0;
        int len = s.size();
        vector<vector<int>> dp(len, vector<int>(len));
        vector<int> cuts(len + 1);
        cuts[0] = -1;
        for (int i = 0; i < len; i++) {
            cuts[i+1] = i;
            for (int j = 0; j <= i; ++j) {
                if (s[i] == s[j] && (i - j < 2 || dp[i-1][j+1])) {
                    dp[i][j] = 1;
                    cuts[i+1] = min(cuts[i+1], cuts[j] + 1);
                }
            }
        }
        return cuts[len];
    }
};
```

## 不同的二叉搜索树

给你一个整数 n，求恰由 n 个节点组成且节点值从 1 到 n 互不相同的二叉搜索树有多少种？

**力扣**：[unique-binary-search-trees](https://leetcode-cn.com/problems/unique-binary-search-trees)

```cpp
class Solution {
public:
    int numTrees(int n) {
        if (n <= 1) return 1;
        vector<int> dp(n+1);
        dp[0] = 1;
        for (int i = 1; i <= n; i++) {
            for (int j = 0; j < i; j++) {
                dp[i] += dp[j] * dp[i-j-1];
            }
        }
        return dp[n];
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-双遍历的二维数组(状态数组).md]