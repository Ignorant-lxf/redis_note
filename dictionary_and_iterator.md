# 字典

字典结构是 redis 中非常重要的一种结构，不仅可以作为数据库的结构，还可以作为哈希表的结构，是一个非常重要的数据结构

```cassandraql
//哈希表结点
typedef struct dictEntry {
    
    // 键 用于哈希冲突的时候进行比较 
    void *key;

    // 存储值 很巧妙 使得哈希表中的值可以存任何类型,其十分节省内存
union {
        void *val;
uint64_t u64;
int64_t s64;
} v;

    // 指向下个哈希表节点，形成链表 防止哈希冲突
struct dictEntry *next;

} dictEntry;

//哈希表
typedef struct dictht {
    
    // 哈希表数组  一个指针数组
    dictEntry **table;

    // 哈希表大小
unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
unsigned long sizemask;

    // 该哈希表已有节点的数量 用于O(1)计算size与计算负载因子
unsigned long used;

} dictht;

/*
 * 字典
 */
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
void *privdata;

    // 哈希表 两个的原因是用作在扩容和减少容量的时候rehash
dictht ht[2];

    // 判断rehash是否进行 当 rehash 不在进行时，值为 -1
int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量 文章后面会详细说说这个
int iterators; /* number of iterators currently running */

} dict;

```

字典里面添加 key

```c
dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;

    //如果运行的安全 安全迭代器数量为零的话才会进行单步rehash,其中会跳过为NULL的entry
    //为什么安全迭代器数量为零的话才会进行单步rehash呢?原因是为了保证安全迭代器的安全性,
    //防止一次迭代器遍历中一个entry被遍历了两次 当迭代器遍历和rehash同时进行时就会有这种情况
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    // 计算键在哈希表中的索引值
    // 如果值为 -1 ，那么表示键已经存在
    // 如果正在进行rehash 返回的index是ht[1]中需要插入的下标 否则返回ht[0]中的索引
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry */
    //因为上面我们根据现在是否进行rehash返回的索引不同,所以这里要判断返回的所以是ht[0]上的还是ht[1]上的.
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    // 为新节点分配空间
    entry = zmalloc(sizeof(*entry));
    // 将新节点插入到链表表头 原因是我们并没有维护尾部结点 这样可以保证O(1)的插入,而且和尾插是一样的(硬要说局部性那就没办法了)
    entry->next = ht->table[index]; 
    ht->table[index] = entry;
    // 更新哈希表已使用节点数量
    ht->used++;
	//entry中肯定是要保存key的 不然没办法处理哈希冲突 每一个key在哈希冲突以后需要在对应的桶中比较嘛
    dictSetKey(d, entry, key); 
	
	//返回分配好的entry
    return entry;
}

```

rehash

```c
int dictRehash(dict *d, int n) {

    // 只可以在rehash进行中时执行 这没什么毛病 相当于一个检测了
    if (!dictIsRehashing(d)) return 0;
	
	//传入的n的意思是把n个有值的项从ht[0]转移到ht[1].
    while(n--) {
        dictEntry *de, *nextde;

        /* Check if we already rehashed the whole table... */
        //如果发现ht[0]中值存在的元素为0,代表我们已经完成了这次rehash,释放ht[0]的资源
        if (d->ht[0].used == 0) {
            zfree(d->ht[0].table);
            // 将ht[1]设置为ht[0]
            d->ht[0] = d->ht[1];
            // 重置旧的 1 号哈希表
            _dictReset(&d->ht[1]);
            // 关闭 rehash 标识
            d->rehashidx = -1;
            //只有两个返回值 0代表此次rehash已经完成 1代表未完成(返回1的时候可能已经完成,但外界不知道而已) 
            return 0;
        }

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        // 确保 rehashidx 没有越界
        assert(d->ht[0].size > (unsigned)d->rehashidx);

       	 //找到一个非空就退出 不然可能会有多次单步rehash是无效的 这也意味着有效值在rehash时存在在两张表中
        while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;

        // 指向该索引的链表表头节点
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        // 将链表中的所有节点迁移到新哈希表
        while(de) {
            unsigned int h;

            // 保存下个节点的指针
            nextde = de->next;

            /* Get the index in the new hash table */
            // 计算新哈希表的哈希值，以及节点插入的索引位置
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;

            // 插入节点到新哈希表
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;

            // 更新计数器
            d->ht[0].used--;
            d->ht[1].used++;

            // 继续处理下个节点
            de = nextde;
        }
        // 将刚迁移完的哈希表索引的指针设为空
        d->ht[0].table[d->rehashidx] = NULL;
        // 更新 rehash 索引
        d->rehashidx++;
    }
    return 1;
}

```

单步rehash，每次更新字典中对应 rehashidx 索引处对应的链表到 ht[1],然后更新 rehashidx；倘若 rehashidx 为-1，则说明渐进式
rehash 结束

* 那么可以每次只转移我们需要的部分吗？ 因为我们可以通过 key 与 sizemask 相与得到 ht[1]里面是否为 Null 可以判断是否完成rehash

其实不然，如果每次操作都是某个固定区间的值，会导致 rehash 永远不结束。 而在 rehash 时是不能进行扩容的，因此会导致负载因子变大而影响效率

**字典扩容**

```c
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    // 渐进式 rehash 已经在进行了，直接返回 这也是上面那一大段话中提到的问题,当rehash正在进行时不会再次扩容了
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    // 如果字典（的 0 号哈希表）为空,为ht[0]分配空间
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    // 一下两个条件之一为真时，对字典进行扩展
    // 1）字典已使用节点数和字典大小之间的比率接近 1：1
    //    并且 dict_can_resize 为真 当我们进行后台持久化的时候会设置这个值 此时会产生子进程 不希望此时进行扩容 
    	//这里有疑问 当子进程exec以后与父进程就没关系了呀,为什么要在子进程存在的时候节约内存而把ratio设置成5呢?
    // 2）已使用节点数和字典大小之间的比率超过 dict_force_resize_ratio 5
    if (d->ht[0].used >= d->ht[0].size &&dict_can_resize ||
        d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        // 新哈希表的大小至少是目前已使用节点数的两倍
        return dictExpand(d, d->ht[0].used*2);
    }

    return DICT_OK;
}
```

```c
int dictExpand(dict *d, unsigned long size)
{
    // 新哈希表
    dictht n; /* the new hash table */

    // 根据 size 参数 计算哈希表的大小 其实就是大于等于size的第一个2的幂
    unsigned long realsize = _dictNextPower(size);

    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    // 不能在字典正在 rehash 时进行
    // size 的值也不能小于 0 号哈希表的当前已使用节点
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    // 为哈希表分配空间，并将所有指针指向 NULL
    n.size = realsize;
    n.sizemask = realsize-1;
    // T = O(N)
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    // 如果 0 号哈希表为空，那么这是一次初始化：
    // 程序将新哈希表赋给 0 号哈希表的指针，然后字典就可以开始处理键值对了。
    if (d->ht[0].table == NULL) { //这也是我们在_dictExpandIfNeeded是提到的问题 第一次操作时进行初始化
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    // 如果 0 号哈希表非空，那么这是一次 rehash ：
    // 程序将新哈希表设置为ht[1]
    // 设置字典的rehash rehash开始
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}

```

查找中的 rehash

```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    unsigned int h, idx, table;

    // 字典（的哈希表）为空
    if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */

    // 进行单步rehash 上面已经说过这个问题
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 计算键的哈希值
    h = dictHashKey(d, key);
    // 在字典的哈希表中查找这个键
    // T = O(1)
    for (table = 0; table <= 1; table++) {

        // 计算索引值
        idx = h & d->ht[table].sizemask;

        // 遍历给定索引上的链表的所有节点，查找 key
        he = d->ht[table].table[idx];
        while(he) {

            if (dictCompareKeys(d, key, he->key))
                return he;

            he = he->next;
        }
		
		//如果找完ht[0],没有找到且正在进行rehash的话,我们需要再去找ht[1]
		//上面已经说过 此时有效值是存在两张表中的
        if (!dictIsRehashing(d)) return NULL;
    }

    // 进行到这里时，说明两个哈希表都没找到
    return NULL;
}

```

**迭代器**

```c
typedef struct dictIterator {
        
    // 被迭代的字典
    dict *d;
    
    int table, index, safe;
	//table取值只有两种结果 即0或1
	//index是当前遍历到的哈希桶的序号
	//这个迭代器是否是安全的 下面会说

    // entry ：当前迭代到的节点的指针
    // nextEntry ：当前迭代节点的下一个节点
    //             因为在安全迭代器运作时， entry 所指向的节点可能会被修改，
    //             所以需要一个额外的指针来保存下一节点的位置，
    //             从而防止指针丢失
    dictEntry *entry, *nextEntry;

    long long fingerprint; /* unsafe iterator fingerprint for misuse detection */
	//非安全迭代器要使用的验证机制 指纹 原理是通过比较指纹来判断迭代器遍历时是否会出现重复遍历的情况 下面会说
} dictIterator;

```

* 安全迭代器与非安全迭代器  
  前面我们知道，在 rehash 的过程中，有效的值存储在两个哈希表中，因此可能在遍历完 ht[0] 里面的元素后，元素 rehash
  到了 ht[1]，因此出现了 安全迭代器与非安全迭代器。当有安全迭代器时，不允许进行 rehash

迭代器的构造函数

```c
dictIterator *dictGetIterator(dict *d)
{
    dictIterator *iter = zmalloc(sizeof(*iter));

    iter->d = d;
    iter->table = 0; //默认ht[0]
    iter->index = -1;
    iter->safe = 0; //非安全迭代器
    iter->entry = NULL;
    iter->nextEntry = NULL;

    return iter;
}

dictIterator *dictGetSafeIterator(dict *d) {
    dictIterator *i = dictGetIterator(d);

    // 设置安全迭代器标识
    i->safe = 1;

    return i;
}
```

可以看到，safe 值为 0 的是非安全迭代器，safe 值为 1 的是安全迭代器

- 指纹是一个64位的数字代表了当前时间字典的状态,这只是将几个字典的属性亦或在一起,当一个不安全的迭代器被初始化的时候,我们会得到字典的指纹,而且会在其被释放时检查它,如果两个指纹不同的话就意味着在迭代器迭代的时候执行了被禁止的操作.
- 它使用到了两张表的table,size和used字段,也就是如果发生修改或者rehash,这个指纹就会改变,如果发生改变,就可能对我们的遍历的结果造成影响.所以说指纹的存在可以很好的发现是否进行了rehash.