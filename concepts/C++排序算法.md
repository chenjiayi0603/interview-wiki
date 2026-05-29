# C++ 排序算法

## 一、快排

**题目**：给定整数数组，原地升序排序。要求 O(nlogn) 平均时间复杂度。

**力扣**：[sort-an-array](https://leetcode-cn.com/problems/sort-an-array) — 数组排序，选最右为基准分区，递归左右

```cpp
void qs(vector<int>& v, int l, int r) {
    if (l >= r) return;
    int pi = v[r], k = l;  // 选最右为基准
    for (int i = l; i <= r; i++)
        if (v[i] < pi) swap(v[k++], v[i]);  // 小于基准的放左侧
    swap(v[k], v[r]);  // 基准归位到分界点
    qs(v, l, k - 1);   // 递归左半
    qs(v, k + 1, r);   // 递归右半
}
```

#### 栈模拟递归（DFS）

非递归快排用栈模拟 LIFO，严格模拟递归顺序。

```cpp
void qs_stack(vector<int>& v, int l, int r) {
    if (l >= r) return;
    stack<pair<int,int>> s;
    s.push({l, r});
    while (!s.empty()) {
        auto [L, R] = s.top(); s.pop();
        if (L >= R) continue;
        int pi = v[R], k = L;
        for (int i = L; i <= R; i++)
            if (v[i] < pi) swap(v[k++], v[i]);
        swap(v[k], v[R]);   // 分区完成
        s.push({k + 1, R}); // 先压右区间，模拟 DFS 先处理左
        s.push({L, k - 1});
    }
}
```

## 二、堆排

**题目**：给定整数数组，原地升序排序，要求 O(nlogn)。

**说明**：大顶堆下沉建堆，每次取堆顶与末尾交换，堆化剩余部分，O(nlogn) 原地排序。

```cpp
// 大顶堆下沉：以 l 为根的子树，将 nums[l] 下沉到合适位置
void adjust(vector<int>& nums, int l, int r) {
    int pi = nums[l], k = l;
    for (int i = 2*l+1; i <= r; i = 2*i+1) {
        if (i < r && nums[i] < nums[i+1]) i++;  // 取较大子节点
        if (pi >= nums[i]) break;
        nums[k] = nums[i]; k = i;  // 子节点上浮
    }
    nums[k] = pi;  // 原根节点落位
}
void heapSort(vector<int>& nums) {
    for (int i = nums.size()/2; i >= 0; i--) adjust(nums, i, nums.size()-1);  // 建堆
    for (int i = nums.size()-1; i > 0; i--) {
        swap(nums[0], nums[i]);  // 堆顶与末尾交换
        adjust(nums, 0, i-1);   // 重新堆化
    }
}
```

## 三、归并排序

**题目**：给定整数数组，升序排序，可非原地实现。

**说明**：分治，左右分别有序后合并，稳定排序，O(nlogn)。

```cpp
void mergeSort(vector<int>& nums, int l, int r) {
    if (l >= r) return;
    int m = l + (r - l) / 2;
    mergeSort(nums, l, m);      // 左半有序
    mergeSort(nums, m + 1, r);   // 右半有序
    vector<int> tmp(r - l + 1);
    int i = l, j = m + 1, k = 0;
    while (i <= m && j <= r) tmp[k++] = nums[i] <= nums[j] ? nums[i++] : nums[j++];
    while (i <= m) tmp[k++] = nums[i++];
    while (j <= r) tmp[k++] = nums[j++];
    for (int i = 0; i < k; i++) nums[l + i] = tmp[i];  // 回写
}
```

## 四、STL 排序技巧

**题目**：对自定义结构体数组按指定规则排序，需自定义比较器。

**说明**：自定义比较的三种方式：重载 `operator<`、仿函数、函数指针。

```cpp
// 类重载 operator<
struct Student { int id; bool operator<(const Student& s) { return id < s.id; } };
sort(V.begin(), V.end());

// 仿函数
struct Less { bool operator()(const Student& a, const Student& b) { return a.id < b.id; } };
sort(V.begin(), V.end(), Less());

// 函数指针
bool cmp(int a, int b) { return a < b; }
sort(A, A + n, cmp);
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-一、排序.md]