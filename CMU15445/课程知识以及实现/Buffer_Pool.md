#
## 缓存替换策略
### LRU（Least Recently Used）不同实现
* 使用 链表+hash表 实现
```c++
struct DLinkNode {
    int key, value;
    DLinkNode* pre;
    DLinkNode* next;
    DLinkNode(): key(0), value(0), pre(nullptr), next(nullptr) {}
    DLinkNode(int _key, int _value): key(_key), value(_value), pre(nullptr), next(nullptr) {}
};

class LRUCache {
public:
    LRUCache(int _capacity): size(0), capacity(_capacity) {
        // 通过双向链表和哈希表来存储和搜索节点
        head = new DLinkNode();
        tail = new DLinkNode();
        head->next = tail;
        tail->pre = head;
    }
    
    int get(int key) {
        if (!cache.count(key)) return -1;
        DLinkNode* node = cache[key];
        MoveToHead(node);
        return node->value;

    }
    
    void put(int key, int value) {
        // 如果key不在缓存区中
        if (!cache.count(key)) {
            DLinkNode* node = new DLinkNode(key, value);
            if (size >= capacity) {
                DLinkNode* oldtail = RemoveTail();
                cache.erase(oldtail->key);
                delete oldtail;
            }
            else {
                size++;
            }
            AddToHead(node);
            // 添加新的key
            cache[key] = node;
        }
        // 如果key在缓冲区中，只需要更改节点在链表中的位置同时改变value就行
        else {
            DLinkNode* node = cache[key];
            node->value = value;
            MoveToHead(node);
        } 
    }
    
    DLinkNode* RemoveTail() {
        DLinkNode* _tail = tail->pre;
        _tail->pre->next = tail;
        tail->pre = _tail->pre;
        return _tail;
    }

    void AddToHead(DLinkNode * node) {
        node->next = head->next;
        head->next->pre = node;
        head->next = node;
        node->pre = head;
    }

    void MoveToHead(DLinkNode* node) {
        // 修改当前节点
        node->next->pre = node->pre;
        node->pre->next = node->next;
        // 修改头
        AddToHead(node);
    }


private:
    unordered_map<int, DLinkNode*> cache;
    DLinkNode* head;
    DLinkNode* tail;
    int size;
    int capacity;
};

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache* obj = new LRUCache(capacity);
 * int param_1 = obj->get(key);
 * obj->put(key,value);
 */
```
leetcode: [146. LRU 缓存](https://leetcode.cn/problems/lru-cache/)
### CLOCK 模糊LRU

