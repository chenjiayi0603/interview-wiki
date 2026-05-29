# Go LeetCode 技巧：贪心靠近

> 本文档收录了 LeetCode 中与贪心靠近相关的 Go 语言解题技巧，涵盖二叉搜索树最近公共祖先、H 指数等经典题目。

See also: [[Go_LeetCode技巧]], [[Go语言语法快速入门]], [[Go常用数据结构]]

## 二叉搜索树最近公共祖先

[题目链接](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-search-tree)

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	for root != nil {
		if root == nil || root == p|| root == q {
			break
		}
		if math.Max(float64(p.Val),float64(q.Val)) < float64(root.Val){
			root = root.Left//向左 
		} else if math.Min(float64(p.Val),float64(q.Val)) > float64(root.Val){
			root = root.Right//向右
		}else {
			break
		}
	}
	return root
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-贪心靠近.md]

## 从后往前：H 指数

[题目链接](https://leetcode-cn.com/problems/h-index)

给定一位研究者论文被引用次数的数组（被引用次数是非负整数）。编写一个方法，计算出研究者的 h 指数。

h 指数的定义：h 代表“高引用次数”（high citations），一名科研人员的 h 指数是指他（她）的 （N 篇论文中）总共有 h 篇论文分别被引用了至少 h 次。且其余的 N - h 篇论文每篇被引用次数 不超过 h 次。

例如：某人的 h 指数是 20，这表示他已发表的论文中，每篇被引用了至少 20 次的论文总共有 20 篇。

示例：

输入：citations = [3,0,6,1,5]
输出：3 
解释：给定数组表示研究者总共有 5 篇论文，每篇论文相应的被引用了 3, 0, 6, 1, 5 次。由于研究者有 3 篇论文每篇 至少 被引用了 3 次，其余两篇论文每篇被引用 不多于 3 次，所以她的 h 指数是 3。

提示：如果 h 有多种可能的值，h 指数是其中最大的那个。

坐标和数值的关系，利用了有序性。数组成员的值和数组成员的 index 之间的比较计算关系。

```go
func hIndex(citations []int) int {
	sort.Ints(citations)
	h := 0
	for i := len(citations) - 1; i >= 0; i-- {
		if citations[i] >= len(citations)-i {
			h++
		} else {
			break
		}
	}
	return h
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-贪心靠近.md]