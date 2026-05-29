# C++ 双遍历一维数组

使用双遍历的，遍历一次i，再遍历一次j（j<i）， 根据局部条件，计算局部解，再记录全局变量。

## 最长上升子序列

给你一个整数数组 nums ，找到其中最长严格递增子序列的长度。

子序列 是由数组派生而来的序列，删除（或不删除）数组中的元素而不改变其余元素的顺序。例如，[3,6,2,7] 是数组 [0,3,1,6,2,2,7] 的子序列。

**力扣**：[longest-increasing-subsequence](https://leetcode-cn.com/problems/longest-increasing-subsequence)

示例 1：
输入：nums = [10,9,2,5,3,7,101,18]
输出：4
解释：最长递增子序列是 [2,3,7,101]，因此长度为 4 。

示例 2：
输入：nums = [0,1,0,3,2,3]
输出：4

示例 3：
输入：nums = [7,7,7,7,7,7,7]
输出：1

提示：
- 1 <= nums.length <= 2500
- -104 <= nums[i] <= 104

进阶：你能将算法的时间复杂度降低到 O(n log(n)) 吗?

不同于连续的，所以双遍历。

转移方程：
if (nums[i] > nums[j]) dp[i] = max(dp[j]+1, dp[i]); //i = [1,n] ,j = [0,i)

### O(n²) 解法

```cpp
class Solution {
public:
    int lengthOfLIS(vector<int>& nums)
    {
        if(nums.size()==0)return 0;
        vector<int> dp(nums.size(),1);//dp[0] = 1
        int g = 1;
        for (int i=1; i<nums.size(); ++i) //i = [1,n] ,j = [0,i)
        {
            for (int j = 0; j < i; ++j)
            {
                //nums[i]必须用才记录到dp[i]
                if (nums[i] > nums[j]) dp[i] = max(dp[j]+1, dp[i]);
            }
            g = max(g, dp[i]);
        }
        return g;
    }
};
```

### O(n log n) 优化解法

```cpp
class Solution {
public:
    int lengthOfLIS(vector<int>& nums)
    {
        if(nums.size() == 0) return 0;
        vector<int> dp(nums.size());
        dp[0]= nums[0];
        int len = 0;
        for(int i = 1;i < nums.size();i++)
        {
            if(nums[i] > dp[len])//len的数组是有序的
            {
                dp[++len] = nums[i];//放后面
            }
            else if (nums[i] == dp[len])continue;
            else//nums[i] < dp[len]
            {
                //log(n),二分查找第一个小于或等于num的数字，找到返回该数字的地址
                //用更小的nums[i]替换位置（影响的只可能是后面的数据）,这样不影响总体长度
                auto j = lower_bound(dp.begin(),dp.begin() + len + 1,nums[i]);
                *j = nums[i];
            }
        }
        return len + 1;
    }
};
```

## 完全平方数

给定正整数 n，找到若干个完全平方数（比如 1, 4, 9, 16, ...）使得它们的和等于 n。你需要让组成和的完全平方数的个数最少。

给你一个整数 n ，返回和为 n 的完全平方数的 最少数量 。

完全平方数 是一个整数，其值等于另一个整数的平方；换句话说，其值等于一个整数自乘的积。例如，1、4、9 和 16 都是完全平方数，而 3 和 11 不是。

**力扣**：[perfect-squares](https://leetcode-cn.com/problems/perfect-squares)

示例 1：
输入：n = 12
输出：3
解释：12 = 4 + 4 + 4

示例 2：
输入：n = 13
输出：2
解释：13 = 4 + 9

递推公式：dp[i] = min(dp[i - k * k] + 1,dp[i])  (i 在[1,n], k*k <= i,k >= 1)

```cpp
class Solution {
public:
    int numSquares(int n)
    {
        if (n <= 1)return n;
        vector<int> dp(n+1);//初始态 
        for (int i = 1;i <= n;++i)//从前往后
        {
            dp[i] = i;
            for(int k = 1;k * k <= i;++k) 
            {
                dp[i] = min(dp[i - k * k] + 1,dp[i]);
            }
        }
        return dp[n];
    }
};
```

## 单词拆分

给定一个非空字符串 s 和一个包含非空单词的列表 wordDict，判定 s 是否可以被空格拆分为一个或多个在字典中出现的单词。

说明：
- 拆分时可以重复使用字典中的单词。
- 你可以假设字典中没有重复的单词。

**力扣**：[word-break](https://leetcode-cn.com/problems/word-break)

示例 1：
输入: s = "leetcode", wordDict = ["leet", "code"]
输出: true
解释: 返回 true 因为 "leetcode" 可以被拆分成 "leet code"。

示例 2：
输入: s = "applepenapple", wordDict = ["apple", "pen"]
输出: true
解释: 返回 true 因为 "applepenapple" 可以被拆分成 "apple pen apple"。
注意你可以重复使用字典中的单词。

示例 3：
输入: s = "catsandog", wordDict = ["cats", "dog", "sand", "and", "cat"]
输出: false

哈希表是为了快速访问，保存动规状态才能递推。

复杂度：n * n + K

```cpp
class Solution {
public:
    bool wordBreak(string s, vector<string>& wordDict)
    {
        unordered_set<string> st(wordDict.begin(),wordDict.end());
        vector<bool> dp(s.size()+1);       
        dp[0] = true;//0 初态
        for(int i = 1;i <=s.size();++i)
        {
            for(int j = 0;j < i;++j)//dp，两次递归，遍历dp出所有的可能性
            {
                if (dp[j] && st.count(s.substr(j,i-j)))
                {
                    dp[i] = true;
                    break;
                }
            }
        }
        return dp[s.size()];
    }
};
```

[src: raw/ingested/2技术/算法/cpp_leetcode技巧-双遍历的一维数组.md]