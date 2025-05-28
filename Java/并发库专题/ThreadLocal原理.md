## 一 什么是 ThreadLocal？ ThreadLocal 的使用场景有哪些？
### 1.1 什么是 ThreadLocal
✅ ThreadLocal是Java提供的一种**线程局部变量**机制，可以为每个线程提供一个**独立的变量副本**，线程之间互不干扰，常用于线程安全的场景    
✅ 或者说hreadLocal指的是本地线程变量，一个变量在线程生命周期内是有效，并且不受其他线程干扰  
✅ 既然是在线程生命周期内有效，那么同一个线程就可以在不同的方法获取到这个变量  

### 1.2 ThreadLocal 的使用场景有哪些
✅ 用户会话（如：ThreadLocal<User>）  
✅ 数据库连接管理（如：ThreadLocal<Connection>）  
✅ 数据库事务管理（如：ThreadLocal<Connection>）  
✅ 日志追踪上下文（如：日志链路 ID）  

## 二 ThreadLocal 和 Synchronized 的区别？
ThreadLocal和Synchronized都是为了保证线程并发安全的机制，但是也存在一些差异:    
### 2.1 原理不同
1. [ ] ThreadLocal: 每个线程维护自己的变量副本
2. [ ] Synchronized: 多线程共享同一个变量，靠锁控制访问

### 2.2 开销不同
1. [ ] ThreadLocal: 较低
2. [ ] Synchronized: 较高（需要加锁）

### 2.3 场景不同
1. [ ] ThreadLocal:每个线程需要独立的变量
2. [ ] Synchronized: 多线程访问共享变量


## 三 ThreadLocal 的底层实现原理？
### 3.1 检查ThreadLocalMap是否存在
✅ 当线程第一次调用 ThreadLocal 的 set() 或 get() 方法时，会检查当前线程（Thread 对象）中是否已存在一个 ThreadLocalMap  
✅ 如果没有，则准备创建  

### 3.2 不存在ThreadLocalMap则创建
✅ 初始化一个默认长度为16的哈希表(Entry数组)  
✅ Entry继承WeakReference<ThreadLocal>对象  
✅ Entry的key就是ThreadLocal，value是一个Object对象，用户存储的实际对象  

### 3.3 对ThreadLocal变量计算hashcode
✅ 对ThreadLocal变量计算hashcode, 然后根据hashcode计算落在哪一个桶内  
✅ 槽上Entry为空, 直接放进去  
✅ 如果不为空，则判断key是否相同: key相同则更新; key不相同则利用线性探测，往下找下一个位置放进去  

### 3.4 GC回收
✅ ThreadLocalMap 的 Entry 使用 WeakReference 作为 key，意味着当 ThreadLocal 实例没有外部强引用时，GC 会回收这个 key（即 WeakReference.get() 返回 null）    
✅ 此时，value 仍然存在，可能导致内存泄漏  
✅ ThreadLocalMap 会在后续操作中清理这些 key 为 null 的 Entry（称为 stale entries），以释放其 value，防止泄漏  

## 四 为什么是Entry继承WeakReference<ThreadLocal>对象？而不是Entry包含一个WeakReference<ThreadLocal>对象作为key呢?
### 4.1 为什么要使用弱引用
#### 4.1.1 我们先考虑正常情况: 线程执行完后会被销毁
✅ 那么Thread线程持有的ThreadLocalMap也将会被GC回收，因为Thread已经没有引用指向它了  
✅ 因此，ThreadLocalMap中Entry数组, 以及Entry中的key和value都会被回收  
✅ 如果只是站在这个角度看，Entry没有必要继承WeakReference<ThreadLocal>  

#### 4.1.2 我们先考虑正常情况: 线程执行完后不被销毁

✅ 当线程池中的线程，在执行完上一个任务后，可能并不会销毁，那么这个时候，Thread对象也不会被GC回收  
✅ 但是如果ThreadLocal是局部变量，在上一轮任务执行完毕后，感觉就没有强引用  
✅ 但是，Thread对象也不会被GC回收, ThreadLocalMap就会继续持有对Entry的key和value的强引用，导致ThreadLocal应该被回收，但是没有被回收  
**如以下代码举例子:**  
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalLeakExample {
    private static final ExecutorService executor = Executors.newFixedThreadPool(1);

    public static void main(String[] args) {
        for (int i = 0; i < 10000; i++) {
            executor.execute(() -> {
                // 方法局部变量：执行完就没有强引用指向它了！
                ThreadLocal<byte[]> local = new ThreadLocal<>();
                local.set(new byte[1024 * 1024]); // 1MB

                // 没有调用 local.remove()，留下 value 对象
                // 此时 local 对象会被置空，但 ThreadLocalMap 中的 Entry 仍保留 value

                // 关键：ThreadLocal 实例生命周期已结束，但线程还活着
            });
        }
    }
}

```

如果我们希望已经在业务上没有引用关系的对象应该被回收，而不是继续被强引用，最终可能会产生内存溢出的问题。那么就针对Entry的key采用弱引用技术  
这样，这里只需要判断在外部有没有强引用指向ThreadLocal, 如果没有这个ThreadLocal就会被当做弱引用处理  
GC在回收的时候，就会把它置为null  

🎯 **结论:**   
❗️ 避免ThreadLocal在外面已经是没有强引用，但是在ThreadLocalMap中还被当做强引用对象，从而，不会被GC回收，可能导致内存泄漏的风险  
❗️ 故这里ThreadLocal使用弱引用技术



### 4.2 为什么是Entry继承WeakReference<ThreadLocal>对象
我们已经明确知道为什么这里ThreadLocal需要使用WeakReference弱引用技术，而不是持有一个WeakReference<ThreadLocal> key的属性呢?
类似这样:   
```java
// 实际设计
class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
}

// 而不是这样：
class Entry {
    WeakReference<ThreadLocal<?>> key;
    Object value;
}
```

#### 4.2.1 JVM 的 GC 如何处理弱引用对象
弱引用通常可以与一个引用队列（ReferenceQueue）联合使用。当弱引用所指向的对象被垃圾回收后，这个弱引用本身会被加入到与之关联的引用队列中  
JVM 扫描这个 queue，就能发现哪些对象可以清理了:  
```
ReferenceQueue<Object> queue = new ReferenceQueue<>();
WeakReference<Object> weakRef = new WeakReference<>(new Object(), queue);

System.gc(); // 触发垃圾回收

// 检查引用队列
Reference<? extends Object> ref = queue.poll();
if (ref != null) {
    System.out.println("弱引用对象已被回收");
}
```

#### 4.2.2 继承WeakReference vs 包装 WeakReference
✅ 当检测到ThreadLocal可以被回收的时候，JVM 会把这个 Entry 对象本身加入到 ReferenceQueue 里  
✅ 这样 ThreadLocalMap 能知道： 这个 Entry 的 key 已经失效了，我可以清理这整条记录（key=null 时把 value 也清掉）  
✅ 这就是继承的优势：你一旦继承了 WeakReference<T>，JVM 会在回收 T 的时候通知你  

**如果这样设计:**  
```java
class Entry {
    WeakReference<ThreadLocal<?>> key;
    Object value;
}
```  
✅ Entry 本身不是一个 Reference 对象  
✅ GC 回收 ThreadLocal 的时候，并不会把 Entry 加入 ReferenceQueue  
✅ 你就没法高效知道这个 Entry 的 key 是不是已经失效了  
✅ 每次清理都只能遍历 map，看哪些 entry.key.get() == null。 这会导致： 清理效率低 和 无法配合 JVM 的 ReferenceQueue 实现“自动清理策略



🎯 **总结：为什么 Entry 要继承 WeakReference**
✅ 高效自动清理: GC 后的 Entry	JVM 只会把 WeakReference 实例放入 ReferenceQueue，不能放普通对象  
✅ 简化清理逻辑: Entry 本身就是弱引用，失效后直接作为垃圾引用对象进入清理队列  
✅ 节省内存: 减少额外包装类、避免 Entry 持有多余的 WeakReference 成员  

🎯 **结尾总结记忆点**
✅ Entry 继承 WeakReference<ThreadLocal>，是为了让 JVM 能通过 ReferenceQueue 精确跟踪哪些 ThreadLocal 被回收，从而高效地清除 ThreadLocalMap 中的 stale entry  
✅ 如果只是 Entry 包含一个 WeakReference 字段，就无法被 GC 跟踪，也无法自动进入 ReferenceQueue，会导致内存泄漏清理不及时或效率低下  

## 五 ThreadLocal 会导致内存泄漏吗？

### 5.1 结论
✅ ThreadLocal本身不会造成内存泄漏  
✅ 但如果你没有手动调用 remove() 方法，并且 ThreadLocal 实例被 GC 回收了，它在 ThreadLocalMap 中的 Entry.key 变为 null，而 Entry.value 仍然强引用存在，就可能发生内存泄漏


### 5.2 内存泄漏场景
#### 5.2.1 线程池 + 忘记调用 remove()
✅ 线程池中，线程是复用的  
✅ 如果你将一些带状态的对象(如数据库连接、用户上下文、敏感数据)放入 ThreadLocal，却没有清除, 调用 remove()  
✅ 那么这些对象就会一直存在于线程的 ThreadLocalMap 中，无法释放  

🎯 **示例代码:**  
```java
public class ThreadLocalLeakDemo {
    private static final ExecutorService executor = Executors.newFixedThreadPool(1);
    private static final ThreadLocal<byte[]> local = new ThreadLocal<>();

    public static void main(String[] args) {
        for (int i = 0; i < 1000; i++) {
            executor.submit(() -> {
                // 每次都分配 10MB 内存
                local.set(new byte[10 * 1024 * 1024]);
                // ❌ 没有调用 local.remove()
                // 模拟业务逻辑处理
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
    }
}
```
🎯 **解释：**
* 由于线程池只用了 一个线程（fixedThreadPool(1)），这个线程会重复使用；
* 每一次提交任务，都会在 ThreadLocalMap 中添加新的 Entry（如果 key 被 GC 了，旧的 Entry.key 为 null，value 还在）；
* 而你又没有手动 remove()，所以内存不断增长；
* 最终导致堆内存溢出（OutOfMemoryError），即使任务结束，线程还活着，value 没法回收

#### 5.2.2 如何避免内存泄漏？
✅ 使用完 ThreadLocal 一定要调用 remove()
✅ 避免在静态变量中持有 ThreadLocal 引用  
✅ 谨慎使用线程池 + ThreadLocal  

## 六 为什么 ThreadLocalMap 的 key 是 WeakReference？value不是呢？
### 6.1 为什么 key（即 ThreadLocal）是弱引用？
如果 ThreadLocalMap 的 key 是强引用，线程就永远持有 ThreadLocal 实例，即使程序员不再使用它，它也无法被 GC 回收 → 内存泄漏。  

**使用 WeakReference<ThreadLocal> 后：**
如果业务代码中没有其他强引用指向这个 ThreadLocal 对象， 它就可以被 GC 回收， 被回收后，Entry.key == null，ThreadLocalMap 会清理这些 “无效” 的键值对  


### 6.2 为什么 value 是强引用？
value 通常是你真正想存在线程局部变量里的对象（比如：用户信息、数据库连接、缓存等）  
如果把 value 也设置为弱引用，那么：你刚放进去的数据，可能下一次 GC 就被回收了， 程序逻辑中还没用完它，value 却变 null，功能就出错了。  
**目的：**保证业务数据在使用期间不被 GC 回收
