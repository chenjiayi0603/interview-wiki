# C++ 二叉树前序遍历

## 题目

[二叉树的前序遍历](https://leetcode-cn.com/problems/binary-tree-preorder-traversal)

给你二叉树的根节点 root ，返回它节点值的 前序 遍历。

示例 1：
输入：root = [1,null,2,3]
输出：[1,2,3]

示例 2：
输入：root = []
输出：[]

示例 3：
输入：root = [1]
输出：[1]

示例 4：
输入：root = [1,2]
输出：[1,2]

示例 5：
输入：root = [1,null,2]
输出：[1,2]

提示：
树中节点数目在范围 [0, 100] 内
-100 <= Node.val <= 100
进阶：递归算法很简单，你可以通过迭代算法完成吗？

## 解法

本节点访问、压右、压左

### 栈方法1

```cpp
class Solution {
public:
    vector<int> preorderTraversal(TreeNode* root) {
        if(!root)return {};
        vector<int> res;
        stack<TreeNode*> s;
        TreeNode* p = root;
        while(p ||s.size())
        {
            while(p)
            {
                res.push_back(p->val);//先序
                s.push(p);
                p = p->left;//左
            }
            p = s.top();
            s.pop();
            p = p->right;//右
        }
        return res;
    }
};
```

### 递归

```cpp
class Solution {
public:
    vector<int> preorderTraversal(TreeNode* root) {
        vector<int> res;
        dfs(root,res);
        return res;
    }
    void dfs(TreeNode* root,vector<int>& output) {
        if (root != nullptr) {
        	output.push_back(root->val);
            dfs(root->left, output);
            dfs(root->right, output);
        }
    }
};
```

### 栈方法2

```cpp
class Solution {
public:
    vector<int> preorderTraversal(TreeNode* root) {
        if (root == nullptr)return {};
        vector<int> res;
        list<TreeNode*> l;
        l.push_back(root);
        while (l.size() > 0){
            auto node = l.back();
            l.pop_back();
            res.push_back(node->val);
            if (node->right)l.push_back(node->right);//先压栈的后访问
            if (node->left)l.push_back(node->left);
        }
        return res;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-二叉树的前序遍历.md]