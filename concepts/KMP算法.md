# KMP 算法

KMP（Knuth-Morris-Pratt）算法是一种高效的字符串匹配算法，通过利用前缀后缀相同的信息，在匹配失败时移动模式串，避免从头开始匹配，从而将时间复杂度优化到 O(n+m)。

## 核心思想

- **前缀后缀相同**：计算模式串中每个位置的最长相等前后缀长度（即 next 数组），当匹配失败时，利用 next 数组跳过已匹配的前缀部分，继续匹配。
- **类似 DP**：next 数组相当于状态数组，失配时取出的下标就是状态转换。

## 实现 strStr()

实现 `strStr()` 函数，在 haystack 字符串中找出 needle 字符串出现的第一个位置（下标从 0 开始），如果不存在则返回 -1。当 needle 是空字符串时返回 0。

**力扣**：[implement-strstr](https://leetcode-cn.com/problems/implement-strstr/)

### 示例

- 输入：haystack = "hello", needle = "ll" → 输出：2
- 输入：haystack = "aaaaa", needle = "bba" → 输出：-1
- 输入：haystack = "", needle = "" → 输出：0

### 代码实现

```cpp
class Solution {
public:
    int strStr(string haystack, string needle) {
        if (needle.empty()) return 0;
        int n = haystack.size(), m = needle.size();
        if (n < m) return -1;
        vector<int> next(m);
        getNext(needle, next);
        for (int i = 0, j = 0; i < n;) {
            if (j == -1) { i++; j = 0; }
            else if (haystack[i] == needle[j]) {
                i++; j++;
                if (j == m) return i - m;
            } else j = next[j];
        }
        return -1;
    }
    void getNext(const string& s, vector<int>& next) {
        next[0] = -1;
        int j = -1;
        for (int i = 1; i < (int)s.size(); i++) {
            while (j >= 0 && s[j + 1] != s[i]) j = next[j];
            if (s[j + 1] == s[i]) j++;
            next[i] = j;
        }
    }
};
```

### 算法说明

- `next[i]` 表示模式串中前 i+1 个字符（即 s[0..i]）的最长相等前后缀长度减 1（以 -1 为起始）。
- 匹配时，若 `haystack[i] != needle[j]`，则 `j = next[j]`，相当于模式串向右移动 `k - next[k]` 位。
- 时间复杂度 O(n+m)，空间复杂度 O(m)。

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-kmp.md]