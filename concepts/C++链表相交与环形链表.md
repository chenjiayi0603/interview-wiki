# C++ 链表相交与环形链表

## 相交链表

### [相交链表](https://leetcode-cn.com/problems/intersection-of-two-linked-lists)

编写一个程序，找到两个单链表相交的起始节点。

注意：
- 如果两个链表没有交点，返回 null.
- 在返回结果后，两个链表仍须保持原有的结构。
- 可假定整个链表结构中没有循环。
- 程序尽量满足 O(n) 时间复杂度，且仅用 O(1) 内存。

**两个链表分别遍历两次**

```cpp
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        if (!headA || !headB) return nullptr;
        ListNode* p1 = headA;
        ListNode* p2 = headB;
        while (p1 != p2) {
            p1 = (!p1) ? headB : p1->next;
            p2 = (!p2) ? headA : p2->next;
        }
        return p1;
    }
};
```

> [!contradiction]
> 新源提供了另一种解法：使用哈希表（unordered_set）遍历一次链表，但需要 O(N) 空间复杂度，不符合题目要求的 O(1) 内存。该解法在源中被标注为不符合要求。

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-两链表相交.md]

## 环形链表

### [环形链表](https://leetcode-cn.com/problems/linked-list-cycle)

给定一个链表，判断链表中是否有环。

使用快慢指针的方式遍历判断。

```cpp
class Solution {
public:
    bool hasCycle(ListNode *head) {
        if (!head || !head->next) return false;
        ListNode* fp = head, * sp = head;
        while (fp && fp->next) {
            sp = sp->next;
            fp = fp->next->next;
            if (fp == sp) return true;
        }
        return false;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-两链表相交.md]

### [环形链表 II](https://leetcode-cn.com/problems/linked-list-cycle-ii)

给定一个链表，返回链表开始入环的第一个节点。如果链表无环，则返回 null。

使用快慢指针找到重叠，重置快指针，再找到重叠点就是入环点。

```cpp
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        if (!head || !head->next) return nullptr;
        ListNode* fp = head, * sp = head;
        while (fp && fp->next) {
            sp = sp->next;
            fp = fp->next->next;
            if (fp == sp) break;
        }
        if (fp != sp) return nullptr;
        fp = head;
        while (fp != sp) {
            sp = sp->next;
            fp = fp->next;
        }
        return sp;
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-两链表相交.md]