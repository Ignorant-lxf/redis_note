# 动态字符串

```c

struct sdshdr { 
    
    // buf 中已占用空间的长度 使用len来判断可以保证字节数组存储的对象二进制安全 即不以'\0'来判断结尾
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};

```

* 动态字符串相比于普通 C 字符串的好处：
- 二进制安全，不是用 "\0" 结尾来标记字符串结尾，因此可以存储视频、图片等
- 不会有使用 C 标准库的字符串函数时出现的缓冲区溢出的问题
- 兼容部分标准的字符串函数

# 链表
```c
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value; //没有泛型 这也是没办法的办法
    
} listNode;
```

```c
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

    // 链表所包含的节点数量
    unsigned long len;

} list;
```

```c
typedef struct listIter {

    // 当前迭代到的节点
    listNode *next;

    // 迭代的方向
    int direction;

} listIter;

```
