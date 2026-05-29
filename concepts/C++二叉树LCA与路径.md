# C++ 二叉树 LCA 与路径

> 本文涵盖二叉树最近公共祖先（LCA）、最大路径和、最大深度、最小深度、BST 第 K 小、路径总和 III、BST 插入等经典问题。

See also: [[C++二叉树遍历]], [[C++二叉树BFS应用]], [[Go_LeetCode技巧-条件递归]]

## 二叉树最近公共祖先

给定二叉树和两个节点 p、q，求它们的最近公共祖先节点。

**力扣**：[lowest-common-ancestor-of-a-binary-tree](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree)

```cpp
// 二叉树最近公共祖先：p/q 在左右则 root 为 LCA，否则在单侧继续找
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;
    auto L = lowestCommonAncestor(root->left, p, q);
    auto R = lowestCommonAncestor(root->right, p, q);
    return L && R ? root : (L ? L : R);
}
```

## BST 最近公共祖先

在二叉搜索树中求两节点的最近公共祖先。

**力扣**：[lowest-common-ancestor-of-a-binary-search-tree](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-search-tree)

```cpp
// BST 最近公共祖先：p、q 都小于 root 往左，都大于往右，否则 root 即 LCA
TreeNode* lowestCommonAncestorBST(TreeNode* root, TreeNode* p, TreeNode* q) {
    while (root) {
        if (max(p->val, q->val) < root->val) root = root->left;
        else if (min(p->val, q->val) > root->val) root = root->right;
        else return root;
    }
    return root;
}
```

## 二叉树最大路径和

求二叉树中路径上节点值之和的最大值（路径可不过根）。

**力扣**：[binary-tree-maximum-path-sum](https://leetcode-cn.com/problems/binary-tree-maximum-path-sum)

```cpp
// 二叉树最大路径和：经过某节点的路径=左单边+右单边+节点值，取 max
int maxPathSum(TreeNode* root) {
    int ans = INT_MIN;
    function<int(TreeNode*)> dfs = [&](TreeNode* node) {
        if (!node) return 0;
        int L = max(0, dfs(node->left));
        int R = max(0, dfs(node->right));
        ans = max(ans, node->val + L + R);  // 经过当前节点的路径
        return node->val + max(L, R);       // 返回单边最大
    };
    dfs(root);
    return ans;
}
```

## 二叉树最大深度

求从根到最远叶子节点的路径长度。

**力扣**：[maximum-depth-of-binary-tree](https://leetcode-cn.com/problems/maximum-depth-of-binary-tree)

```cpp
// 二叉树最大深度
int maxDepth(TreeNode* root) {
    if (!root) return 0;
    return max(maxDepth(root->left), maxDepth(root->right)) + 1;
}
```

## BST 第 K 小

在二叉搜索树中求第 k 小的元素值。

**力扣**：[kth-smallest-element-in-a-bst](https://leetcode-cn.com/problems/kth-smallest-element-in-a-bst)

```cpp
// BST 第 K 小：中序遍历第 k 个
int kthSmallest(TreeNode* root, int k) {
    stack<TreeNode*> stk;
    TreeNode* p = root;
    int cnt = 0;
    while (p || !stk.empty()) {
        while (p) { stk.push(p); p = p->left; }
        p = stk.top(); stk.pop();
        if (++cnt == k) return p->val;
        p = p->right;
    }
    return 0;
}
```

## 二叉树最小深度

求从根到最近叶子节点的最短路径长度。

**力扣**：[minimum-depth-of-binary-tree](https://leetcode-cn.com/problems/minimum-depth-of-binary-tree)

```cpp
// 二叉树最小深度：叶子节点才算，单子节点需走有子的一侧
int minDepth(TreeNode* root) {
    if (!root) return 0;
    if (!root->left) return minDepth(root->right) + 1;
    if (!root->right) return minDepth(root->left) + 1;
    return min(minDepth(root->left), minDepth(root->right)) + 1;
}
```

## 路径总和 III

路径和等于 targetSum 的路径数目，路径可不过根、可不过叶，方向向下。

**力扣**：[path-sum-iii](https://leetcode-cn.com/problems/path-sum-iii)

```cpp
int pathSum(TreeNode* root, int sum) {
    if (!root) return 0;
    return findPath(root, 0, sum) + pathSum(root->left, sum) + pathSum(root->right, sum);
}
int findPath(TreeNode* node, int curSum, int sum) {
    if (!node) return 0;
    curSum += node->val;
    return (curSum == sum) + findPath(node->left, curSum, sum) + findPath(node->right, curSum, sum);
}
```

## BST 插入

在二叉搜索树中插入新值，保持 BST 性质。

**力扣**：[insert-into-a-binary-search-tree](https://leetcode-cn.com/problems/insert-into-a-binary-search-tree)

```cpp
TreeNode* insertIntoBST(TreeNode* root, int val) {
    if (!root) return new TreeNode(val);
    if (root->val > val) root->left = insertIntoBST(root->left, val);
    else root->right = insertIntoBST(root->right, val);
    return root;
}
```

## 二叉树中序遍历（迭代与递归）

**力扣**：[binary-tree-inorder-traversal](https://leetcode-cn.com/problems/binary-tree-inorder-traversal)

```cpp
// 迭代：使用栈，往左遍历树，并压栈，弹栈即访问
class Solution {
public:
    vector<int> inorderTraversal(TreeNode* root) {
        vector<int> res;
        stack<TreeNode*> s;
        TreeNode* p = root;
        while (p || !s.empty()) {
            while (p) { s.push(p); p = p->left; }
            p = s.top(); s.pop();
            res.push_back(p->val);
            p = p->right;
        }
        return res;
    }
};

// 递归
class Solution {
public:
    vector<int> inorderTraversal(TreeNode* root) {
        vector<int> res;
        dfs(root, res);
        return res;
    }
    void dfs(TreeNode* r, vector<int>& res) {
        if (r) { dfs(r->left, res); res.push_back(r->val); dfs(r->right, res); }
    }
};
```

## 二叉树后序遍历（迭代与递归）

**力扣**：[binary-tree-postorder-traversal](https://leetcode-cn.com/problems/binary-tree-postorder-traversal)

```cpp
// 迭代（根右左再reverse）
class Solution {
public:
    vector<int> postorderTraversal(TreeNode* root) {
        vector<int> res;
        stack<TreeNode*> s;
        TreeNode* p = root;
        while (p || !s.empty()) {
            while (p) { res.push_back(p->val); s.push(p); p = p->right; }
            p = s.top(); s.pop();
            p = p->left;
        }
        reverse(res.begin(), res.end());
        return res;
    }
};

// 递归
class Solution {
public:
    vector<int> postorderTraversal(TreeNode* root) {
        vector<int> res;
        dfs(root, res);
        return res;
    }
    void dfs(TreeNode* r, vector<int>& res) {
        if (r) { dfs(r->left, res); dfs(r->right, res); res.push_back(r->val); }
    }
};
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-五、二叉树.md]
[src: raw/ingested/2技术/算法/cpp_leetcode技巧-二叉树递归.md]