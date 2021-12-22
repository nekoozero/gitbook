# Trie

又称单词查找树（前缀树，字典树），Trie树，是一种树形结构，是一种哈希树的变种。典型应用是用于统计，排序和保存大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计。

它的优点是：**利用字符串的公共前缀来减少查询时间，最大限度地减少无谓的字符串比较，查询效率比哈希树高。**

每个节点包含以下字段：

1. 指向子节点的指针数组 children。
2. 布尔字段isEnd们表示该节点是否为字符串的结尾。

[力扣208题](https://leetcode-cn.com/problems/implement-trie-prefix-tree/)就是实现前缀树(字符串是由26个英文小写字母组成)。

```java
class Trie {

    private boolean isEnd;

    private Trie[] t;

    public Trie() {
        t = new Trie[26];

    }
    
    public void insert(String word) {
        Trie node = this;
        for(int i = 0;i<word.length();i++) {
            Character ch = word.charAt(i);
            int index = ch - 'a';
            if(node.t[index]==null) node.t[index] = new Trie();

            node = node.t[index];
        }
        node.isEnd = true;
    }
    
    public boolean search(String word) {
        Trie node= this;
        for(int i=0;i<word.length();i++) {
            Character ch = word.charAt(i);
            int index = ch - 'a';
            if(node.t[index]==null) return false;

            node = node.t[index];
        }
        return node.isEnd;
    }
    
    public boolean startsWith(String prefix) {
        Trie node= this;
        for(int i=0;i<prefix.length();i++) {
            Character ch = prefix.charAt(i);
            int index = ch - 'a';
            if(node.t[index]==null) return false;

            node = node.t[index];
        }
        return true;
    }
}

/**
 * Your Trie object will be instantiated and called as such:
 * Trie obj = new Trie();
 * obj.insert(word);
 * boolean param_2 = obj.search(word);
 * boolean param_3 = obj.startsWith(prefix);
 */
```

在纯算法领域，前缀树算是一种较为常用的数据结构。

不过在工程中，不考虑前缀匹配的话，基本上使用hash就能满足，如果考虑前缀匹配的话，工程也不会使用Trie。

> 主要是字符集大小不好确定（题目只考虑26个字母，字符集的大小限制在较小的26内，因此可以使用Trie），工程一般兼容各种字符集，一旦字符集大小和大的话，Trie会带来很大的空间浪费。

至于联想输入、模糊匹配、全文搜索的典型场景在工程中主要是通过ES（ElasticSearch）解决的。ES的实现则主要是依靠**倒叙索引**。



