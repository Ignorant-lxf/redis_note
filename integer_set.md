# 整数集合

```c
typedef struct intset {
    
    // 编码方式
    uint32_t encoding; //初始编码为INTSET_ENC_INT16

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组 这里是整数集合的核心,可以进行动态扩容
    int8_t contents[];
    
} intset;
```

往里面添加元素
```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {

    // 计算编码 value 所需的长度,当所插入的数的范围区间大于当前的编码范围区间的时候进行扩容
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;

    // 默认设置插入为成功
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    // 如果 value 的编码比整数集合现在的编码要大
    // 那么表示 value 必然可以添加到整数集合中
    // 开始升级 也就是把当前集合内的转换为范围更大的编码 比如数字从32位存储成64位
    if (valenc > intrev32ifbe(is->encoding)) { //进行升级
        /* This always succeeds, so we don't need to curry *success. */
        // T = O(N)
        return intsetUpgradeAndAdd(is,value); 
    } else {
		//证明到这里的时候我们不需要升级
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        // 在整数集合中查找 value ，看他是否存在：
        // - 如果存在，那么将 *success 设置为 0 ，并返回未经改动的整数集合
        // - 如果不存在，那么可以插入 value 的位置将被保存到 pos 指针中
        //   等待后续程序使用
        if (intsetSearch(is,value,&pos)) { //从现有的集合中查找value 如果找到的话设置其插入的索引为pos.
            if (success) *success = 0;
            return is;
        }

        // 运行到这里，表示 value 不存在于集合中
    
        // 为 value 在集合中分配空间
        // 这里面使用了zrmalloc,这会保存扩容前的数据,这也是为什么结构体最后一个数据要保存一个指针的原因吧
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        // 那么需要对现有元素的数据进行移动，空出 pos 上的位置，用于设置新值
        // 举个例子
        // 如果数组为：
        // | x | y | z | ? |
        //     |<----->|
        // 而新元素 n 的 pos 为 1 ，那么数组将移动 y 和 z 两个元素
        // | x | y | y | z |
        //         |<----->|
        // 这样就可以将新元素设置到 pos 上了：
        // | x | n | y | z |
        // T = O(N)
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1); 
        //这里其实就是如果插入的位置不是末尾的话,因为这是set,所以我们需要保证有序.所以要把插入后面的元素移动一格
    }

    // 将新值设置到底层数组的指定位置中
    _intsetSet(is,pos,value); 

    // 增一集合元素数量的计数器
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);

    // 返回添加新元素后的整数集合
    return is;
}

```
整数集合底层是数组，在元素较多时插入效率较低。

**refer https://blog.csdn.net/weixin_43705457/article/details/104961895**