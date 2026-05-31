# C++ DFS / 回溯

> 本文涵盖 DFS 回溯算法的经典题目：条件递归（正则匹配、括号生成）、排列组合（全排列、子集、组合总和、目标和、划分为 k 个相等子集）、多向 DFS（单词搜索）、分治递归（前序+中序/中序+后序构造二叉树、为运算表达式设计优先级）。

See also: [[C++双指针]], [[C++二叉树LCA与路径]], [[Go_LeetCode技巧-条件递归]]

## 6.1 条件递归

**说明**：正则匹配（`*`、`.`）；括号生成保证合法。

### 正则表达式匹配

判断字符串 s 是否完全匹配模式 p，`*` 匹配 0 或多个前一字符，`.` 匹配任意单字符。

**力扣**：[regular-expression-matching](https://leetcode-cn.com/problems/regular-expression-matching)

```cpp
// 正则表达式匹配：* 匹配 0 或多个前一字符，. 匹配任意单字符
bool isMatch(string s, string p) {
    return dfs(s, p, 0, 0);
}
bool dfs(string& s, string& p, int i, int j) {
    if (j >= p.size()) return i == s.size();
    bool jmatch = i < s.size() && (s[i] == p[j] || p[j] == '.');
    if (j + 1 < p.size() && p[j + 1] == '*') {
        if (dfs(s, p, i, j + 2)) return true;   // * 匹配 0 个，跳过 x*
        return jmatch && dfs(s, p, i + 1, j);   // * 匹配 1 个，j 不动
    }
    return jmatch && dfs(s, p, i + 1, j + 1);   // 普通匹配
}
```

### 括号生成

生成 n 对括号的所有合法组合（如 n=2 得 ()()、(())）。

**力扣**：[generate-parentheses](https://leetcode-cn.com/problems/generate-parentheses)

```cpp
// l/r 为剩余左右括号数，保证 l<=r 即合法
vector<string> generateParenthesis(int n) {
    vector<string> res;
    // 使用匿名函数（lambda）定义 dfs，可以直接捕获外部变量（如 res），代码结构紧凑，局部封装递归逻辑，不污染全局命名空间
    function<void(string, int, int)> dfs = [&](string s, int l, int r) {
        if (l == 0 && r == 0) { res.push_back(s); return; }
        if (l > 0) dfs(s + '(', l - 1, r);
        if (l < r) dfs(s + ')', l, r - 1);  // 右不多于左
    };
    dfs("", n, n);
    return res;
}
```

## 6.2 排列组合

**说明**：全排列（交换法/used 去重）、子集、组合总和（可重复选）。

### 全排列（无重复）

给定无重复整数数组，返回所有可能的排列。

**力扣**：[permutations](https://leetcode-cn.com/problems/permutations)

```cpp
// 全排列（无重复）：交换法，pos 为当前填充位置
vector<vector<int>> permute(vector<int>& nums) {
    vector<vector<int>> res;
    function<void(int)> dfs = [&](int pos) {
        if (pos == nums.size()) { res.push_back(nums); return; }
        for (int i = pos; i < nums.size(); i++) {
            swap(nums[pos], nums[i]);
            dfs(pos + 1);
            swap(nums[pos], nums[i]);  // 回溯
        }
    };
    dfs(0);
    return res;
}
```

### 全排列 II（有重复）

给定可能有重复的整数数组，返回所有不重复的排列。

**力扣**：[permutations-ii](https://leetcode-cn.com/problems/permutations-ii)

```cpp
// 全排列 II（有重复）：used 去重，相同元素上一个未选则跳过
vector<vector<int>> permuteUnique(vector<int>& nums) {
    sort(nums.begin(), nums.end());
    vector<vector<int>> res;
    vector<bool> used(nums.size(), false);
    vector<int> path;
    function<void()> dfs = [&]() {
        if (path.size() == nums.size()) { res.push_back(path); return; }
        for (int i = 0; i < nums.size(); i++) {
            if (used[i]) continue;
            if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;  // 去重
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

### 子集

给定无重复整数数组，返回所有可能的子集（幂集）。

**力扣**：[subsets](https://leetcode-cn.com/problems/subsets)

```cpp
// 子集：每个位置选或不选，从 i 开始枚举
vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> res;
    vector<int> path;
    function<void(int)> dfs = [&](int i) {
        res.push_back(path);
        for (int j = i; j < nums.size(); j++) {
            path.push_back(nums[j]);
            dfs(j + 1);
            path.pop_back();
        }
    };
    dfs(0);
    return res;
}
```

### 组合总和

给定数组和 target，找所有和等于 target 的组合，同一数可重复使用。

**力扣**：[combination-sum](https://leetcode-cn.com/problems/combination-sum)

```cpp
// 组合总和：可重复选，从 pos 起
vector<vector<int>> combinationSum(vector<int>& nums, int target) {
    vector<vector<int>> res;
    vector<int> path;
    function<void(int, int)> dfs = [&](int target, int pos) {
        if (target < 0) return;
        if (target == 0) { res.push_back(path); return; }
        for (int i = pos; i < nums.size(); i++) {
            path.push_back(nums[i]);
            dfs(target - nums[i], i);  // 可重复选，仍从 i 开始
            path.pop_back();
        }
    };
    dfs(target, 0);
    return res;
}
```

### 目标和

每个数前加 + 或 -，使表达式结果等于 target，求方案数。

**力扣**：[target-sum](https://leetcode-cn.com/problems/target-sum)

```cpp
int findTargetSumWays(vector<int>& nums, int S) {
    int res = 0, sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum == S) res++;
    function<void(int, int)> dfs = [&](int sum, int pos) {
        for (int i = pos; i < nums.size(); i++) {
            int tmp = sum - 2 * nums[i];
            if (tmp == S) res++;
            if (tmp >= S) dfs(tmp, i + 1);
        }
    };
    dfs(sum, 0);
    return res;
}
```

### 划分为 k 个相等子集

数组能否分成 k 个非空子集，每子集和相等。

**力扣**：[partition-to-k-equal-sum-subsets](https://leetcode-cn.com/problems/partition-to-k-equal-sum-subsets)

```cpp
bool canPartitionKSubsets(vector<int>& nums, int k) {
    if (k == 1) return true;
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum % k) return false;
    int target = sum / k;
    sort(nums.begin(), nums.end(), greater<int>());  // 从大到小优化
    vector<int> sums(k);
    function<bool(int)> dfs = [&](int pos) {
        if (pos == nums.size()) return true;
        for (int i = 0; i < k; i++) {
            if (nums[pos] + sums[i] <= target) {
                sums[i] += nums[pos];
                if (dfs(pos + 1)) return true;
                sums[i] -= nums[pos];
            }
            if (sums[i] == 0) break;  // 空组后面也加不进去
        }
        return false;
    };
    return dfs(0);
}
```

## 6.3 多向 DFS（回溯）

**说明**：矩阵中四向搜索单词，用 `'\0'` 标记已访问防重复。

### 单词搜索

二维字符网格中，判断是否存在相邻格子（上下左右）连成给定单词的路径。

**力扣**：[word-search](https://leetcode-cn.com/problems/word-search)

```cpp
// 单词搜索：四向 DFS，用 '\0' 标记已访问
bool exist(vector<vector<char>>& board, string word) {
    int h = board.size(), w = board[0].size();
    function<bool(int, int, int)> dfs = [&](int i, int j, int pos) {
        if (i < 0 || j < 0 || i >= h || j >= w || board[i][j] == '\0' || board[i][j] != word[pos])
            return false;
        if (pos == word.size() - 1) return true;
        char t = board[i][j];
        board[i][j] = '\0';  // 标记已访问
        bool ok = dfs(i+1, j, pos+1) || dfs(i-1, j, pos+1) || dfs(i, j+1, pos+1) || dfs(i, j-1, pos+1);
        board[i][j] = t;     // 回溯
        return ok;
    };
    for (int i = 0; i < h; i++)
        for (int j = 0; j < w; j++)
            if (dfs(i, j, 0)) return true;
    return false;
}
```

## 6.4 分治递归

**说明**：前序首为根，中序定位左右子树，递归构造。

### 前序+中序构造二叉树

给定前序和中序遍历序列（无重复值），构造二叉树。

**力扣**：[construct-binary-tree-from-preorder-and-inorder-traversal](https://leetcode-cn.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal)

```cpp
// 前序+中序构造二叉树：前序首为根，中序分左右，递归构造
TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
    if (preorder.size() == 0 || preorder.size() != inorder.size()) return nullptr;
    unordered_map<int, int> inPos;
    for (int i = 0; i < inorder.size(); i++) inPos[inorder[i]] = i;
    return dfs(preorder, 0, preorder.size()-1, inPos, 0);
}
TreeNode* dfs(vector<int>& pre, int preB, int preE, unordered_map<int,int>& inPos, int inB) {
    if (preB > preE) return nullptr;
    TreeNode* root = new TreeNode(pre[preB]);
    int inM = inPos[pre[preB]];
    int leftL = inM - inB;
    root->left = dfs(pre, preB+1, preB+leftL, inPos, inB);
    root->right = dfs(pre, preB+leftL+1, preE, inPos, inM+1);
    return root;
}
```

### 中序+后序构造二叉树

给定中序和后序遍历（无重复），构造二叉树。

**力扣**：[construct-binary-tree-from-inorder-and-postorder-traversal](https://leetcode-cn.com/problems/construct-binary-tree-from-inorder-and-postorder-traversal)

```cpp
TreeNode* buildTree(vector<int>& inorder, vector<int>& postorder) {
    if (postorder.size() == 0 || postorder.size() != inorder.size()) return nullptr;
    unordered_map<int, int> inPos;
    for (int i = 0; i < inorder.size(); i++) inPos[inorder[i]] = i;
    return dfs(postorder, 0, postorder.size()-1, inPos, 0);
}
TreeNode* dfs(vector<int>& post, int postB, int postE, unordered_map<int,int>& inPos, int inB) {
    if (postB > postE) return nullptr;
    TreeNode* root = new TreeNode(post[postE]);
    int inM = inPos[post[postE]];
    int leftL = inM - inB;
    root->left = dfs(post, postB, postB + leftL - 1, inPos, inB);
    root->right = dfs(post, postB + leftL, postE - 1, inPos, inM + 1);
    return root;
}
```

### 为运算表达式设计优先级

数字和运算符组成的表达式，按不同优先级加括号，返回所有可能结果。

**力扣**：[different-ways-to-add-parentheses](https://leetcode-cn.com/problems/different-ways-to-add-parentheses)

```cpp
vector<int> diffWaysToCompute(string input) {
    vector<int> res;
    for (int i = 0; i < input.size(); i++) {
        char c = input[i];
        if (c == '+' || c == '-' || c == '*') {
            auto left = diffWaysToCompute(input.substr(0, i));
            auto right = diffWaysToCompute(input.substr(i + 1));
            for (int l : left) for (int r : right) {
                if (c == '+') res.push_back(l + r);
                else if (c == '-') res.push_back(l - r);
                else res.push_back(l * r);
            }
        }
    }
    if (res.empty()) res.push_back(stoi(input));
    return res;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-六、DFS---回溯.md]
[src: raw/ingested/2技术/算法/cpp_leetcode技巧-分治递归.md]
[src: raw/ingested/2技术/算法/cpp_leetcode技巧-遍历递归（排列）.md]