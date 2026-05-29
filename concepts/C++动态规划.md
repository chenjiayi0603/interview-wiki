# C++ 动态规划

> 本文涵盖动态规划（Dynamic Programming）的经典题型与 C++ 实现，包括线性 DP、背包与网格、其他经典问题。

See also: [[C++手写代码模板]], [[C++高频面试问题]]

## 7.1 线性 DP

**说明**：爬楼梯、LIS（二分优化）、LCS 等线性递推。

### 爬楼梯

n 阶楼梯，每次可爬 1 或 2 阶，求到顶的不同方法数。

**力扣**：[climbing-stairs](https://leetcode-cn.com/problems/climbing-stairs)

```cpp
// 爬楼梯：dp[i]=dp[i-1]+dp[i-2]，空间可压成 O(1)
int climbStairs(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; i++) { int t = a + b; a = b; b = t; }
    return b;
}
```

### 最长连续递增序列

求数组中最长**连续**递增子序列长度。

**力扣**：[longest-continuous-increasing-subsequence](https://leetcode-cn.com/problems/longest-continuous-increasing-subsequence)

```cpp
int findLengthOfLCIS(vector<int>& nums) {
    if (nums.size() <= 1) return nums.size();
    int g = 1, l = 1;
    for (int i = 1; i < nums.size(); i++) {
        if (nums[i] > nums[i-1]) l++;
        else { g = max(g, l); l = 1; }
    }
    return max(g, l);
}
```

### 最长上升子序列

求数组中最长严格递增子序列的长度。

**力扣**：[longest-increasing-subsequence](https://leetcode-cn.com/problems/longest-increasing-subsequence)

```cpp
// 最长上升子序列 O(nlogn)：tail[i] 为长度 i+1 的 LIS 末尾最小值，二分维护
int lengthOfLIS(vector<int>& nums) {
    vector<int> tail;
    for (int x : nums) {
        auto it = lower_bound(tail.begin(), tail.end(), x);
        if (it == tail.end()) tail.push_back(x);
        else *it = x;
    }
    return tail.size();
}
```

// 动规写法（O(n^2)），dp[i] 表示以 nums[i] 结尾的 LIS 长度
```cpp
int lengthOfLIS_dp(vector<int>& nums) {
    int n = nums.size();
    if (n == 0) return 0;
    vector<int> dp(n, 1);  // 每个位置至少为 1（自身）

    int ans = 1;
    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < i; ++j) {
            if (nums[j] < nums[i]) {
                dp[i] = max(dp[i], dp[j] + 1);
            }
        }
        ans = max(ans, dp[i]);
    }
    return ans;
}
```

### 最长公共子序列

求两个字符串的最长公共子序列长度（可不连续）。

**力扣**：[longest-common-subsequence](https://leetcode-cn.com/problems/longest-common-subsequence)

```cpp
// 最长公共子序列：dp[i][j]=a[i]==b[j]?dp[i-1][j-1]+1:max(dp[i-1][j],dp[i][j-1])
int longestCommonSubsequence(string a, string b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = a[i-1] == b[j-1] ? dp[i-1][j-1] + 1 : max(dp[i-1][j], dp[i][j-1]);
    return dp[m][n];
}
```

## 7.2 背包与网格

**说明**：0-1 背包逆序、完全背包正序；网格路径数递推。

### 0-1 背包

n 件物品有重量 w 和价值 v，背包容量 W，每件最多选一次，求最大价值。

```cpp
// 0-1背包：每件最多选一次，逆序避免重复使用
int knapsack01(vector<int>& w, vector<int>& v, int W) {
    vector<int> dp(W + 1, 0);
    // 遍历每个物品
    for (int i = 0; i < w.size(); i++)
        // 从大到小枚举容量，保证每个物品只选一次，避免重复
        for (int j = W; j >= w[i]; j--)
            dp[j] = max(dp[j], dp[j - w[i]] + v[i]);
    return dp[W];
}
```

### 完全背包

n 件物品，每件可无限选，求容量 W 内的最大价值。

```cpp
// 完全背包：每件可无限选，正序允许重复使用
int knapsackFull(vector<int>& w, vector<int>& v, int W) {
    // 完全背包：每件可选多次，正序遍历容量
    vector<int> dp(W + 1, 0);
    for (int i = 0; i < w.size(); i++)
        // 正序遍历容量，允许同一物品被多次使用
        // 为啥前到后就是多次，后到前就是单次？
        // 正序（前到后）：j=w[i]~W，本物品每次都可以用到上一轮刚更新的 dp[j-w[i]]，即同一物品可多次取用
        // 逆序（后到前）：j=W~w[i]，本物品每次只能用到上一轮还未更新的 dp[j-w[i]]，即每件物品只取一次
        for (int j = w[i]; j <= W; j++)
            dp[j] = max(dp[j], dp[j - w[i]] + v[i]);
    return dp[W];
}
```

### 不同路径

m×n 网格从左上到右下，每次只能右或下，求路径数。

**力扣**：[unique-paths](https://leetcode-cn.com/problems/unique-paths)

```cpp
// 不同路径：dp[i][j]=dp[i-1][j]+dp[i][j-1]，第一行/列均为 1
int uniquePaths(int m, int n) {
    // dp[i][j] 记录到 (i,j) 的路径数，起点和边界初始化为 1
    vector<vector<int>> dp(m, vector<int>(n, 1));
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = dp[i-1][j] + dp[i][j-1];
    return dp[m-1][n-1];
}
```

### 完全平方数

正整数 n 表示为若干完全平方数之和，求最少个数。

**力扣**：[perfect-squares](https://leetcode-cn.com/problems/perfect-squares)

```cpp
int numSquares(int n) {
    vector<int> dp(n + 1);
    for (int i = 1; i <= n; i++) {
        dp[i] = i;
        for (int k = 1; k * k <= i; k++)
            dp[i] = min(dp[i], dp[i - k * k] + 1);
    }
    return dp[n];
}
```

### 单词拆分

判断字符串 s 能否被 wordDict 中的单词拆分（可重复使用）。

**力扣**：[word-break](https://leetcode-cn.com/problems/word-break)

```cpp
bool wordBreak(string s, vector<string>& wordDict) {
    unordered_set<string> st(wordDict.begin(), wordDict.end());
    vector<bool> dp(s.size() + 1, false);
    dp[0] = true;
    for (int i = 1; i <= s.size(); i++)
        for (int j = 0; j < i; j++)
            if (dp[j] && st.count(s.substr(j, i - j))) { dp[i] = true; break; }
    return dp[s.size()];
}
```

### 不同的子序列

s 的子序列中 t 出现的个数。

**力扣**：[distinct-subsequences](https://leetcode-cn.com/problems/distinct-subsequences)

```cpp
int numDistinct(string s, string t) {
    vector<vector<long>> dp(s.size() + 1, vector<long>(t.size() + 1, 0));
    for (int i = 0; i <= s.size(); i++) dp[i][0] = 1;
    for (int i = 1; i <= s.size(); i++)
        for (int j = 1; j <= t.size() && j <= i; j++)
            dp[i][j] = s[i-1] == t[j-1] ? dp[i-1][j] + dp[i-1][j-1] : dp[i-1][j];
    return dp[s.size()][t.size()];
}
```

### 分割回文串 II

将 s 分割成若干回文子串，求最少分割次数。

**力扣**：[palindrome-partitioning-ii](https://leetcode.cn/problems/palindrome-partitioning-ii)

```cpp
int minCut(string s) {
    if (s.size() <= 1) return 0;
    int n = s.size();
    vector<vector<int>> dp(n, vector<int>(n));
    vector<int> cuts(n + 1);
    cuts[0] = -1;
    for (int i = 0; i < n; i++) {
        cuts[i + 1] = i;
        for (int j = 0; j <= i; j++)
            if (s[i] == s[j] && (i - j < 2 || dp[i-1][j+1])) {
                dp[i][j] = 1;
                cuts[i + 1] = min(cuts[i + 1], cuts[j] + 1);
            }
    }
    return cuts[n];
}
```

### 不同的二叉搜索树

n 个节点（1~n）能组成多少种不同的 BST。

**力扣**：[unique-binary-search-trees](https://leetcode-cn.com/problems/unique-binary-search-trees)

```cpp
int numTrees(int n) {
    if (n <= 1) return 1;
    vector<int> dp(n + 1);
    dp[0] = 1;
    for (int i = 1; i <= n; i++)
        for (int j = 0; j < i; j++)
            dp[i] += dp[j] * dp[i - j - 1];
    return dp[n];
}
```

### 鸡蛋掉落

k 个鸡蛋、n 层楼，确定临界楼层 f 的最小操作次数。

**力扣**：[super-egg-drop](https://leetcode-cn.com/problems/super-egg-drop)

```cpp
int superEggDrop(int k, int n) {
    if (n == 1) return 1;
    vector<vector<int>> dp(k + 1, vector<int>(n + 1, 0));
    int c = 0;
    while (dp[k][c] < n) {
        c++;
        for (int i = 1; i <= k; i++)
            dp[i][c] = dp[i-1][c-1] + dp[i][c-1] + 1;
    }
    return c;
}
```

### 买卖股票 III

最多完成两笔交易，求最大利润。

**力扣**：[best-time-to-buy-and-sell-stock-iii](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-iii)

```cpp
int maxProfit(vector<int>& prices) {
    if (prices.empty()) return 0;
    int p1 = INT_MAX, p2 = INT_MAX, dp1 = 0, dp2 = 0;
    for (int x : prices) {
        dp1 = max(dp1, x - p1);
        dp2 = max(dp2, x - p2);
        p1 = min(p1, x);
        p2 = min(p2, x - dp1);  // 第二次买入价减去第一次利润
    }
    return dp2;
}
```

## 7.3 其他经典

**说明**：最大子序和（贪心）、最长回文子串（中心扩展）、丑数（三指针）。

### 最大子序和

求数组中连续子数组的最大和。

**力扣**：[maximum-subarray](https://leetcode-cn.com/problems/maximum-subarray)

```cpp
// 最大子序和：累加 sum，sum<0 则重置，全程维护 max
int maxSubArray(vector<int>& nums) {
    int res = nums[0], sum = 0;
    for (int x : nums) {
        sum += x;
        res = max(res, sum);
        if (sum < 0) sum = 0;
    }
    return res;
}
```

### 最长回文子串

求字符串中最长回文子串。

**力扣**：[longest-palindromic-substring](https://leetcode-cn.com/problems/longest-palindromic-substring)

```cpp
// 最长回文子串：以每个位置为中心（奇/偶长度）向两侧扩展
string longestPalindrome(string s) {
    int n = s.size(), start = 0, maxLen = 1;
    auto expand = [&](int l, int r) {
        while (l >= 0 && r < n && s[l] == s[r]) l--, r++;
        if (r - l - 1 > maxLen) start = l + 1, maxLen = r - l - 1;
    };
    // 依次以每个字符为中心（奇/偶）扩展
    for (int i = 0; i < n; i++) expand(i, i), expand(i, i + 1);
    return s.substr(start, maxLen);
}
```

### 丑数 II

丑数指只含质因子 2、3、5 的正整数，求第 n 个丑数。

**力扣**：[ugly-number-ii](https://leetcode-cn.com/problems/ugly-number-ii)

```cpp
// 丑数 II：dp[i]=min(2*dp[p2], 3*dp[p3], 5*dp[p5])，三指针推进
int nthUglyNumber(int n) {
    vector<int> dp(n);
    dp[0] = 1;
    // 三指针i2/i3/i5分别跟踪下一个要乘2/3/5的索引，每次取最小更新并推进对应指针
    int i2 = 0, i3 = 0, i5 = 0;
    for (int i = 1; i < n; ++i) {
        int next = min({dp[i2] * 2, dp[i3] * 3, dp[i5] * 5});
        if (next == dp[i2] * 2) ++i2;
        if (next == dp[i3] * 3) ++i3;
        if (next == dp[i5] * 5) ++i5;
        dp[i] = next;
    }
    return dp[n-1];
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-七、动态规划.md]
[src: raw/ingested/2技术/算法/cpp_leetcode技巧-使用双遍历的二维数组递推方程.md]