# DFS 回溯与递归

> 排列、组合、子集、括号生成、单词搜索、正则匹配等 —— 回溯法的核心是"做选择→递归→撤销选择"。

---

## 一、排列

### 1.1 全排列（无重复）

> LeetCode 46: https://leetcode.cn/problems/permutations

**思路**：**交换法**——将每个元素交换到当前位置 start，递归处理后续位置。不需要额外 used 数组和 path，直接在原数组上操作，空间更省。**时间复杂度 O(n!)**，n 最大通常 ≤ 10。

```cpp
vector<vector<int>> permute(vector<int>& nums) {
    vector<vector<int>> res;
    function<void(int)> dfs = [&](int start) {
        if (start == nums.size()) {
            res.push_back(nums);          // 当前排列已确定，收集结果
            return;
        }
        for (int i = start; i < nums.size(); i++) {
            swap(nums[start], nums[i]);   // 将 nums[i] 固定到当前位置
            dfs(start + 1);               // 递归处理下一个位置
            swap(nums[start], nums[i]);   // 撤销交换
        }
    };
    dfs(0);
    return res;
}
```
- **复杂度**：时间 O(n!)，空间 O(n)（递归深度），无额外 path/used 开销
- **变体**：排列包含重复元素（见 1.2）→ 同层用 set 去重

### 1.2 全排列 II（有重复）

> LeetCode 47: https://leetcode.cn/problems/permutations-ii

**思路**：交换法 + `used[]` 判重。先排序让相同元素相邻，同层递归时如果当前元素和前一个相同且前一个没被用过，说明是重复分支，跳过。相比每层建 set，排序 + used 的方式空间更省。

```cpp
vector<vector<int>> permuteUnique(vector<int>& nums) {
    vector<vector<int>> res;
    sort(nums.begin(), nums.end());
    vector<int> path;
    vector<bool> used(nums.size(), false);
    function<void()> dfs = [&]() {
        if (path.size() == nums.size()) {
            res.push_back(path);
            return;
        }
        for (int i = 0; i < nums.size(); i++) {
            if (used[i]) continue;
            if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;
            used[i] = true;
            path.push_back(nums[i]);
            dfs();
            path.pop_back();
            used[i] = false;
        }
    };
    dfs();
    return res;
}
```
- **复杂度**：时间 O(n!)，空间 O(n)（递归深度 + path + used）
- **对比**：无重复用交换法（原地操作，无需额外标记），有重复用排序 + used 去重

---

## 二、组合与子集

### 2.1 子集

> LeetCode 78: https://leetcode.cn/problems/subsets

**思路**：回溯模板——每个节点都是子集（提前收集结果，不等叶子节点）。用 `start` 控制遍历起点，保证有序组合，避免重复（如 [1,2] 和 [2,1] 视为同一子集）。每个元素选或不选，共 2^n 种。

```cpp
vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> res;
    vector<int> path;
    function<void(int)> dfs = [&](int start) {
        res.push_back(path);                     // 每个节点都收集（包括空集）
        for (int i = start; i < nums.size(); i++) {
            path.push_back(nums[i]);
            dfs(i + 1);                          // 下一层从 i+1 开始，避免重复
            path.pop_back();
        }
    };
    dfs(0);
    return res;
}
```
- **复杂度**：时间 O(2ⁿ)，空间 O(n)（递归深度 + path）
- **边界**：空数组返回 `[[]]`
- **变体**：子集 II（LeetCode 90，有重复）→ 排序 + 剪枝 `i > start && nums[i] == nums[i-1]`

### 2.2 组合总和

> LeetCode 39: https://leetcode.cn/problems/combination-sum

**思路**：回溯——每个元素可无限重复选，所以递归时传 `i` 而非 `i+1`。配合排序后可 `break` 剪枝（`candidates[i] > remain` 时后续更大元素无需遍历）。remain 减到 0 收集结果。

```cpp
vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
    vector<vector<int>> res;
    vector<int> path;
    sort(candidates.begin(), candidates.end());         // 排序后方便剪枝
    function<void(int, int)> dfs = [&](int start, int remain) {
        if (remain == 0) { res.push_back(path); return; }
        for (int i = start; i < candidates.size(); i++) {
            if (candidates[i] > remain) break;          // 剪枝（配合排序）
            path.push_back(candidates[i]);
            dfs(i, remain - candidates[i]);             // 可重复用同一元素
            path.pop_back();
        }
    };
    dfs(0, target);
    return res;
}
```
- **复杂度**：时间 O(2ⁿ)（最坏），空间 O(target)（递归深度）
- **边界**：candidates 为空返回空；target 为 0 返回 `[[]]`
- **变体**：组合总和 II（LeetCode 40）→ 每个元素只能用一次，传 `i+1` + 去重剪枝

### 2.3 目标和

> LeetCode 494: https://leetcode.cn/problems/target-sum

**思路**：每个数选正或负，形成二叉树递归。n ≤ 20 时 2ⁿ 可接受。**优化**：n 较大时转化为 DP——设正数和为 P，则 `P - (sum - P) = target` → `P = (target + sum) / 2`，转化为子集和问题（找和为 P 的子集个数）。

```cpp
int findTargetSumWays(vector<int>& nums, int S) {
    int count = 0;
    function<void(int, int)> dfs = [&](int i, int sum) {
        if (i == nums.size()) {
            if (sum == S) count++;
            return;
        }
        dfs(i + 1, sum + nums[i]);
        dfs(i + 1, sum - nums[i]);
    };
    dfs(0, 0);
    return count;
}
```
- **复杂度**：时间 O(2ⁿ)，空间 O(n)
- **变体**：DP 解法 O(n × sum) —— 适合 n 大但 sum 小的场景

### 2.4 划分为 K 个相等子集

> LeetCode 698: https://leetcode.cn/problems/partition-to-k-equal-sum-subsets

**思路**：先判断总和能否整除 k，单元素 > target 直接 false。**关键剪枝**：从大到小排序（大元素先放，减少搜索分支）；桶值相同时跳过（`i > 0 && bucket[i] == bucket[i-1]`，因为放哪个桶都一样）。

```cpp
bool canPartitionKSubsets(vector<int>& nums, int k) {
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum % k) return false;
    int target = sum / k;
    sort(nums.rbegin(), nums.rend());          // 降序排列，大数先放
    if (nums[0] > target) return false;
    vector<int> bucket(k, 0);
    function<bool(int)> dfs = [&](int idx) {
        if (idx == nums.size()) return true;
        for (int i = 0; i < k; i++) {
            if (bucket[i] + nums[idx] > target) continue;
            if (i > 0 && bucket[i] == bucket[i-1]) continue;  // 相同值的桶剪枝
            bucket[i] += nums[idx];
            if (dfs(idx + 1)) return true;
            bucket[i] -= nums[idx];
        }
        return false;
    };
    return dfs(0);
}
```
- **复杂度**：时间 O(kⁿ)，但剪枝后实际快很多；空间 O(k + n)
- **变体**：划分为两个相等子集（LeetCode 416）→ 子集和问题，DP 即可

---

## 三、括号生成

> LeetCode 22: https://leetcode.cn/problems/generate-parentheses

**思路**：回溯——保证任意前缀中左括号 ≥ 右括号。左括号数 < n 时可加 `(`，右括号数 < 左括号数时可加 `)`。结果数量为卡特兰数 C(2n,n)/(n+1)。

```cpp
vector<string> generateParenthesis(int n) {
    vector<string> res;
    string path;
    function<void(int, int)> dfs = [&](int open, int close) {
        if (path.size() == n * 2) { res.push_back(path); return; }
        if (open < n) { path.push_back('('); dfs(open + 1, close); path.pop_back(); }
        if (close < open) { path.push_back(')'); dfs(open, close + 1); path.pop_back(); }
    };
    dfs(0, 0);
    return res;
}
```
- **复杂度**：时间 O(4ⁿ/√n)（卡特兰数），空间 O(n)
- **边界**：n=0 返回 `[""]`（空字符串）
- **变体**：括号的分数（LeetCode 856）→ 栈或递归计分

---

## 四、单词搜索（多向 DFS）

> LeetCode 79: https://leetcode.cn/problems/word-search

**思路**：枚举每个格子作为起点，四方向 DFS 匹配单词。用 visited 标记已用字符（防止同一位置重复使用）。匹配到 word 最后一个字符时返回 true。**剪枝**：如果剩余字符数 > 剩余格子数可提前返回。

```cpp
bool exist(vector<vector<char>>& board, string word) {
    int m = board.size(), n = board[0].size();
    vector<vector<bool>> visited(m, vector<bool>(n, false));
    int dirs[4][2] = {{1,0},{-1,0},{0,1},{0,-1}};

    function<bool(int, int, int)> dfs = [&](int x, int y, int idx) {
        if (idx == word.size()) return true;
        if (x < 0 || x >= m || y < 0 || y >= n) return false;
        if (visited[x][y] || board[x][y] != word[idx]) return false;
        visited[x][y] = true;
        for (auto [dx, dy] : dirs)
            if (dfs(x + dx, y + dy, idx + 1)) return true;
        visited[x][y] = false;          // 回溯撤销
        return false;
    };

    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (dfs(i, j, 0)) return true;
    return false;
}
```
- **复杂度**：时间 O(m × n × 3^L)，L 为单词长度（除起点外只有 3 个方向可选）；空间 O(L)（递归栈）
- **边界**：board 为空返回 false；word 为空返回 true
- **变体**：单词搜索 II（LeetCode 212）→ 多单词用 Trie 剪枝

---

## 五、正则表达式匹配

> LeetCode 10: https://leetcode.cn/problems/regular-expression-matching

**思路**：动态规划。`dp[i][j]` 表示 s 前 i 个字符和 p 前 j 个字符是否匹配。核心处理 `*`——匹配 0 次（跳过 `c*`）或匹配 1+ 次（要求当前字符匹配，然后看 s 前 i-1 个）。`.*` 可以匹配任意序列。

```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;

    // 空串匹配 a*b* 等形式（如 "" 匹配 "a*b*c*"）
    for (int j = 2; j <= n; j++)
        if (p[j-1] == '*') dp[0][j] = dp[0][j-2];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (p[j-1] == '*') {
                dp[i][j] = dp[i][j-2];  // 匹配 0 次，跳过 c*
                if (p[j-2] == '.' || p[j-2] == s[i-1])
                    dp[i][j] = dp[i][j] || dp[i-1][j];  // 匹配 1+ 次
            } else if (p[j-1] == '.' || p[j-1] == s[i-1]) {
                dp[i][j] = dp[i-1][j-1];  // 单字符匹配
            }
        }
    }
    return dp[m][n];
}
```
- **复杂度**：时间 O(mn)，空间 O(mn)（可优化到 O(n)）
- **边界**：空模式 p="" 只能匹配空串；`.*` 匹配任意字符串
- **变体**：通配符匹配（LeetCode 44）→ `*` 可匹配任意序列，`?` 匹配单字符，更简单

---

## 六、分治递归

### 6.1 为运算表达式设计优先级

> LeetCode 241: https://leetcode.cn/problems/different-ways-to-add-parentheses

**思路**：分治——遍历表达式，遇到运算符时，将表达式分成左右两半，递归计算所有可能结果，再组合。相当于在所有运算符位置"切一刀"，加括号优先计算两边。

```cpp
vector<int> diffWaysToCompute(string expression) {
    vector<int> res;
    for (int i = 0; i < expression.size(); i++) {
        char c = expression[i];
        if (c == '+' || c == '-' || c == '*') {
            auto left = diffWaysToCompute(expression.substr(0, i));
            auto right = diffWaysToCompute(expression.substr(i + 1));
            for (int a : left)
                for (int b : right) {
                    if (c == '+') res.push_back(a + b);
                    else if (c == '-') res.push_back(a - b);
                    else res.push_back(a * b);
                }
        }
    }
    if (res.empty()) res.push_back(stoi(expression));  // 纯数字
    return res;
}
```
- **复杂度**：时间 O(卡特兰数) —— 结果数量是卡特兰数；空间 O(n)
- **边界**：纯数字表达式返回 `[数字]`
- **变体**：加括号改变表达式结果（类似题目）→ 可用记忆化搜索优化重复子问题
