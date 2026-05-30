# C++ 链表

> 链表（Linked List）是一种常见的线性数据结构，由一系列节点组成，每个节点包含数据和指向下一个节点的指针。本文涵盖链表的基础操作和进阶算法，包括反转、两数相加、合并有序链表、删除节点、相交链表、合并 K 个升序链表、排序链表、旋转链表等。

See also: [[C++双指针]], [[C++DFS回溯]], [[C++手写代码模板]]

## 4.1 基础操作

**说明**：反转、两数相加、合并有序、删倒数第 N、相交链表等基础操作。

### 反转链表

将链表所有节点指针反向，返回新头节点。

**力扣**：[reverse-linked-list](https://leetcode-cn.com/problems/reverse-linked-list)

```cpp
// 反转链表：三指针 pre/cur/nxt，逐个反转 next
ListNode* reverseList(ListNode* head) {
    ListNode* pre = nullptr;
    while (head) {
        ListNode* nxt = head->next;
        head->next = pre;
        pre = head;
        head = nxt;
    }
    return pre;
}
```

### 两数相加

两条链表表示非负整数（逆序存），求其和的链表表示。

**力扣**：[add-two-numbers](https://leetcode-cn.com/problems/add-two-numbers)

```cpp
// 两数相加：逐位相加，carry 进位，dummy 简化头处理
ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
    ListNode dummy(0);
    ListNode* cur = &dummy;
    int carry = 0;
    while (l1 || l2 || carry) {
        int s = carry + (l1 ? l1->val : 0) + (l2 ? l2->val : 0);
        cur->next = new ListNode(s % 10);
        carry = s / 10;
        if (l1) l1 = l1->next;
        if (l2) l2 = l2->next;
        cur = cur->next;
    }
    return dummy.next;
}
```

### 合并两个有序链表

将两条升序链表合并为一条升序链表。

**力扣**：[merge-two-sorted-lists](https://leetcode-cn.com/problems/merge-two-sorted-lists)

```cpp
// 合并两个有序链表：双指针归并，dummy 头
ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
    if (!l1 || !l2) return l1 ? l1 : l2;
    ListNode dummy(0);
    ListNode* p = &dummy;
    while (l1 && l2) {
        if (l1->val <= l2->val) { p->next = l1; l1 = l1->next; }
        else { p->next = l2; l2 = l2->next; }
        p = p->next;
    }
    p->next = l1 ? l1 : l2;
    return dummy.next;
}
```

### 删除链表中的节点

只给待删节点，无头指针，将后继值复制到当前节点，删除后继。

**力扣**：[delete-node-in-a-linked-list](https://leetcode-cn.com/problems/delete-node-in-a-linked-list)

```cpp
// 删除链表中的节点
void deleteNode(ListNode* node) {
    ListNode* tmp = node->next;
    node->val = tmp->val;
    node->next = tmp->next;
    delete tmp;
}
```

### 删除倒数第 N 个节点

删除链表倒数第 n 个节点，返回头节点。

**力扣**：[remove-nth-node-from-end-of-list](https://leetcode-cn.com/problems/remove-nth-node-from-end-of-list)

```cpp
// 删除倒数第N个节点：fast 先走 n 步，再同步走，slow 停在待删前驱
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0); dummy.next = head;
    ListNode *fast = &dummy, *slow = &dummy;
    for (int i = 0; i < n; i++) fast = fast->next;
    while (fast->next) { fast = fast->next; slow = slow->next; }
    slow->next = slow->next->next;
    return dummy.next;
}
```

### 相交链表

两条链表可能在某节点后合并，找出第一个公共节点。

**力扣**：[intersection-of-two-linked-lists](https://leetcode-cn.com/problems/intersection-of-two-linked-lists)

```cpp
// 相交链表：长链先走差值步，再同步走，相遇即交点
ListNode* getIntersectionNode(ListNode* headA, ListNode* headB) {
    int la = 0, lb = 0;
    for (auto p = headA; p; p = p->next) la++;
    for (auto p = headB; p; p = p->next) lb++;
    auto a = headA, b = headB;
    if (la > lb) for (int i = 0; i < la - lb; i++) a = a->next;
    else for (int i = 0; i < lb - la; i++) b = b->next;
    while (a != b) { a = a->next; b = b->next; }
    return a;
}
```

> [!contradiction]
> 新源提供了另一种解法：使用哈希表（unordered_map）遍历一次链表，但需要 O(N) 空间复杂度，不符合题目要求的 O(1) 内存。该解法在源中被标注为不符合要求。

[src: raw/ingested/2技术/算法/cpp_leetcode_腾讯精选练习50题-[相交链表](https---leetcode-cn.com-problems-intersection-of-two-.md]

## 4.2 进阶

**说明**：合并 K 条链表用堆；旋转链表成环后断环。

### 合并 K 个升序链表

将 K 条升序链表合并为一条升序链表。

**力扣**：[merge-k-sorted-lists](https://leetcode-cn.com/problems/merge-k-sorted-lists)

**方法一：小根堆** O(KN·logK)

```cpp
// 合并K个升序链表：小根堆存各链头，每次取最小接上
ListNode* mergeKLists(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    for (auto h : lists) if (h) pq.push(h);
    ListNode dummy(0);
    ListNode* p = &dummy;
    while (!pq.empty()) {
        auto node = pq.top(); pq.pop();
        p->next = node; p = p->next;
        if (node->next) pq.push(node->next);
    }
    return dummy.next;
}
```

**方法二：分治两两合并** O(KN·logK)，空间 O(1)

```cpp
// 分治：两两合并，类似归并排序
ListNode* mergeKLists(vector<ListNode*>& lists) {
    if (lists.empty()) return nullptr;
    list<ListNode*> l(lists.begin(), lists.end());
    while (l.size() > 1) {
        auto l1 = l.front(); l.pop_front();
        auto l2 = l.front(); l.pop_front();
        l.push_back(mergeTwoLists(l1, l2));
    }
    return l.front();
}
```

### 排序链表

O(n log n) 时间、O(1) 空间对链表排序，用自底向上归并。

**力扣**：[sort-list](https://leetcode-cn.com/problems/sort-list)

```cpp
// 排序链表：自底向上归并，cut 按长度分割，merge 合并
ListNode* sortList(ListNode* head) {
    if (!head) return head;
    ListNode dummy(0); dummy.next = head;
    int len = 0;
    for (auto p = head; p; p = p->next) len++;
    for (int size = 1; size < len; size <<= 1) {
        auto cur = dummy.next, tail = &dummy;
        while (cur) {
            auto left = cur, right = cut(left, size);
            cur = right ? cut(right, size) : nullptr;
            tail->next = merge(left, right);
            while (tail->next) tail = tail->next;
        }
    }
    return dummy.next;
}
ListNode* cut(ListNode* head, int n) {
    if (!head) return nullptr;
    while (--n && head->next) head = head->next;
    auto next = head->next;
    head->next = nullptr;
    return next;
}
ListNode* merge(ListNode* l1, ListNode* l2) {
    ListNode dummy(0);
    auto p = &dummy;
    while (l1 && l2) {
        if (l1->val <= l2->val) { p->next = l1; l1 = l1->next; }
        else { p->next = l2; l2 = l2->next; }
        p = p->next;
    }
    p->next = l1 ? l1 : l2;
    return dummy.next;
}
```

### 旋转链表

将链表向右旋转 k 个位置（尾部 k 个节点移到头部）。

**力扣**：[rotate-list](https://leetcode-cn.com/problems/rotate-list)

```cpp
// 旋转链表：成环，找新头位置 len-k，断环
ListNode* rotateRight(ListNode* head, int k) {
    if (!head || !head->next) return head;
    int len = 1;
    ListNode* tail = head;
    while (tail->next) { tail = tail->next; len++; }
    k %= len;
    if (k == 0) return head;
    ListNode* p = head;
    for (int i = 0; i < len - k - 1; i++) p = p->next;
    tail->next = head;
    head = p->next;
    p->next = nullptr;
    return head;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-四、链表.md]

## 合并K个排序链表（腾讯精选练习50题）

合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

**力扣**：[merge-k-sorted-lists](https://leetcode-cn.com/problems/merge-k-sorted-lists)

示例:
输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6

考虑分治的思想来解这个题（类似归并排序的思路）。把这些链表分成两半，如果每一半都合并好了，那么我就最后把这两个合并了就行了。这就是分治法的核心思想。
但是这道题由于存的都是指针，就具有了更大的操作灵活性，可以不用递归来实现分治。就是先两两合并后在两两合并。。。一直下去直到最后成了一个。（相当于分治算法的那棵二叉树从底向上走了）。
第一次两两合并是进行了k/2次，每次处理2n个值,即2n * k/2 = kn 次比较。
第二次两两合并是进行了k/4次，每次处理4n个值,即4n * k/4 = kn 次比较。
。。。
最后一次两两合并是进行了k/(2^logk)次（=1次），每次处理2^logK  * N个值（kn个），即1*kn= kn 次比较。
所以时间复杂度：
O(KN* logK)
空间复杂度是O(1)。

```cpp
class Solution {
public:
    shared_ptr<ListNode> dummy = make_shared<ListNode>();
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        if(lists.empty()){
            return nullptr;
        }
        list<ListNode*> l;
        for(auto h:lists)l.emplace_back(h);
        while(l.size() > 1){
            ListNode* l1 = l.front();
            l.pop_front();
            ListNode* l2 = l.front();
            l.pop_front();
            l.emplace_back(mergeTwoLists(l1,l2));
        }
        return l.front();
    }
    ListNode *mergeTwoLists(ListNode *l1, ListNode *l2) {
        if(!l1 || !l2) return l1? l1:l2;
        ListNode *tmp = dummy.get();
        while (l1 && l2)
        {
            if(l1->val <= l2->val)
            {
                tmp ->next = l1;
                l1 = l1->next;
            }
            else
            {
                tmp->next = l2;
                l2 = l2->next;
            }
            tmp = tmp->next;
        }
        if (l1) tmp->next = l1;
        else tmp->next = l2;
        return dummy->next;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode_腾讯精选练习50题-[合并K个排序链表](https---leetcode-cn.com-problems-merge-k-sorted-l.md]

## 排序链表（腾讯精选练习50题）

在 O(n log n) 时间复杂度和常数级空间复杂度下，对链表进行排序。

**力扣**：[sort-list](https://leetcode-cn.com/problems/sort-list)

示例 1:
输入: 4->2->1->3
输出: 1->2->3->4

示例 2:
输入: -1->5->3->4->0
输出: -1->0->3->4->5

### 冒泡排序方案（不满足 O(n log n) 要求）

复杂度 O(N*N)，冒泡的时间复杂度是不能满足题目要求的，只是比较简单。

通过 1152 ms 15.3 MB

```cpp
class Solution {
public:
    ListNode* sortList(ListNode* head) {
        if (!head)return head;
        ListNode* cur = head;
        for(;cur;cur = cur->next)      
        {
            ListNode* mark = cur,* tmp = cur;
            while(tmp)
            {
                if (mark->val > tmp->val)
                {
                    mark = tmp;
                }
                tmp = tmp->next;
            }
            swap(cur->val,mark->val);
        }
        return head;
    }
};
```

### 二路归并排序（自底向上）

通过 68 ms  15.5 MB Cpp
从运行结果来看，归并排序要好很多。

```cpp
class Solution {
public:
    ListNode* sortList(ListNode* head) {
        if (!head) return head;
        ListNode dummyHead(0);
        dummyHead.next = head;
        auto p = head;
        int length = 0;
        while (p) {++length;p = p->next;}
        for (int size = 1; size < length; size <<= 1) {
            auto cur = dummyHead.next;
            auto tail = &dummyHead;
            while (cur) {
                auto left = cur;
                auto right = cut(left, size);
                if (right)
                {
                    cur = cut(right, size);
                    tail->next = merge(left, right);
                    while (tail->next) tail = tail->next;
                }
                else
                {
                    tail->next = left;
                    break;
                }
            }
        }
        return dummyHead.next;
    }
    ListNode* cut(ListNode* head, int n) {
        if (!head) return nullptr;
        while (--n && head->next) head = head->next;
        auto next = head->next;
        head->next = nullptr;
        return next;
    }
    ListNode* merge(ListNode* l1, ListNode* l2) {
        ListNode dummyHead(0);
        auto p = &dummyHead;
        while (l1 && l2) {
            if (l1->val < l2->val) {
                p->next = l1;
                p = l1;
                l1 = l1->next;       
            } else {
                p->next = l2;
                p = l2;
                l2 = l2->next;
            }
        }
        p->next = l1 ? l1 : l2;
        return dummyHead.next;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode_腾讯精选练习50题-[排序链表](https---leetcode-cn.com-problems-sort-list).md]