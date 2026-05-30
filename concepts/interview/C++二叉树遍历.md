# C++ 二叉树遍历

> 二叉树遍历包括前序、中序、后序、层序四种基本方式，本文给出迭代实现。

See also: [[C++二叉树LCA与路径]], [[C++二叉树BFS应用]], [[Go_LeetCode技巧-条件递归]]

## 前序遍历

按根-左-右顺序访问二叉树所有节点。

**力扣**：[binary-tree-preorder-traversal](https://leetcode-cn.com/problems/binary-tree-preorder-traversal)

```cpp
// 前序（迭代）：根-左-右，先压右再压左
vector<int> preorder(TreeNode* root) {
    vector<int> res;
    stack<TreeNode*> stk;
    if (root) stk.push(root);
    while (!stk.empty()) {
        auto node = stk.top(); stk.pop();
        res.push_back(node->val);
        if (node->right) stk.push(node->right);
        if (node->left) stk.push(node->left);
    }
    return res;
}
```

## 中序遍历

按左-根-右顺序访问二叉树所有节点。

**力扣**：[binary-tree-inorder-traversal](https://leetcode-cn.com/problems/binary-tree-inorder-traversal)

```cpp
// 中序（迭代）：左-根-右，往左压栈，弹栈即访问
vector<int> inorder(TreeNode* root) {
    vector<int> res;
    stack<TreeNode*> stk;
    TreeNode* cur = root;
    while (cur || !stk.empty()) {
        while (cur) { stk.push(cur); cur = cur->left; }
        cur = stk.top(); stk.pop();
        res.push_back(cur->val);
        cur = cur->right;
    }
    return res;
}
```

## 后序遍历

按左-右-根顺序访问二叉树所有节点。

**力扣**：[binary-tree-postorder-traversal](https://leetcode-cn.com/problems/binary-tree-postorder-traversal)

```cpp
// 后序（迭代）：右-左-根，再 reverse
vector<int> postorder(TreeNode* root) {
    if (!root) return {};
    vector<int> res;
    stack<TreeNode*> s;
    TreeNode* p = root;
    while (p || !s.empty()) {
        while (p) {
            res.push_back(p->val);
            s.push(p);
            p = p->right;
        }
        p = s.top(); s.pop();
        p = p->left;
    }
    reverse(res.begin(), res.end());
    return res;
}
```

## 层序遍历

按从上到下、每层从左到右顺序访问所有节点。

**力扣**：[binary-tree-level-order-traversal](https://leetcode-cn.com/problems/binary-tree-level-order-traversal)

```cpp
// 层序遍历
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res;
    if (!root) return res;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int sz = q.size();
        vector<int> level;
        for (int i = 0; i < sz; i++) {
            auto node = q.front(); q.pop();
            level.push_back(node->val);
            if (node->left) q.push(node->left);
            if (node->right) q.push(node->right);
        }
        res.push_back(level);
    }
    return res;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-五、二叉树.md]