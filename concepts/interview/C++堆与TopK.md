# C++ 堆与 TopK

> 本文涵盖堆（Heap）与 TopK 问题的经典题型与 C++ 实现，包括第 K 大、前 K 个高频单词、最后一块石头的重量、任务调度器、至少有 K 个重复字符的最长子串。

See also: [[C++算法精选合并版]], [[C++手写代码模板]], [[C++高频面试问题]]

[src: raw/ingested/2技术/算法/C++算法精选合并版-十三、堆与-TopK.md]

## 第 K 大

未排序数组中找第 k 大的元素。

**力扣**：[kth-largest-element-in-an-array](https://leetcode-cn.com/problems/kth-largest-element-in-an-array)

```cpp
// 第 K 大：维护大小为 k 的小根堆，堆顶即第 k 大
int findKthLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> pq;
    for (int x : nums) {
        pq.push(x);
        if (pq.size() > k) pq.pop();
    }
    return pq.top();
}
```

## 前 K 个高频单词

单词数组中，返回出现频次最高的 k 个单词，同频按字典序。

**力扣**：[top-k-frequent-words](https://leetcode-cn.com/problems/top-k-frequent-words)

```cpp
// 小顶堆维护 K 个，频次高优先，同频字典序小优先
vector<string> topKFrequent(vector<string>& words, int k) {
    unordered_map<string, int> cnt;
    for (auto& w : words) cnt[w]++;
    auto cmp = [](const pair<string,int>& a, const pair<string,int>& b) {
        if (a.second != b.second) return a.second > b.second;  // 频次小先被 pop
        return a.first > b.first;  // 同频字典序大先被 pop，保留小字典序
    };
    priority_queue<pair<string,int>, vector<pair<string,int>>, decltype(cmp)> pq(cmp);
    for (auto& [w, c] : cnt) {
        pq.push({w, c});
        if (pq.size() > k) pq.pop();
    }
    vector<string> res;
    while (!pq.empty()) { res.push_back(pq.top().first); pq.pop(); }
    reverse(res.begin(), res.end());
    return res;
}
```

## 最后一块石头的重量

每次选最重两块粉碎，y-x 放回，求最后剩余重量。

**力扣**：[last-stone-weight](https://leetcode.cn/problems/last-stone-weight)

```cpp
int lastStoneWeight(vector<int>& stones) {
    priority_queue<int> pq(stones.begin(), stones.end());
    while (pq.size() > 1) {
        int x = pq.top(); pq.pop();
        int y = pq.top(); pq.pop();
        if (x > y) pq.push(x - y);
    }
    return pq.empty() ? 0 : pq.top();
}
```

## 任务调度器

相同任务间隔至少 n，求完成所有任务的最短时间。

**力扣**：[task-scheduler](https://leetcode-cn.com/problems/task-scheduler)

```cpp
int leastInterval(vector<char>& tasks, int n) {
    if (n == 0) return tasks.size();
    vector<int> cnt(26);
    int maxCnt = 0;
    for (char c : tasks) { cnt[c-'A']++; maxCnt = max(maxCnt, cnt[c-'A']); }
    int res = (maxCnt - 1) * (n + 1);
    for (int c : cnt) if (c == maxCnt) res++;
    return max(res, (int)tasks.size());
}
```

## 至少有 K 个重复字符的最长子串

子串中每个字符出现次数都不少于 k，求最长长度。

**力扣**：[longest-substring-with-at-least-k-repeating-characters](https://leetcode-cn.com/problems/longest-substring-with-at-least-k-repeating-characters)

```cpp
int longestSubstring(string s, int k) {
    int res = 0;
    for (int i = 0; i + k <= (int)s.size(); ) {
        int m[26] = {}, mask = 0, cur = i;
        for (int j = i; j < s.size(); j++) {
            int t = s[j] - 'a';
            m[t]++;
            mask = (m[t] < k) ? (mask | (1 << t)) : (mask & ~(1 << t));
            if (mask == 0) { res = max(res, j - i + 1); cur = j; }
        }
        i = cur + 1;
    }
    return res;
}
```
