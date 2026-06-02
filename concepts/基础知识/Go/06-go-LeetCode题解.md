# LeetCode 题解（Go 实现）

> 排序、回溯、树、动态规划、链表、二分搜索、图、位运算。

---

## 一、排序算法

### 1.1 快速排序

**解决的问题**：O(n log n) 平均时间复杂度的原地排序。

```go
func quickSort(nums []int, l, r int) {
    if l >= r { return }
    pivot := partition(nums, l, r)
    quickSort(nums, l, pivot-1)
    quickSort(nums, pivot+1, r)
}

func partition(nums []int, l, r int) int {
    pivot := nums[r]
    i := l
    for j := l; j < r; j++ {
        if nums[j] < pivot {
            nums[i], nums[j] = nums[j], nums[i]
            i++
        }
    }
    nums[i], nums[r] = nums[r], nums[i]
    return i
}
```

### 1.2 堆排序

```go
func heapSort(nums []int) {
    n := len(nums)
    // 建堆
    for i := n/2 - 1; i >= 0; i-- {
        heapify(nums, n, i)
    }
    // 逐个取出最大值
    for i := n - 1; i > 0; i-- {
        nums[0], nums[i] = nums[i], nums[0]
        heapify(nums, i, 0)
    }
}

func heapify(nums []int, n, i int) {
    largest := i
    left, right := 2*i+1, 2*i+2
    if left < n && nums[left] > nums[largest] { largest = left }
    if right < n && nums[right] > nums[largest] { largest = right }
    if largest != i {
        nums[i], nums[largest] = nums[largest], nums[i]
        heapify(nums, n, largest)
    }
}
```

---

## 二、回溯算法

### 2.1 全排列

**LeetCode**: [46. 全排列](https://leetcode.cn/problems/permutations/)

```go
func permute(nums []int) [][]int {
    var result [][]int
    var track []int
    used := make([]bool, len(nums))
    backtrack(nums, track, used, &result)
    return result
}

func backtrack(nums, track []int, used []bool, result *[][]int) {
    if len(track) == len(nums) {
        tmp := make([]int, len(track))
        copy(tmp, track)
        *result = append(*result, tmp)
        return
    }
    for i := 0; i < len(nums); i++ {
        if used[i] { continue }
        used[i] = true
        track = append(track, nums[i])
        backtrack(nums, track, used, result)
        track = track[:len(track)-1]
        used[i] = false
    }
}
```

### 2.2 全排列 II（含重复元素）

**LeetCode**: [47. 全排列 II](https://leetcode.cn/problems/permutations-ii/)

```go
func permuteUnique(nums []int) [][]int {
    sort.Ints(nums)
    var result [][]int
    var track []int
    used := make([]bool, len(nums))
    backtrackUnique(nums, track, used, &result)
    return result
}

func backtrackUnique(nums, track []int, used []bool, result *[][]int) {
    if len(track) == len(nums) {
        tmp := make([]int, len(track))
        copy(tmp, track)
        *result = append(*result, tmp)
        return
    }
    for i := 0; i < len(nums); i++ {
        if used[i] { continue }
        // 去重：前一个相同元素未使用 → 跳过
        if i > 0 && nums[i] == nums[i-1] && !used[i-1] { continue }
        used[i] = true
        track = append(track, nums[i])
        backtrackUnique(nums, track, used, result)
        track = track[:len(track)-1]
        used[i] = false
    }
}
```

### 2.3 子集

**LeetCode**: [78. 子集](https://leetcode.cn/problems/subsets/)

```go
func subsets(nums []int) [][]int {
    var result [][]int
    var track []int
    backtrackSub(nums, 0, track, &result)
    return result
}

func backtrackSub(nums []int, start int, track []int, result *[][]int) {
    tmp := make([]int, len(track))
    copy(tmp, track)
    *result = append(*result, tmp)

    for i := start; i < len(nums); i++ {
        track = append(track, nums[i])
        backtrackSub(nums, i+1, track, result)
        track = track[:len(track)-1]
    }
}
```

### 2.4 括号生成

**LeetCode**: [22. 括号生成](https://leetcode.cn/problems/generate-parentheses/)

```go
func generateParenthesis(n int) []string {
    var result []string
    backtrackParen(&result, "", 0, 0, n)
    return result
}

func backtrackParen(result *[]string, cur string, open, close, max int) {
    if len(cur) == max*2 {
        *result = append(*result, cur)
        return
    }
    if open < max {
        backtrackParen(result, cur+"(", open+1, close, max)
    }
    if close < open {
        backtrackParen(result, cur+")", open, close+1, max)
    }
}
```

### 2.5 分割回文串

**LeetCode**: [131. 分割回文串](https://leetcode.cn/problems/palindrome-partitioning/)

```go
func partition(s string) [][]string {
    var result [][]string
    var track []string
    backtrackPart(s, 0, track, &result)
    return result
}

func backtrackPart(s string, start int, track []string, result *[][]string) {
    if start == len(s) {
        tmp := make([]string, len(track))
        copy(tmp, track)
        *result = append(*result, tmp)
        return
    }
    for end := start; end < len(s); end++ {
        if isPalindrome(s, start, end) {
            track = append(track, s[start:end+1])
            backtrackPart(s, end+1, track, result)
            track = track[:len(track)-1]
        }
    }
}

func isPalindrome(s string, l, r int) bool {
    for l < r {
        if s[l] != s[r] { return false }
        l++; r--
    }
    return true
}
```

---

## 三、树

### 3.1 二叉树遍历

**LeetCode**: [144. 前序遍历](https://leetcode.cn/problems/binary-tree-preorder-traversal/) · [94. 中序遍历](https://leetcode.cn/problems/binary-tree-inorder-traversal/) · [145. 后序遍历](https://leetcode.cn/problems/binary-tree-postorder-traversal/)

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// 前序递归
func preorderRecur(root *TreeNode) []int {
    var result []int
    var dfs func(*TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil { return }
        result = append(result, node.Val)
        dfs(node.Left)
        dfs(node.Right)
    }
    dfs(root)
    return result
}

// 前序迭代
func preorderIter(root *TreeNode) []int {
    if root == nil { return nil }
    var result []int
    stack := []*TreeNode{root}
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)
        if node.Right != nil { stack = append(stack, node.Right) }
        if node.Left != nil { stack = append(stack, node.Left) }
    }
    return result
}

// 中序迭代
func inorderIter(root *TreeNode) []int {
    var result []int
    stack := []*TreeNode{}
    cur := root
    for cur != nil || len(stack) > 0 {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, cur.Val)
        cur = cur.Right
    }
    return result
}

// 后序迭代（前序的变体：根右左 → 反转）
func postorderIter(root *TreeNode) []int {
    if root == nil { return nil }
    var result []int
    stack := []*TreeNode{root}
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)
        if node.Left != nil { stack = append(stack, node.Left) }
        if node.Right != nil { stack = append(stack, node.Right) }
    }
    // 反转
    for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
        result[i], result[j] = result[j], result[i]
    }
    return result
}
```

### 3.2 二叉树的最大深度

**LeetCode**: [104. 二叉树的最大深度](https://leetcode.cn/problems/maximum-depth-of-binary-tree/)

```go
func maxDepth(root *TreeNode) int {
    if root == nil { return 0 }
    return 1 + max(maxDepth(root.Left), maxDepth(root.Right))
}
func max(a, b int) int { if a > b { return a }; return b }
```

### 3.3 验证二叉搜索树

**LeetCode**: [98. 验证二叉搜索树](https://leetcode.cn/problems/validate-binary-search-tree/)

```go
func isValidBST(root *TreeNode) bool {
    return validate(root, math.MinInt64, math.MaxInt64)
}

func validate(node *TreeNode, min, max int) bool {
    if node == nil { return true }
    if node.Val <= min || node.Val >= max { return false }
    return validate(node.Left, min, node.Val) &&
           validate(node.Right, node.Val, max)
}
```

### 3.4 翻转二叉树

**LeetCode**: [226. 翻转二叉树](https://leetcode.cn/problems/invert-binary-tree/)

```go
func invertTree(root *TreeNode) *TreeNode {
    if root == nil { return nil }
    root.Left, root.Right = invertTree(root.Right), invertTree(root.Left)
    return root
}
```

### 3.5 二叉树的最近公共祖先

**LeetCode**: [236. 二叉树的最近公共祖先](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree/)

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q { return root }
    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)
    if left != nil && right != nil { return root }
    if left != nil { return left }
    return right
}
```

### 3.6 路径总和 III

**LeetCode**: [437. 路径总和 III](https://leetcode.cn/problems/path-sum-iii/)

```go
func pathSum(root *TreeNode, targetSum int) int {
    prefix := map[int]int{0: 1}
    return dfs(root, prefix, 0, targetSum)
}

func dfs(node *TreeNode, prefix map[int]int, cur, target int) int {
    if node == nil { return 0 }
    cur += node.Val
    count := prefix[cur-target]
    prefix[cur]++
    count += dfs(node.Left, prefix, cur, target)
    count += dfs(node.Right, prefix, cur, target)
    prefix[cur]--
    return count
}
```

---

## 四、动态规划

### 4.1 最长递增子序列（LIS）

**LeetCode**: [300. 最长递增子序列](https://leetcode.cn/problems/longest-increasing-subsequence/)

```go
func lengthOfLIS(nums []int) int {
    dp := make([]int, len(nums))
    for i := range dp { dp[i] = 1 }
    maxLen := 1
    for i := 0; i < len(nums); i++ {
        for j := 0; j < i; j++ {
            if nums[j] < nums[i] {
                dp[i] = max(dp[i], dp[j]+1)
            }
        }
        maxLen = max(maxLen, dp[i])
    }
    return maxLen
}

// 优化：贪心+二分 O(n log n)
func lengthOfLIS2(nums []int) int {
    tails := []int{}
    for _, x := range nums {
        i := sort.SearchInts(tails, x)
        if i == len(tails) {
            tails = append(tails, x)
        } else {
            tails[i] = x
        }
    }
    return len(tails)
}
```

### 4.2 最长回文子串

**LeetCode**: [5. 最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)

```go
func longestPalindrome(s string) string {
    n := len(s)
    if n < 2 { return s }
    start, maxLen := 0, 1

    for i := 0; i < n; i++ {
        // 奇数长度
        l, r := i-1, i+1
        for l >= 0 && r < n && s[l] == s[r] {
            if r-l+1 > maxLen {
                start, maxLen = l, r-l+1
            }
            l--; r++
        }
        // 偶数长度
        l, r = i, i+1
        for l >= 0 && r < n && s[l] == s[r] {
            if r-l+1 > maxLen {
                start, maxLen = l, r-l+1
            }
            l--; r++
        }
    }
    return s[start : start+maxLen]
}
```

### 4.3 爬楼梯

**LeetCode**: [70. 爬楼梯](https://leetcode.cn/problems/climbing-stairs/)

```go
func climbStairs(n int) int {
    if n <= 2 { return n }
    a, b := 1, 2
    for i := 3; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
```

### 4.4 打家劫舍 III（树形 DP）

**LeetCode**: [337. 打家劫舍 III](https://leetcode.cn/problems/house-robber-iii/)

```go
func rob(root *TreeNode) int {
    robRoot, notRobRoot := dfsRob(root)
    return max(robRoot, notRobRoot)
}

// 返回 [偷当前节点, 不偷当前节点]
func dfsRob(node *TreeNode) (int, int) {
    if node == nil { return 0, 0 }
    leftRob, leftNot := dfsRob(node.Left)
    rightRob, rightNot := dfsRob(node.Right)

    rob := node.Val + leftNot + rightNot
    notRob := max(leftRob, leftNot) + max(rightRob, rightNot)
    return rob, notRob
}
```

### 4.5 完全平方数

**LeetCode**: [279. 完全平方数](https://leetcode.cn/problems/perfect-squares/)

```go
func numSquares(n int) int {
    dp := make([]int, n+1)
    for i := 1; i <= n; i++ {
        dp[i] = i  // 最坏：全由 1 组成
        for j := 1; j*j <= i; j++ {
            dp[i] = min(dp[i], dp[i-j*j]+1)
        }
    }
    return dp[n]
}
func min(a, b int) int { if a < b { return a }; return b }
```

### 4.6 最小路径和

**LeetCode**: [64. 最小路径和](https://leetcode.cn/problems/minimum-path-sum/)

```go
func minPathSum(grid [][]int) int {
    m, n := len(grid), len(grid[0])
    dp := make([][]int, m)
    for i := range dp { dp[i] = make([]int, n) }

    dp[0][0] = grid[0][0]
    for i := 1; i < m; i++ { dp[i][0] = dp[i-1][0] + grid[i][0] }
    for j := 1; j < n; j++ { dp[0][j] = dp[0][j-1] + grid[0][j] }

    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            dp[i][j] = min(dp[i-1][j], dp[i][j-1]) + grid[i][j]
        }
    }
    return dp[m-1][n-1]
}
```

---

## 五、链表

### 5.1 合并有序链表

**LeetCode**: [21. 合并两个有序链表](https://leetcode.cn/problems/merge-two-sorted-lists/)

```go
func mergeTwoLists(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    cur := dummy
    for l1 != nil && l2 != nil {
        if l1.Val < l2.Val {
            cur.Next = l1
            l1 = l1.Next
        } else {
            cur.Next = l2
            l2 = l2.Next
        }
        cur = cur.Next
    }
    if l1 != nil { cur.Next = l1 }
    if l2 != nil { cur.Next = l2 }
    return dummy.Next
}
```

### 5.2 反转链表

**LeetCode**: [206. 反转链表](https://leetcode.cn/problems/reverse-linked-list/)

```go
// 迭代
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    cur := head
    for cur != nil {
        next := cur.Next
        cur.Next = prev
        prev, cur = cur, next
    }
    return prev
}

// 递归
func reverseListRecur(head *ListNode) *ListNode {
    if head == nil || head.Next == nil { return head }
    newHead := reverseListRecur(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}
```

### 5.3 LRU 缓存

**LeetCode**: [146. LRU 缓存](https://leetcode.cn/problems/lru-cache/)

```go
type LRUCache struct {
    cap  int
    m    map[int]*list.Element
    list *list.List
}

type entry struct {
    key   int
    value int
}

func Constructor(capacity int) LRUCache {
    return LRUCache{
        cap:  capacity,
        m:    make(map[int]*list.Element),
        list: list.New(),
    }
}

func (c *LRUCache) Get(key int) int {
    if elem, ok := c.m[key]; ok {
        c.list.MoveToFront(elem)
        return elem.Value.(*entry).value
    }
    return -1
}

func (c *LRUCache) Put(key int, value int) {
    if elem, ok := c.m[key]; ok {
        elem.Value.(*entry).value = value
        c.list.MoveToFront(elem)
        return
    }
    if c.list.Len() >= c.cap {
        tail := c.list.Back()
        delete(c.m, tail.Value.(*entry).key)
        c.list.Remove(tail)
    }
    elem := c.list.PushFront(&entry{key, value})
    c.m[key] = elem
}
```

---

## 六、二分搜索

### 6.1 基本二分

```go
func binarySearch(nums []int, target int) int {
    l, r := 0, len(nums)-1
    for l <= r {
        mid := l + (r-l)/2
        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            l = mid + 1
        } else {
            r = mid - 1
        }
    }
    return -1
}
```

### 6.2 旋转数组搜索

**LeetCode**: [33. 搜索旋转排序数组](https://leetcode.cn/problems/search-in-rotated-sorted-array/)

```go
func search(nums []int, target int) int {
    l, r := 0, len(nums)-1
    for l <= r {
        mid := l + (r-l)/2
        if nums[mid] == target { return mid }

        if nums[l] <= nums[mid] {  // 左半有序
            if nums[l] <= target && target < nums[mid] {
                r = mid - 1
            } else {
                l = mid + 1
            }
        } else {  // 右半有序
            if nums[mid] < target && target <= nums[r] {
                l = mid + 1
            } else {
                r = mid - 1
            }
        }
    }
    return -1
}
```

### 6.3 H 指数 II

**LeetCode**: [275. H 指数 II](https://leetcode.cn/problems/h-index-ii/)

```go
func hIndex(citations []int) int {
    n := len(citations)
    l, r := 0, n-1
    for l <= r {
        mid := l + (r-l)/2
        if citations[mid] >= n-mid {
            r = mid - 1
        } else {
            l = mid + 1
        }
    }
    return n - l
}
```

---

## 七、图

### 7.1 拓扑排序

**LeetCode**: [207. 课程表](https://leetcode.cn/problems/course-schedule/) · [210. 课程表 II](https://leetcode.cn/problems/course-schedule-ii/)

```go
func topoSort(numCourses int, prerequisites [][]int) []int {
    graph := make([][]int, numCourses)
    inDegree := make([]int, numCourses)

    for _, pre := range prerequisites {
        course, require := pre[0], pre[1]
        graph[require] = append(graph[require], course)
        inDegree[course]++
    }

    queue := []int{}
    for i := 0; i < numCourses; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    result := []int{}
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        result = append(result, node)
        for _, neighbor := range graph[node] {
            inDegree[neighbor]--
            if inDegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }
    if len(result) != numCourses { return nil }  // 有环
    return result
}
```

---

## 八、位运算

### 8.1 常用操作

```go
// 判断第 i 位是否为 1
func getBit(num, i int) bool {
    return (num >> i) & 1 == 1
}

// 设置第 i 位为 1
func setBit(num, i int) int {
    return num | (1 << i)
}

// 清除第 i 位
func clearBit(num, i int) int {
    return num & ^(1 << i)
}

// 统计 1 的个数
func popCount(x int) int {
    count := 0
    for x != 0 {
        x = x & (x - 1)  // 清除最低位的 1
        count++
    }
    return count
}

// 判断 2 的幂
func isPowerOfTwo(n int) bool {
    return n > 0 && n&(n-1) == 0
}
```

### 8.2 只出现一次的数字

**LeetCode**: [136. 只出现一次的数字](https://leetcode.cn/problems/single-number/)

```go
func singleNumber(nums []int) int {
    result := 0
    for _, n := range nums {
        result ^= n
    }
    return result
}
```

---

## 九、其他高频题

### 9.1 盛最多水的容器

**LeetCode**: [11. 盛最多水的容器](https://leetcode.cn/problems/container-with-most-water/)

```go
func maxArea(height []int) int {
    l, r := 0, len(height)-1
    maxWater := 0
    for l < r {
        water := min(height[l], height[r]) * (r - l)
        maxWater = max(maxWater, water)
        if height[l] < height[r] {
            l++
        } else {
            r--
        }
    }
    return maxWater
}
```

### 9.2 三数之和

**LeetCode**: [15. 三数之和](https://leetcode.cn/problems/3sum/)

```go
func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    var result [][]int
    for i := 0; i < len(nums)-2; i++ {
        if i > 0 && nums[i] == nums[i-1] { continue }
        l, r := i+1, len(nums)-1
        target := -nums[i]
        for l < r {
            sum := nums[l] + nums[r]
            if sum == target {
                result = append(result, []int{nums[i], nums[l], nums[r]})
                for l < r && nums[l] == nums[l+1] { l++ }
                for l < r && nums[r] == nums[r-1] { r-- }
                l++; r--
            } else if sum < target {
                l++
            } else {
                r--
            }
        }
    }
    return result
}
```

### 9.3 跳表

**LeetCode**: [1206. 设计跳表](https://leetcode.cn/problems/design-skiplist/)

```go
type SkiplistNode struct {
    Val    int
    Next   []*SkiplistNode  // 每层的下一个节点
}

type Skiplist struct {
    head  *SkiplistNode
    level int
    p     float64  // 晋升概率
}

func Constructor() Skiplist {
    return Skiplist{
        head:  &SkiplistNode{Val: -1, Next: make([]*SkiplistNode, maxLevel)},
        level: 1,
        p:     0.25,
    }
}

func (s *Skiplist) randomLevel() int {
    level := 1
    for rand.Float64() < s.p && level < maxLevel {
        level++
    }
    return level
}

// Search/Add/Erase 实现略（标准跳表操作）
```
