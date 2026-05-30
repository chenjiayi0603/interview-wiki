# C++ 归并排序（两两合并）

## 合并K个排序链表

**力扣**：[merge-k-sorted-lists](https://leetcode-cn.com/problems/merge-k-sorted-lists)

给你一个链表数组，每个链表都已经按升序排列。请你将所有链表合并到一个升序链表中，返回合并后的链表。

### 示例

示例 1：
输入：lists = [[1,4,5],[1,3,4],[2,6]]
输出：[1,1,2,3,4,4,5,6]

示例 2：
输入：lists = []
输出：[]

示例 3：
输入：lists = [[]]
输出：[]

### 提示

- k == lists.length
- 0 <= k <= 10^4
- 0 <= lists[i].length <= 500
- -10^4 <= lists[i][j] <= 10^4
- lists[i] 按 升序 排列
- lists[i].length 的总和不超过 10^4

### 思路

考虑分治的思想来解这个题（类似归并排序的思路）。把这些链表分成两半，如果每一半都合并好了，那么我就最后把这两个合并了就行了。这就是分治法的核心思想。
但是这道题由于存的都是指针，就具有了更大的操作灵活性，可以不用递归来实现分治。就是先两两合并后在两两合并。。。一直下去直到最后成了一个。（相当于分治算法的那棵二叉树从底向上走了）。

第一次两两合并是进行了k/2次，每次处理2n个值,即2n * k/2 = kn 次比较。
第二次两两合并是进行了k/4次，每次处理4n个值,即4n * k/4 = kn 次比较。
。。。
最后一次两两合并是进行了k/(2^logk)次（=1次），每次处理2^logK  * N个值（kn个），即1*kn= kn 次比较。
所以时间复杂度：O(KN*logK)，空间复杂度 O(1)。

### 代码实现

```cpp
class Solution {
public:
    ListNode dummy;
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        if (lists.empty()) return nullptr;
        list<ListNode*> l;
        for (auto h : lists) l.emplace_back(h);
        while (l.size() > 1) {
            ListNode* l1 = l.front(); l.pop_front();
            ListNode* l2 = l.front(); l.pop_front();
            l.emplace_back(mergeTwoLists(l1, l2));
        }
        return l.front();
    }
    ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
        if (!l1 || !l2) return l1 ? l1 : l2;
        ListNode* pre = &dummy;
        while (l1 && l2) {
            if (l1->val <= l2->val) { pre->next = l1; l1 = l1->next; }
            else { pre->next = l2; l2 = l2->next; }
            pre = pre->next;
        }
        pre->next = l1 ? l1 : l2;
        return dummy.next;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-归并排序（两两合并）.md]