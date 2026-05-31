# C++ 条件递归

> 本文涵盖条件递归的经典 LeetCode 题目：正则表达式匹配、括号生成、二叉搜索树插入操作。

See also: [[C++DFS回溯]], [[C++二叉树LCA与路径]], [[Go_LeetCode技巧-条件递归]]

## 正则表达式匹配

给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。
'.' 匹配任意单个字符
'*' 匹配零个或多个前面的那一个元素
所谓匹配，是要涵盖整个字符串 s 的，而不是部分字符串。

**力扣**：[regular-expression-matching](https://leetcode-cn.com/problems/regular-expression-matching)

*条件，零匹配或多字符匹配递归；.字符或相同字符，单字符匹配递归

```cpp
class Solution {
public:
    bool isMatch(string s, string p) { return dfs(s, p, 0, 0); }
    bool dfs(string& s, string& p, int i, int j) {
        if (j >= p.size()) return i == s.size();
        bool jmatch = i < s.size() && (s[i] == p[j] || p[j] == '.');
        if (j + 1 < p.size() && p[j+1] == '*') {
            if (dfs(s, p, i, j+2)) return true;
            return jmatch && dfs(s, p, i+1, j);
        }
        return jmatch && dfs(s, p, i+1, j+1);
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-条件递归.md]

## 括号生成

数字 n 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 有效的 括号组合。

**力扣**：[generate-parentheses](https://leetcode-cn.com/problems/generate-parentheses)

```cpp
class Solution {
public:
    vector<string> generateParenthesis(int n) {
        vector<string> res;
        dfs(res, "", n, n);
        return res;
    }
    void dfs(vector<string>& res, string s, int l, int r) {
        if (l == 0 && r == 0) { res.push_back(s); return; }
        if (l > 0) dfs(res, s + '(', l - 1, r);
        if (l < r) dfs(res, s + ')', l, r - 1);
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-条件递归.md]

## 二叉搜索树中的插入操作

给定二叉搜索树（BST）的根节点 root 和要插入树中的值 value ，将值插入二叉搜索树。返回插入后二叉搜索树的根节点。输入数据保证新值和原始二叉搜索树中的任意节点值都不同。

**力扣**：[insert-into-a-binary-search-tree](https://leetcode.cn/problems/insert-into-a-binary-search-tree)

```cpp
// 递归写法
class Solution {
public:
    TreeNode* insertIntoBST(TreeNode* root, int val) {
        if (!root) return new TreeNode(val);
        if (root->val > val) root->left = insertIntoBST(root->left, val);
        else root->right = insertIntoBST(root->right, val);
        return root;
    }
};

// 迭代写法
class Solution {
public:
    TreeNode* insertIntoBST(TreeNode* root, int val) {
        if (!root) return new TreeNode(val);
        TreeNode *cur = root, *pre = root;
        while (cur) {
            pre = cur;
            cur = cur->val > val ? cur->left : cur->right;
        }
        (val < pre->val ? pre->left : pre->right) = new TreeNode(val);
        return root;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-条件递归.md]