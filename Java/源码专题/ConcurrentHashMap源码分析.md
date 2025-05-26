## 一 ConcurrentHashMap 核心结构

### 1.1 数据结构
哈希表 + 链表 + 红黑树  

### 1.2 初始容量和扩容阈值
✅  初始容量: 16  
✅  扩容大小: 2倍   
✅  扩容阈值: 75%  

### 1.3 sizeCtl是什么？不同的值代表什么含义
**sizeCtl** 是ConcurrentHashMap重要字段，它是用来控制和协调**哈希表**的扩容和线程初始化行为  
✅  0: 表示哈希表还没有初始化  
✅  N(N > 0): 表示下次扩容的阈值，比如当前容量是16，那么这里就是12，表示下次从16扩容到32的时候，触发条件就是当前容量大于12  
✅  -1：表示某一个线程正在初始化  
✅  -(1+N): 表示有N个线程正在帮助扩容  

### 1.4 Node节点类型
✅  Node: 普通的Node节点  
✅  TreeNode: 红黑树节点  
✅  TreeBin: 红黑树的根节点  
✅  ForwardingNode:  旧的哈希表某个位置已经迁移完了，就会将该位置设置为该类型，他的next指针指向新的哈希表中真实的数据  
✅  ReverseNode: 保留节点  

### 1.5 ConcurrentHashMap中hash code计算方式和HashMap有什么不一样，为什么这么设计？
**HashMap的hash code计算方式:**  
```
h = key.hashcode()
h ^ (h >>> 16)
```
这种方式计算出来的hash code 有可能是负数  


**ConcurrentHashMap的hash code计算方式:**  
```
h = key.hashcode() 
h ^ (h >>> 16) & 0x7fffffff 
```
❗ 这种方式计算计算出来的hash code是没有负数的  
❗ 因为ConcurrentHashMap的hash code如果是负数，某些负数是有特殊含义的  
❗ 比如 -1 -> MOVE 代表 当前节点ForwardingNode -2 -> TREEBIN 表示当前节点是TreeBin -3 -> RESERVED 预留使用  


## 二 ConcurrentHashMap添加和删除流程

### 2.1 ConcurrentHashMap添加流程
✅  根据key计算hashcode, 根据hashcode & (cap - 1)进行寻址  
✅  如果该位置没有元素，通过CAS设置桶内第一个元素  
✅  如果CAS设置失败，意味着其他线程已经设置成功，此时走桶不是空的逻辑  
✅  如果该位置有元素，则需要判断当前节点是链表还是红黑树  
✅  如果是链表，则通过synchronized同步代码块添加数据  
✅  如果是红黑树，则获取TreeBin根节点的lockState锁状态:  0 未锁定 1 已锁定 -1 读锁定状态  
1. [ ] 如果当前没有其他线程抢占锁，则通过CAS方式修改锁状态，修改成功，获取到锁，更新  
2. [ ] 如果有其他线程抢占锁或者CAS修改失败，则将当前线程挂起，等待被唤醒  

**注意:**   
❗ Treebin中实现了读写分离，如果当前锁状态是读锁，后续的读线程可以持有锁  
❗ 如果当前锁状态是读锁，后面的线程是写锁，则需要等读锁释放后才可以写  
❗ 或者当前是写锁，后面是读线程或者写线程，则更需要等待写锁释放后才可以继续操作  


### 2.2 ConcurrentHashMap删除流程
✅  根据key计算hashcode, 根据hashcode & (cap - 1)进行寻址, 看落在哪一个桶内  
✅  判断桶内的节点是链表节点还是红黑树  
✅  链表的话，通过synchronized同步代码块进行删除  
✅  红黑树的话，先要根据根节点Treebin获取写锁，即lockState = 1， 获取到写锁，然后进行删除  

## 三 ConcurrentHashMap扩容机制

### 3.1 判断是否还能继续扩容
✅  哈希表扩容也是有限制，不可能无限制扩容  
✅  扩容阈值最大允许就是Integer.MAX_VALUE,  如果超过这个极限，哈希表不会再进行扩容  

### 3.2 更新sizeCtl
✅  如果有线程到来发现正在扩容，则会加入进来帮只扩容  
✅  首先会通过CAS方式更新sizeCtl  
✅  当前的值如果是-2，表示有一个线程在帮助扩容; 那么此时更新为-3，表示有2个线程再帮助扩容  

### 3.3 帮助扩容和数据迁移
✅  新的哈希表容量是以前2倍，但是最大不超过Integer.MAX_VALUE    
✅  对旧的哈希表，提供了一个transferIndex变量，代表的是当前旧的哈希表数据已经迁移到什么位置了  
1. [ ] 第一个线程获取到的transferIndex=(旧的哈希表的容量大小-1)，这也就意味着他是从就哈希表的尾部开始头部进行数据迁移  
2. [ ] 每一个新来的线程并不是按照一个桶一个桶的分配，而是按照步长分配连续多个桶  
3. [ ] 如果transferIndex如果为负数，表示不需要线程再帮助扩容了，扩容已经完成  

✅  在数据的迁移过程中, 如果某个桶迁移完毕，设置为ForwardingNode, next指针会指向新的位置；如果没有ForwardingNode,表示哈希表还没有迁移完成，只能从原始表查询数据  
