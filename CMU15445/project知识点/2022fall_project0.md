# C++ 知识
## std::unqiue_str
* 独占指针（unqiue_str）通过 `get()` 方法获取指针所保存的对象
* `reset()` 方法可以将指针的指向重设，独占指针如果更换其中的对象，需要使用`reset()`进行重设
* 在独占指针`reset()`成子类的时候，在访问子类，需要通过 `dynamic_cast<子类类型指针>(子类指针)`进行转换然后去访问子类


## 经验
* 容器通过`find()`方法去查找容器中是否存在，如果不存在，则 `== end()`
* `std::forward`为转发函数，将左值或右值引用转发到另一个函数
* 容器第一次使用前最好调用`clear()`去处理一下
* 字符串的操作，使用迭代器进行访问每一个元素
* 如果是访问链式的内容，但是仅通过最后一个元素进行判断以及操作这个表链，可以通过一遍迭代，然后保留最后的节点进行操作，如果需要上一节点，就需要通过快慢指针对上一节点进行记录
* 通过栈可以模仿递归操作
* 子类调用构造函数的时候，必须也调用父类的构造函数，父类中有些变量需要通过构造函数去赋值，但是子类初始化的时候不会去赋值
****
# Trie树
## 使用 vector 实现
```c++
class Trie {
public:
    Trie() :children(26), isEnd(false){
    }
    
    void insert(string word) {
        Trie* node = this;
        for (char ch : word) {
            ch -= 'a';
            if (node->children[ch] == nullptr) {
                node->children[ch] = new Trie();
            }
            node = node->children[ch];
        }
        node->isEnd = true;
    }
    
    bool search(string word) {
        Trie* node = searchPrefix(word);
        return node!=nullptr && node->isEnd;
    }
    
    bool startsWith(string prefix) {
        return this->searchPrefix(prefix)!=nullptr;
    }
private:
    vector<Trie*> children;
    bool isEnd;

    Trie* searchPrefix(string prefix) {
        Trie* node = this;
        for (auto ch: prefix) {
            ch -= 'a';
            if (node->children[ch] == nullptr) return nullptr;
            node = node->children[ch];
        }
        return node;
    }
};

/**
 * Your Trie object will be instantiated and called as such:
 * Trie* obj = new Trie();
 * obj->insert(word);
 * bool param_2 = obj->search(word);
 * bool param_3 = obj->startsWith(prefix);
 */
```
* 缺点，会有节点冗余

leetcode: [208. 实现 Trie (前缀树)](https://leetcode.cn/problems/implement-trie-prefix-tree/)
## project0 实现
* 通过`unordered_map`以及`独占指针`进行实现，
* 一个是不存值的节点（TrieNode），一个是继承了不存值的节点的存值子节点（TrieNodeWithValue）

****
# 调试经验
## Test的cpp文件
* 先看单测，在编写代码，逐个调式
* 每个测试名称的`DISABLED_`前缀删除才能进行测试
* 通过 make 总测试代码中的每个单测名称，对单测进行构建单测可执行文件
## 格式检查
```shell
$ make format
$ make check-lint
$ make check-clang-tidy-p0
```
* 格式检查的时候，`if else` 都需要带 `{}`
* 这个会检查你的错误
## 内存泄露检查
```shell
valgrind \
    --error-exitcode=1 \
    --leak-check=full \
    ./test/starter_trie_test
```
## 开发提示
```c++
LOG_INFO("# Pages: %d", num_pages);
LOG_DEBUG("Fetching page %d", page_id);
```
* 用来添加日志
