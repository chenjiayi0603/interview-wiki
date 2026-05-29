# C++ 图论

## 10.0 可增删节点的图（邻接表 + 智能指针）

**说明**：节点用 `Node` 表示，`shared_ptr` 管理内存，支持动态增删节点和边。

| 函数 | 时间复杂度 |
|------|------------|
| `addNode(id)` | O(1) 均摊 |
| `addEdge(u, v)` | O(1) 均摊 |
| `removeNode(id)` | O(V) |

```cpp
struct Node {
    int id;                                    // 节点标识
    unordered_set<shared_ptr<Node>> successors; // 后继集合（去重、erase O(1)）
    Node(int i) : id(i) {}
};

class DynamicGraph {
    unordered_map<int, shared_ptr<Node>> nodes;  // id -> Node

public:
    shared_ptr<Node> addNode(int id) {          // O(1)
        if (nodes.count(id)) return nodes[id];
        auto p = make_shared<Node>(id);
        nodes[id] = p;
        return p;
    }

    void addEdge(shared_ptr<Node> u, shared_ptr<Node> v) {  // O(1)
        if (!u || !v) return;
        u->successors.insert(v);
    }
    void removeNode(int id) {                   // O(V)
        if (!nodes.count(id)) return;
        auto u = nodes[id];
        for (auto& [_, p] : nodes) p->successors.erase(u);  // O(1) 均摊
        nodes.erase(id);
    }
};
```

**removeNode 进一步优化**：维护 predecessor 表可降至 O(in_degree(u))，仅遍历指向 u 的前驱。

## 10.1 拓扑排序

**说明**：课程表依赖关系，BFS 拓扑排序判环。

#### 课程表

n 门课，prerequisites 表示先修关系，判断能否完成所有课程（无环）。

**力扣**：[course-schedule](https://leetcode-cn.com/problems/course-schedule)

```cpp
// 课程表：建图+入度，BFS 拓扑排序，能处理完所有节点则无环
bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
    // 使用邻接表建图：graph[i] 存放课程 i 的后继课程
    vector<vector<int>> graph(numCourses);
    // 统计每门课程的入度（被多少门课依赖）
    vector<int> indegree(numCourses, 0);
    // 根据先修关系，填充图结构和入度数组
    for (auto& e : prerequisites) {
        // e[1] -> e[0]，即学完 e[1] 才能学 e[0]，建立有向边
        graph[e[1]].push_back(e[0]);
        indegree[e[0]]++;
    }
    // 将所有入度为 0 的课程（无需先修课）入队
    queue<int> q;
    for (int i = 0; i < numCourses; i++)
        if (indegree[i] == 0) q.push(i);
    int count = 0; // 记录可学完的课程数量
    // 拓扑排序过程：每次弹出入度为 0 的课程，并将其后继课程入度减一
    while (!q.empty()) {
        int u = q.front(); q.pop();
        count++;
        for (int v : graph[u])
            if (--indegree[v] == 0) q.push(v); // 新变为入度 0 的课程入队
    }
    // 能弹出所有课程说明无环，返回 true，否则有环返 false
    return count == numCourses;
}
```

#### 课程表 II

返回一种完成所有课程的顺序，不可能则返回空数组。

**力扣**：[course-schedule-ii](https://leetcode-cn.com/problems/course-schedule-ii)

```cpp
vector<int> findOrder(int numCourses, vector<vector<int>>& prerequisites) {
    vector<vector<int>> graph(numCourses);
    vector<int> indegree(numCourses, 0);
    for (auto& e : prerequisites) {
        graph[e[1]].push_back(e[0]);
        indegree[e[0]]++;
    }
    queue<int> q;
    for (int i = 0; i < numCourses; i++) if (indegree[i] == 0) q.push(i);
    vector<int> res;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        res.push_back(u);
        for (int v : graph[u]) if (--indegree[v] == 0) q.push(v);
    }
    return res.size() == numCourses ? res : vector<int>();
}
```

## 10.2 并查集

**说明**：路径压缩，find/unite 维护连通分量。

#### 并查集

维护 n 个元素的集合，支持合并两集合、查询两元素是否同集合。

```cpp
// 并查集（带路径压缩）：支持查找和合并两元素所属集合
class UnionFind {
    vector<int> parent;
public:
    UnionFind(int n) : parent(n) {
        // 令每个节点的父节点初始为自己（即 parent[i] = i）
        iota(parent.begin(), parent.end(), 0); 
    }
    int find(int x) { // 路径压缩
        // 意思是并查集的路径压缩查找根节点
        // 路径压缩不会影响其他节点的查找。它只是让当前遍历路径上的节点直接指向根节点，让树更扁平，实际仅优化了这些节点后续查询的效率，并不会影响其它未经过此次 find 的节点后续的查找及结果。所有合并与查询操作的正确性都不会受影响。
        while (x != parent[x]) {
            parent[x] = parent[parent[x]]; // 路径压缩：x 直接跳到爷爷
            x = parent[x];                 // 继续向根节点查找
        }
        return x;
    }
    bool unite(int x, int y) { // 合并两个集合，若已同集合返回 false
        // 意思是：找出 x 和 y 所属集合的根（即祖先），如果已在同一个集合则返回 false，否则将 x 的根父指向 y 的根，实现合并两个集合。
        int px = find(x), py = find(y); // 查找 x, y 的根
        if (px == py) return false;     // 已在同一集合，无法合并
        parent[px] = py;                // 合并集合：让 x 的根挂到 y 的根下
        return true;
    }
};
// 例：并查集用法举例
/*
示例：判断一组无向边是否联通
int n = 5;
UnionFind uf(n);
vector<pair<int,int>> edges = {{0,1},{1,2},{3,4}};
// 合并所有无向边对应的节点
for (const auto& edge : edges) {
    uf.unite(edge.first, edge.second);
}
// 查询 0,2 是否联通
bool connected = uf.find(0) == uf.find(2); // true
// 查询 0,4 是否联通
bool connected2 = uf.find(0) == uf.find(4); // false
*/

```

## 10.3 最短路径

**说明**：Dijkstra 小根堆 + 松弛，求单源最短路。

#### Dijkstra

带权有向图，求从源点 s 到终点 t 的最短路径长度。

```cpp
// Dijkstra：小根堆存 (距离, 点)，松弛时更新 dist 并入堆
long long dijkstra(int n, vector<vector<pair<int,int>>>& g, int s, int t) {
    vector<long long> dist(n, LLONG_MAX);
    dist[s] = 0;
    priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;
    pq.push({0, s});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;
        for (auto [v, w] : g[u])
            if (dist[v] > dist[u] + w) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
    }
    return dist[t];
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十一、图论.md]