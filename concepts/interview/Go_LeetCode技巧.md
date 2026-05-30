# Go LeetCode 技巧

> Go 语言刷 LeetCode 常用技巧总结，涵盖位运算、数据结构、算法模板等。

See also: [[Go语言基础]], [[Go常用数据结构]], [[Slice与Map]], [[Go高频面试问题]]

---

## 位运算

### 判断奇偶

```go
// X & 1 == 1 判断是否是奇数（偶数）
```

[src: raw/ingested/2技术/go/go_leetcode技巧-[课程表](https---leetcode-cn.com-problems-course-schedule).md]

### 抹掉最低位的1

```go
// n & (n-1) 抹掉最低位的1
```

### 异或找缺失数字

```go
// a ^ a = 0, a ^ 0 = a
// 利用异或找出只出现一次的数字
```

---

## 图论：课程表（拓扑排序）

### 问题描述

你这个学期必须选修 numCourses 门课程，记为 0 到 numCourses - 1。

在选修某些课程之前需要一些先修课程。先修课程按数组 prerequisites 给出，其中 prerequisites[i] = [ai, bi]，表示如果要学习课程 ai 则必须先学习课程 bi。

例如，先修课程对 [0, 1] 表示：想要学习课程 0，你需要先完成课程 1。

请你判断是否可能完成所有课程的学习？如果可以，返回 true；否则，返回 false。

### 示例

**示例 1：**

输入：numCourses = 2, prerequisites = [[1,0]]
输出：true
解释：总共有 2 门课程。学习课程 1 之前，你需要完成课程 0。这是可能的。

**示例 2：**

输入：numCourses = 2, prerequisites = [[1,0],[0,1]]
输出：false
解释：总共有 2 门课程。学习课程 1 之前，你需要先完成​课程 0；并且学习课程 0 之前，你还应先完成课程 1。这是不可能的。

### 提示

- 1 <= numCourses <= 105
- 0 <= prerequisites.length <= 5000
- prerequisites[i].length == 2
- 0 <= ai, bi < numCourses
- prerequisites[i] 中的所有课程对互不相同

### 解题思路

利用哈希表记录入度表，利用哈希表记录入度的出度，利用 set 来遍历。

### Go 实现

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    n := numCourses
    pre := prerequisites
    in := make([]int, n)
    deps := make([][]int, n)
    learned := make([]int, 0, n)
    for _, v := range pre {
        in[v[0]]++ //有入度
        deps[v[1]] = append(deps[v[1]], v[0]) //出度的入度 0=>1
    }
    for i := 0; i < n; i++ {
        if in[i] == 0 { //无入度
            learned = append(learned, i)
        }
    }
    for i := 0; i != len(learned); i++ {
        c := learned[i] //入度0
        v := deps[c] //有出度
        for _, vv := range v { //遍历入度
            in[vv]--
            if in[vv] == 0 {
                learned = append(learned, vv)
            }
        }
    }
    return len(learned) == n
}
```

[src: raw/ingested/2技术/go/go_leetcode技巧-[课程表](https---leetcode-cn.com-problems-course-schedule).md]

---

## 跳表（Skip List）

详见 [[跳表]]。

[src: raw/ingested/2技术/go/go_leetcode技巧-[设计跳表](https---leetcode.cn-problems-design-skiplist-).md]