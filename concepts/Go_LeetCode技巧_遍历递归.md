# Go LeetCode 技巧：遍历递归

> 本文档收录了 LeetCode 中与遍历递归相关的 Go 语言解题技巧，涵盖全排列、全排列 II、路径总和 III 等经典题目。

See also: [[Go_LeetCode技巧]], [[Go语言语法快速入门]], [[Go常用数据结构]]

## 全排列

[题目链接](https://leetcode-cn.com/problems/permutations)

解法1：替换
```go
func permute(nums []int) [][]int {
    res := [][]int {}
    dfs(nums,0,&res);
    return res;
}
func dfs(nums []int,pos int,res *[][]int){
    if (pos == len(nums))    {
        temp := make([]int, len(nums))
		copy(temp, nums)
        *res = append(*res,temp)
        return;
    }
    for i := pos;i < len(nums);i++    {
        nums[pos],nums[i] = nums[i],nums[pos]
        dfs(nums,pos+1,res);
        nums[pos],nums[i] = nums[i],nums[pos]
    }
}
```

解法2：记录
```go
func permute(nums []int) [][]int {
	if len(nums) == 0 {
		return [][]int{}
	}
	used, p, res := make([]bool, len(nums)), []int{}, [][]int{}
	dfs(nums, 0, p, &res, used)
	return res
}

func dfs(nums []int, pos int, p []int, res *[][]int, used []bool) {
	if pos == len(nums) {
		temp := make([]int, len(p))
		copy(temp, p)
		*res = append(*res, temp)
		return
	}
	for i := 0; i < len(nums); i++ {
		if !used[i] {
			used[i] = true
			p = append(p, nums[i])
			dfs(nums, pos+1, p, res, used)
			p = p[:len(p)-1]
			used[i] = false
		}
	}
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-遍历递归.md]

## 全排列 II

[题目链接](https://leetcode-cn.com/problems/permutations-ii)

给定一个可包含重复数字的序列 nums ，按任意顺序 返回所有不重复的全排列。

示例 1：
输入：nums = [1,1,2]
输出：
[[1,1,2],
 [1,2,1],
 [2,1,1]]

示例 2：
输入：nums = [1,2,3]
输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
 
提示：
1 <= nums.length <= 8
-10 <= nums[i] <= 10

```go
func permuteUnique(nums []int) [][]int {
	if len(nums) == 0 {
		return [][]int{}
	}
	sort.Ints(nums)
    check, path, res := make([]bool, len(nums)), []int{}, [][]int{}
	dfs(nums, 0, path, &res, check)
	return res
}

func dfs(nums []int, pos int, path []int, res *[][]int, check []bool) {
	if pos == len(nums) {
		tmp := make([]int, len(path))//为了生存周期
		copy(tmp, path)
		*res = append(*res, tmp)
		return
	}
	for i := 0; i < len(nums); i++ {
		if !check[i] {
			if i > 0 && nums[i] == nums[i-1] &&!check[i-1]{//去重,相同的之前要访问过才可能有
				continue
			}
			check[i] = true
			path = append(path, nums[i])
			dfs(nums, pos+1, path, res, check)
			path = path[:len(path)-1]
			check[i] = false
		}
	}
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-遍历递归.md]

## 路径总和 III

[题目链接](https://leetcode-cn.com/problems/path-sum-iii)    
给定一个二叉树，它的每个结点都存放着一个整数值。 
找出路径和等于给定数值的路径总数。
路径不需要从根节点开始，也不需要在叶子节点结束，但是路径方向必须是向下的（只能从父节点到子节点）。

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func pathSum(root *TreeNode, targetSum int) int {
    if root == nil {return 0}
    res := findPath(root,0,targetSum) + pathSum(root.Left,targetSum) + pathSum(root.Right,targetSum)
    return res
}
func findPath(root *TreeNode, curSum int,sum int) int {
    if root == nil {return 0}
    curSum += root.Val //上到下
    if curSum == sum {
        return 1 + findPath(root.Left,curSum,sum) + findPath(root.Right,curSum,sum)  
    }else{
        return findPath(root.Left,curSum,sum) + findPath(root.Right,curSum,sum)  
    }    
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-遍历递归.md]