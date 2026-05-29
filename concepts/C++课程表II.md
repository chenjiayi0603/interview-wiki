# C++ 课程表 II

## 问题描述
现在你总共有 numCourses 门课需要选，记为 0 到 numCourses - 1。给你一个数组 prerequisites ，其中 prerequisites[i] = [ai, bi] ，表示在选修课程 ai 前 必须 先选修 bi 。

例如，想要学习课程 0 ，你需要先完成课程 1 ，我们用一个匹配来表示：[0,1] 。
返回你为了学完所有课程所安排的学习顺序。可能会有多个正确的顺序，你只要返回 任意一种 就可以了。如果不可能完成所有课程，返回 一个空数组 。

## 示例

示例 1：
输入：numCourses = 2, prerequisites = [[1,0]]
输出：[0,1]
解释：总共有 2 门课程。要学习课程 1，你需要先完成课程 0。因此，正确的课程顺序为 [0,1] 。

示例 2：
输入：numCourses = 4, prerequisites = [[1,0],[2,0],[3,1],[3,2]]
输出：[0,2,1,3]
解释：总共有 4 门课程。要学习课程 3，你应该先完成课程 1 和课程 2。并且课程 1 和课程 2 都应该排在课程 0 之后。
因此，一个正确的课程顺序是 [0,1,2,3] 。另一个正确的排序是 [0,2,1,3] 。

示例 3：
输入：numCourses = 1, prerequisites = []
输出：[0]

## 提示
- 1 <= numCourses <= 2000
- 0 <= prerequisites.length <= numCourses * (numCourses - 1)
- prerequisites[i].length == 2
- 0 <= ai, bi < numCourses
- ai != bi
- 所有[ai, bi] 互不相同

## 解法思路
因为只需要返回一种，可以直接遍历。使用入度列表和出度列表，每次选择入度为0的课程学习，并减少其依赖课程的入度。

## C++ 实现

```cpp
class Solution {
public:
    vector<int> findOrder(int numCourses, vector<vector<int>>& prerequisites) {
        unordered_map<int,int> inM;//入度列表
        set<int> courses;//课程
        unordered_map<int,vector<int>> depM;//出度的所有入度列表（出度是被依赖）
        for(auto &iter:prerequisites)
        {
            inM[iter[0]]++;//入度计数，0依赖1(1=>0)
            depM[iter[1]].push_back(iter[0]);//出度的所有入度，1（出度）=>0（入度）;
            courses.insert(iter[0]);//有依赖的课程
            courses.insert(iter[1]);
        }
        vector<int> res;
        for(int i = 0;i < numCourses;++i)
        {
            if (courses.count(i) == 0)res.push_back(i);//没有依赖的则先学习
        }
        while(courses.size() > 0)
        {
            int left = courses.size();
            for(auto iter = courses.begin();iter != courses.end();)
            {
                if (inM[*iter] == 0)//没有入度的
                {
                    res.push_back(*iter);//学习没有入度的
                    for(auto i:depM[*iter])//减少该出度指向的所有入度（减少依赖）
                    {
                        inM[i]--; 
                    }
                    courses.erase(iter++);
                }
                else 
                {
                    iter++;
                }
            }
            if (courses.size() == left)return {};//不能完成
        }
        return res;
    }
};
```

## 相关数据结构
- 队列

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-[课程表-II]-(https---leetcode-cn.com-problems-course-schedule-i.md]