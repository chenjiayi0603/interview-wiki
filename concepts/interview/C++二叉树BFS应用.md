# C++ 二叉树 BFS 应用

> 本文涵盖二叉树的层序遍历相关经典问题：二叉树的右视图、二叉树的层平均值、二叉树的锯齿形层次遍历。

See also: [[C++二叉树LCA与路径]], [[C++二叉树遍历]], [[Go_LeetCode技巧-条件递归]]

## 二叉树的右视图

给定一个二叉树的根节点 root，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。

**力扣**：[binary-tree-right-side-view](https://leetcode-cn.com/problems/binary-tree-right-side-view)

```cpp
class Solution {
public:
    vector<int> rightSideView(TreeNode* root) {
        if (!root)return {};
        vector<int> res;
        list<TreeNode*> level{root};
        while (level.size())
        {
        	int s = level.size();
        	while (s--)
        	{
        		if (s == 0)res.push_back(level.front()->val);
        		if (level.front()->left)level.push_back(level.front()->left);
                if (level.front()->right)level.push_back(level.front()->right);
                level.pop_front();
        	}
        }
        return res;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-链表循环迭代.md]

## 二叉树的层平均值

给定一个非空二叉树的根节点 root，以数组的形式返回每一层节点的平均值。

**力扣**：[average-of-levels-in-binary-tree](https://leetcode-cn.com/problems/average-of-levels-in-binary-tree)

```cpp
class Solution {
public:
    vector<double> averageOfLevels(TreeNode* root) {
        if (!root)return {};
        vector<double> res;
        list<TreeNode*> level{root};
        while (level.size())
        {
        	int s = level.size();
            double avg(0.0);
        	for (int i = s;i--;)
        	{
        		avg += level.front()->val;
        		if (i == 0)res.push_back(avg/s);
        		if (level.front()->left)level.push_back(level.front()->left);
                if (level.front()->right)level.push_back(level.front()->right);
                level.pop_front();
        	}
        }
        return res;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-链表循环迭代.md]

## 二叉树的锯齿形层次遍历

给你二叉树的根节点 root，返回其节点值的锯齿形层序遍历。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。

**力扣**：[binary-tree-zigzag-level-order-traversal](https://leetcode-cn.com/problems/binary-tree-zigzag-level-order-traversal)

```cpp
class Solution {
public:
    vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
        if (!root)return {};
        vector<vector<int>> res;
        vector<int> tmp;
        list<TreeNode*> level{root};
        for (int i = 0;level.size();i++)
        {
        	int s = level.size();
        	while (s--)
        	{
        		tmp.push_back(level.front()->val);
        		if (level.front()->left)level.push_back(level.front()->left);
                if (level.front()->right)level.push_back(level.front()->right);
                level.pop_front();
        	}
        	if (i % 2 == 1)reverse(tmp.begin(),tmp.end());
        	res.push_back(tmp);
        	tmp.clear();
        }
        return res; 
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-链表循环迭代.md]