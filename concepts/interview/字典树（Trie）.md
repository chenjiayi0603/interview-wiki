# 字典树（Trie）

字典树（Trie，又称前缀树）是一种用于高效存储和检索字符串集合的树形数据结构。每个节点代表一个字符，从根到叶子的路径表示一个字符串。

## 核心结构

```cpp
struct Trie {
    bool isWord = false;
    Trie* next[26] = {};
    void insert(const string& word) {
        Trie* cur = this;
        for (char c : word) {
            int i = c - 'a';
            if (!cur->next[i]) cur->next[i] = new Trie();
            cur = cur->next[i];
        }
        cur->isWord = true;
    }
    int find(const string& word) {
        Trie* cur = this;
        for (int i = 0; i < word.size(); i++) {
            int idx = word[i] - 'a';
            if (!cur->next[idx]) return -1;
            cur = cur->next[idx];
            if (cur->isWord) return i;
        }
        return -1;
    }
};
```

## 应用示例：单词替换

用最短词根替换继承词。

**力扣**：[replace-words](https://leetcode-cn.com/problems/replace-words)

```cpp
string replaceWords(vector<string>& dict, string sentence) {
    Trie* root = new Trie();
    for (auto& w : dict) root->insert(w);
    string res, word;
    stringstream ss(sentence);
    while (ss >> word) {
        int pos = root->find(word);
        res += (pos == -1 ? word : word.substr(0, pos + 1)) + " ";
    }
    if (res.size()) res.pop_back();
    return res;
}
```

[src: raw/ingested/2技术/算法/C++算法精选合并版-十八、高级数据结构.md]