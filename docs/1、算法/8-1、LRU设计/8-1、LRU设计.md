## 1、设计思想

## 2、LRU缓存设计
运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制(146)  
思路：  
* 使用双向链表
* 创建头尾虚拟节点，省去判空
* 预先定义删除节点和插入头节点的方法
* 插入的时候有三种情况，重复，满了，插入
* 需要定义容量大小字段
```
Map<Integer, Node> search;
Node head;
Node tail;
int capacity;

public LRUCache(int capacity) {
    search = new HashMap<>(capacity);
    this.capacity = capacity;
    head = new Node();
    tail = new Node();
    head.next = tail;
    tail.pre = head;
}

public int get(int key) {
    Node node = search.get(key);
    if (node == null) {
        return -1;
    }
    deleteTarget(node);
    insertHead(node);
    return node.val;
}

public void put(int key, int value) {
    if (search.containsKey(key)) {
        Node oldNode = search.get(key);
        oldNode.val = value;
        deleteTarget(oldNode);
        insertHead(oldNode);
        return;
    }
    if (search.size() == capacity) {
        Node remove = tail.pre;
        search.remove(remove.key);
        deleteTarget(remove);
    }
    Node newNode = new Node(key, value);
    insertHead(newNode);
    search.put(key, newNode);
}

// hair <-> a->[]-c->null
private void insertHead(Node target) {
    Node headNext = head.next;
    headNext.pre = target;
    head.next = target;
    target.pre = head;
    target.next = headNext;
}

//移除target节点
private void deleteTarget(Node target) {
    target.pre.next = target.next;
    target.next.pre = target.pre;
}

class Node {
    public int val;
    public int key;
    public Node next;
    public Node pre;
    public Node (){}
    public Node (int key, int val){
        this.key = key;
        this.val = val;
    }
}
```
