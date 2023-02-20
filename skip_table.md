# 跳跃表
```c
typedef struct zskiplistNode {

    // 成员对象 用于存储真正的值
    robj *obj;

    // 分值 我们在存储有序set是需要指定的分值
    double score;

    // 回退指针 
    struct zskiplistNode *backward;

    // 层的数据结构 跳跃表的核心
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度 用于更快的(logn)算出某节点在全部节点中的排名
        unsigned int span;

    } level[]; //默认32

} zskiplistNode;

/*
 * 跳跃表
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;


```
**跳跃表的插入**
```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
 
    // 记录寻找元素过程中，每层能到达的最右节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
 
    // 记录寻找元素过程中，每层所跨越的节点数
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
 
    int i, level;
 
    redisAssert(!isnan(score));
    x = zsl->header;
    // 记录沿途访问的节点，并计数 span 等属性
    // 平均 O(log N) ，最坏 O(N)
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
 
        // 右节点不为空
        while (x->level[i].forward &&                   
            // 右节点的 score 比给定 score 小
            (x->level[i].forward->score < score ||      
                // 右节点的 score 相同，但节点的 member 比输入 member 要小
                (x->level[i].forward->score == score && 
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            // 记录跨越了多少个元素
            rank[i] += x->level[i].span;
            // 继续向右前进
            x = x->level[i].forward;
        }
        // 保存访问节点
        update[i] = x;
    }
 
    /* we assume the key is not already inside, since we allow duplicated
     * scores, and the re-insertion of score and redis object should never
     * happpen since the caller of zslInsert() should test in the hash table
     * if the element is already inside or not. */
    // 因为这个函数不可能处理两个元素的 member 和 score 都相同的情况，
    // 所以直接创建新节点，不用检查存在性
 
    // 计算新的随机层数
    level = zslRandomLevel();
    // 如果 level 比当前 skiplist 的最大层数还要大
    // 那么更新 zsl->level 参数
    // 并且初始化 update 和 rank 参数在相应的层的数据
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
 
    // 创建新节点
    x = zslCreateNode(level,score,obj);
    // 根据 update 和 rank 两个数组的资料，初始化新节点
    // 并设置相应的指针
    // O(N)
    for (i = 0; i < level; i++) {
        // 设置指针
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;
 
        /* update span covered by update[i] as x is inserted here */
        // 设置 span
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
 
    /* increment span for untouched levels */
    // 更新沿途访问节点的 span 值
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
 
    // 设置后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    // 设置 x 的前进指针
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        // 这个是新的表尾节点
        zsl->tail = x;
 
    // 更新跳跃表节点数量
    zsl->length++;
 
    return x;
}
```

# refer to https://blog.csdn.net/u013536232/article/details/105476382