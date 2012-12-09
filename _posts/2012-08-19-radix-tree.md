---
layout: post
title: radix-tree 源码分析
tags: [linux, 源码分析, C]
---

在判断一个文件内部的页是否为脏及页的状态时，所使用的数据结构就是radix-tree(基数).

    #define RADIX_TREE_INDIRCT_PTR 1
    #define RADIX_TREE_RETRY  ((void *)-1UL)
    #define RADIX_TREE_MAP_SHIFT (CONFIG_BASE_SMALL ? 4 : 6)
    #define RADIX_TREE_MAP_SIZE (1UL << RADIX_TREE_MAP_SHIFT)
    #define RADIX_TREE_MAP_MASK (RADIX_TREE_MAP_SIZE - 1)
    #define RADIX_TREE_TAG_LONGS ((RADIX_TREE_MAP_SIZE + BITS_PER_LONG - 1) / BITS_PER_LONG)
    
    struct radix_tree_root {
        unsigned int   height; //radix-tree 的高度
        gfp_t    gfp_mask; //内存分配的标志
        struct radix_tree_node *rnode; //树的根节点
    };
    
    struct radix_tree_node {
        unsigned int height; //该节点的高度(从底层开始)
        unsigend int count; //该节点的所存储的元素个数
        struct rcu_head rcu_head; //同步机制
        void *slots[RADIX_TREE_MAP_SIZE]; //存放具体的页指针
        unsigend long tags\[RADIX\_TREE\_MAX\_TAGS\]\[RADIX\_TREE\_TAGS\_LONGS\]; //标签标记
    }
    
当系统初始化时，初始化具体的radix-tree:

    void ___init radix_tree_init(void) 
    {
       radix_tree_node_cachep = kmem_cache_create("radix_tree_node",
               sizeof(struct radix_tree_node),0,
               SLAB_PANIC | SLAB_RECLAIM_ACCOUNT,
               radix_tree_node_ctor);
       radix_tree_init_maxindex();
       hotcpu_notifier(radix_tree_callback,0);
    }

其中最重要的部分就是*radix_tree_init_maxindex*函数:

    static __init void radix_tree_init_maxindex(void)
    {
       unsigned int i;
       
       for(i = 0; i < ARRAY_SIZE(height_to_maxindex); i++)
            height_to_maxindex[i] = __maxindex(i);
    }
    
    static __init unsigned long __maxindex(unsigned int height)
    {
        unsigned int width = height * RADIX_TREE_MAP_SHIFT;
        int shift  = RADIX_TREE_INDEX_BITS - width;
        
        if(shift < 0)
            return ~0UL;
        if(shift >= BITS_PER_LONG)
            return 0UL;
        return ~0UL >> shift;
    }
    
该函数的主要作用就是将一个key分为不同的部分，每一部分负责一层，计算每一层的具体掩码.

以32位为例:RADIX\_TREE\_MAP\_SHIFT = 4  RADIX\_TREE\_INDEX\_BITS 32

    高度          index      
     0         0x0000 0000   
     1         0x0000 000f   
     2         0x0000 00ff   
     ....     ............   
     8         0xffff ffff  


 当插入时就需要该索引进行获取对应高度的掩码，来进行判断该slot是否有数据以决定操作，具体如下图所示:
 
 ![radix-tree原理图](http://lwn.net/images/ns/kernel/radix-tree-2.png)

    int radix_tree_insert(struct raidx_tree_root *root,unsigned long index,void *item)
    {
        struct radix_tree_node *node = NULL,*slot;
        unsigned int height,shift;
        int offset;
        
        ......
        
        //不能超过树的最大表示范围，否则需要进行增加高度
        if(index > radix_tree_maxindex(root->heigth)) {
           error = radix_tree_extend(root,index);
           if(error)
               return error;
        }
        
        //这里不是很清楚，为什么低位做为indirect指针的标志?
        slot = radix_tree_indirect_to_ptr(root->rnode); 
        height = root->height;
        //所有高度需要进行的移位数
        shift = (height - 1) * RADIX_TREE_MAP_SHIFT;
        
        offset = 0;
        while(height > 0) {
            //node 记录父节点，slot则记录当前节点
            if(slot == NULL) {
                if(!(slot = radix_tree_node_allc(root)))
                       return -ENOMEM;
                slot->height  = height;
                if(node) {
                    rcu_assign_pointer(node->slots[offset],slot);
                    node->count++;
                } else
                   rcu_assign_pointer(root->rnode,radix_tree_ptr_to_indirect(slot));
            }
        
        offset = (index >> shift) & RADIX_TREE_MAP_MASK;
        node = slot;
        slot = node->slots[offset];
        shift -= RADIX_TREE_MAP_SHIFT;
        height --;
    }
    
    if(slot != NULL)
       return -EEXIST;
    
    //最后就直接将记录插入到父节点的叶子节点中
    if(node) {
       node->count ++;
       rcu_assign_pointer(node->slots[offset],item);
       BUG_ON(tag_get(node,0,offset));
       BUG_ON(tag_get(node,1,offset));
    } else {
       rcu_assign_pointer(root->rnode,item);
       BUG_ON(root_tag_get(root,0));
       BUG_NO(root_tag_get(root,1));
    }
    
    return 0;
    
    }
