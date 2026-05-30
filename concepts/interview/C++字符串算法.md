# C++ 字符串算法

> 本文涵盖字符串经典算法：最长公共前缀、atoi、字符串相乘、反转字符串、有效括号、重复子字符串、旋转字符串、字母异位词、KMP/strStr。

See also: [[C++DFS回溯]], [[C++动态规划]], [[C++双指针]]

## 最长公共前缀

字符串数组中，求所有串的公共前缀。

**力扣**：[longest-common-prefix](https://leetcode-cn.com/problems/longest-common-prefix)

```cpp
// 最长公共前缀：以首串为基准，逐字符与其他串比较，缩短 len
string longestCommonPrefix(vector<string>& strs) {
    if (strs.empty()) return "";
    int len = strs[0].size();
    for (int i = 1; i < strs.size(); i++) {
        int k = 0;
        while (k < len && k < strs[i].size() && strs[0][k] == strs[i][k]) k++;
        len = k;
    }
    return strs[0].substr(0, len);
}
```

## 字符串转换整数 (atoi)

实现 atoi，将字符串转整数，忽略前导空格，支持正负号，溢出返回 INT_MAX/INT_MIN。

**力扣**：[string-to-integer-atoi](https://leetcode-cn.com/problems/string-to-integer-atoi)

```cpp
int myAtoi(string str) {
    if (str.empty()) return 0;
    int i = 0, op = 1, num = 0, sign = 0;
    for (; i < str.size(); i++) {
        if (str[i] == ' ') { if (sign > 0) return num; continue; }
        else if (str[i] == '+') { if (sign++ > 0) return num; continue; }
        else if (str[i] == '-') { if (sign++ > 0) return num; op = -1; continue; }
        else if (isdigit(str[i])) break;
        return num;
    }
    for (; i < str.size(); i++) {
        if (isdigit(str[i])) {
            if (abs((long)num * 10 + (str[i] - '0')) > (long)INT_MAX)
                return op == 1 ? INT_MAX : INT_MIN;
            num = num * 10 + (str[i] - '0');
        } else break;
    }
    return op * num;
}
```

## 字符串相乘

两字符串表示的非负整数相乘，返回字符串。

**力扣**：[multiply-strings](https://leetcode-cn.com/problems/multiply-strings)

```cpp
// 字符串相乘：竖式乘法，低位到高位
string multiply(string num1, string num2) {
    vector<int> mul(num1.size() + num2.size(), 0);
    for (int i = 0; i < num1.size(); i++)
        for (int j = 0; j < num2.size(); j++)
            mul[i + j] += (num1[num1.size() - 1 - i] - '0') * (num2[num2.size() - 1 - j] - '0');
    for (int i = 0, c = 0; i < mul.size(); i++) {
        int tmp = mul[i] + c;
        mul[i] = tmp % 10;
        c = tmp / 10;
    }
    int i = mul.size() - 1;
    while (i > 0 && mul[i] == 0) i--;
    string res;
    for (; i >= 0; i--) res.push_back('0' + mul[i]);
    return res;
}
```

## 反转字符串

原地反转字符数组。

**力扣**：[reverse-string](https://leetcode-cn.com/problems/reverse-string)

```cpp
void reverseString(vector<char>& s) {
    for (int b = 0, e = s.size() - 1; b < e; b++, e--) swap(s[b], s[e]);
}
```

## 反转字符串中的单词 III

反转字符串中每个单词的字符顺序，单词间保留空格。

**力扣**：[reverse-words-in-a-string-iii](https://leetcode-cn.com/problems/reverse-words-in-a-string-iii)

```cpp
string reverseWords(string s) {
    for (int i = 0, left = 0; i <= (int)s.size(); i++) {
        if (s[i] == ' ' || s[i] == '\0') {
            reverse(s.begin() + left, s.begin() + i);
            while (i < (int)s.size() && s[i] == ' ') i++;
            left = i;
        }
    }
    return s;
}
```

## 有效括号

判断只含 ()[]{} 的字符串是否括号正确匹配。

**力扣**：[valid-parentheses](https://leetcode-cn.com/problems/valid-parentheses)

```cpp
// 有效括号：左括号压栈对应右括号，右括号弹栈匹配
bool isValid(string s) {
    stack<char> st;
    for (char c : s) {
        if (c == '(') st.push(')');
        else if (c == '{') st.push('}');
        else if (c == '[') st.push(']');
        else if (st.empty() || st.top() != c) return false;
        else st.pop();
    }
    return st.empty();
}
```

## 重复的子字符串

判断 s 是否可由某子串重复多次构成。

**力扣**：[repeated-substring-pattern](https://leetcode-cn.com/problems/repeated-substring-pattern)

```cpp
bool repeatedSubstringPattern(string s) {
    return (s + s).substr(1, 2 * s.size() - 2).find(s) != string::npos;
}
```

## 旋转字符串

s 经若干次左旋能否变成 goal。

**力扣**：[rotate-string](https://leetcode-cn.com/problems/rotate-string)

```cpp
bool rotateString(string A, string B) {
    return A.size() == B.size() && (A + A).find(B) != string::npos;
}
```

## 有效的字母异位词

判断 t 是否是 s 的字母异位词。

**力扣**：[valid-anagram](https://leetcode-cn.com/problems/valid-anagram)

```cpp
bool isAnagram(string s, string t) {
    if (s.size() != t.size()) return false;
    vector<int> cnt(26);
    for (char c : s) cnt[c-'a']++;
    for (char c : t) if (--cnt[c-'a'] < 0) return false;
    return true;
}
```

## KMP / strStr

在 haystack 中找 needle 首次出现位置。

**力扣**：[implement-strstr](https://leetcode-cn.com/problems/implement-strstr)

```cpp
int strStr(string haystack, string needle) {
    if (needle.empty()) return 0;
    int n = haystack.size(), m = needle.size();
    if (n < m) return -1;
    vector<int> next(m);
    next[0] = -1;
    for (int i = 1, j = -1; i < m; i++) {
        while (j >= 0 && needle[j+1] != needle[i]) j = next[j];
        if (needle[j+1] == needle[i]) j++;
        next[i] = j;
    }
    for (int i = 0, j = 0; i < n;) {
        if (j == -1) { i++; j = 0; }
        else if (haystack[i] == needle[j]) {
            i++; j++;
            if (j == m) return i - m;
        } else j = next[j];
    }
    return -1;
}
```

## 重复叠加字符串匹配

给定两个字符串 a 和 b，寻找重复叠加字符串 a 的最小次数，使得字符串 b 成为叠加后的字符串 a 的子串，如果不存在则返回 -1。

**力扣**：[repeated-string-match](https://leetcode.cn/problems/repeated-string-match/)

```cpp
int repeatedStringMatch(string A, string B) {
    int c = 1, asize = A.size(), bsize = B.size(), basize = bsize / asize;
    for (string tmp = A; c <= basize + 2; tmp += A, c++){
        if (tmp.size() >= bsize && tmp.find(B) != string::npos){
            return c;
        }
    }
    return -1;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十、字符串.md]
[src: raw/ingested/2技术/算法/cpp_leetcode_腾讯精选练习50题-[字符串转换整数-(atoi)](https---leetcode-cn.com-problems-string-to-.md]
[src: raw/ingested/2技术/算法/cpp_leetcode技巧-字符串判断.md]