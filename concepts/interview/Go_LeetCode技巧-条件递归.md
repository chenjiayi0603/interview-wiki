# Go LeetCode 技巧：条件递归

> 本文档收录了 LeetCode 中与条件递归相关的 Go 语言解题技巧，涵盖括号生成、分割回文串、正则表达式匹配、二叉树构造与验证、翻转、打家劫舍 III 等经典题目。

See also: [[Go_LeetCode技巧]], [[Go语言语法快速入门]], [[Go常用数据结构]]

## 括号生成

[题目链接](https://leetcode-cn.com/problems/generate-parentheses)

n 代表生成括号的对数，生成所有可能的并且 有效的 括号组合。

```go
func generateParenthesis(n int) []string {
	if n == 0 {
		return []string{}
	}
	res := []string{}
	dfs(n, n, "", &res)
	return res
}

func dfs(left, right int, str string, res *[]string) {
	if left == 0 && right == 0 {
		*res = append(*res, str)
		return
	}
	if left > 0 {
		dfs(left-1, right, str+"(", res)
	}
	if left < right {
		dfs(left, right-1, str+")", res)
	}
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]

## 分割回文串

[题目链接](https://leetcode-cn.com/problems/palindrome-partitioning)

给你一个字符串 s，请你将 s 分割成一些子串，使每个子串都是 回文串 。返回 s 所有可能的分割方案。

```go
func partition(s string) [][]string {
    if len(s) == 0 {return [][]string{}}
    path := []string{}
    res := [][]string{}
    dfs(s,0,tmp,&res)
    return res
}

func dfs(s string,pos int,path []string,res *[][]string) {
    if pos == len(s) {
        t := make([]string,len(path))
        copy(t,path)
        *res = append(*res,t)
    }else{
        for i := pos;i < len(s);i++ {
            if isPlalindrome(s,pos,i) {
                path = append(path,s[pos:i+1])
                dfs(s,i+1,tmp,res)
                path = path[:len(path)-1]
            }
        }
    }
}

func isPlalindrome(s string,b,e int) bool {
    for ;b < e;b,e = b + 1,e-1 {
        if s[b] != s[e] {return false}
    }
    return true
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]

## 正则表达式匹配

[题目链接](https://leetcode-cn.com/problems/regular-expression-matching)

给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。

```go
func isMatch(s string, p string) bool {
    return dfs(s,p,0,0)
}

func dfs(s string,p string,i,j int) bool {
    if j >= len(p) {return i == len(s)}
    jmatch := i < len(s) && (s[i] == p[j] || p[j] == '.')
    if j + 1 <len(p) && p[j + 1] == '*' {
        if dfs(s,p,i,j + 2) {return true}
        return jmatch && dfs(s,p,i+1,j)
    }else{
        return jmatch && dfs(s,p,i+1,j+1)
    }
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]

## 从前序与中序遍历序列构造二叉树

[题目链接](https://leetcode-cn.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal)

根据一棵树的前序遍历与中序遍历构造二叉树。

```go
func buildTree(preorder []int, inorder []int) *TreeNode {
    if len(preorder) == 0 || len(preorder) != len(inorder) {return nil}
    inPos := make(map[int]int)
    for i := 0;i < len(inorder);i++{
        inPos[inorder[i]] = i
    }
    return dfs(preorder,0,len(preorder)-1,inPos,0)
}

func dfs(pre []int,preB,preE int,inPos map[int]int,inB int) *TreeNode {
    if preB > preE {return nil}
    root := &TreeNode{Val:pre[preB]}
    inMid := inPos[pre[preB]]
    leftL := inMid - inB
    root.Left = dfs(pre,preB + 1,preB + leftL,inPos,inB)
    root.Right = dfs(pre,preB + leftL + 1,preE,inPos,inMid + 1)
    return root
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]

## 不同的二叉搜索树 II

[题目链接](https://leetcode-cn.com/problems/unique-binary-search-trees-ii)

```go
func generateTrees(n int) []*TreeNode {
	if n == 0 {
		return []*TreeNode{}
	}
	return dfs(1, n)
}

func dfs(b, e int) []*TreeNode {
	tree := []*TreeNode{}
	if b > e {
		tree = append(tree, nil)
		return tree
	}
	for i := b; i <= e; i++ {
		left := dfs(b, i-1)
		right := dfs(i+1, e)
		for _, l := range left {
			for _, r := range right {
				root := &TreeNode{Val: i, Left: l, Right: r}
				tree = append(tree, root)
			}
		}
	}
	return tree
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]

## 验证二叉搜索树

[题目链接](https://leetcode-cn.com/problems/validate-binary-search-tree)

```go
import "math"

func isValidBST(root *TreeNode) bool {
	return isValidbst(root, math.MinInt64, math.MaxInt64)
}
func isValidbst(root *TreeNode, min, max int) bool {
	if root == nil {
		return true
	}
	v := root.Val
	return  v > min && v < max && isValidbst(root.Left, min, v) && isValidbst(root.Right, v, max)
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]

## 翻转二叉树

[题目链接](https://leetcode-cn.com/problems/invert-binary-tree)

```go
func invertTree(root *TreeNode) *TreeNode {
	if root == nil {return nil}
	root.Left, root.Right = root.Right, root.Left
	invertTree(root.Left)
	invertTree(root.Right)
	return root
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]

## 打家劫舍 III

[题目链接](https://leetcode-cn.com/problems/house-robber-iii)

```go
func rob(root *TreeNode) int {
	a, b := dfs(root)
	return max(a, b)
}

func dfs(root *TreeNode) (a, b int) {
	if root == nil {return 0, 0}
	l0, l1 := dfs(root.Left)
	r0, r1 := dfs(root.Right)
	tmp0 := max(l0, l1) + max(r0, r1)
	tmp1 := root.Val + l0 + r0
	return tmp0, tmp1
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-条件递归.md]