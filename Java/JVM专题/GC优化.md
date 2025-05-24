## 一 什么是Full GC? 发生Full GC原因有哪些? 遇到Full GC我们应该怎么办?

### 1.1 什么是Full GC?
👉 Java虚拟机会暂停暂停所有用户线程(STW)，然后一次性对整个堆，包括年轻代、老年代和元空间进行一次垃圾回收, 时间开销可能非常大  
👉 与 Minor GC（仅针对年轻代）相比，Full GC 的开销通常更大，耗时更长，因此频繁发生 Full GC 会对系统性能造成严重影响  


### 1.2 发生Full GC原因有哪些?
#### 1.2.1 老年代空间不足
👉 当年轻代中的对象晋升到老年代时，如果老年代空间不足，则会触发 Full GC。这种情况在高并发场景下尤为常见，大量对象短时间内从年轻代晋升到老年代，导致老年代迅速填满  

#### 1.2.2 大对象分配老年代失败
👉 如果程序创建了过大的对象（超过年轻代的大小），这些对象会直接分配到老年代。如果老年代没有足够的空间容纳这些大对象，就会触发 Full GC  

#### 1.2.3 Metaspace元空间不足
👉 在 Java 8 及之后的版本中，永久代被元空间替代。如果元空间不足以容纳新的类信息或其他元数据，也会触发 Full GC  

#### 1.2.4 程序中显示调用System.gc()
👉 如果代码中显式调用了 System.gc() 方法，JVM 会尝试执行一次 Full GC。虽然可以通过 JVM 参数 -XX:+DisableExplicitGC 禁止这种行为，但最好避免直接调用 System.gc()  

### 1.3 Full GC 的影响
1. [ ] Stop The World（STW）：应用卡顿
2. [ ] GC耗时明显上升
3. [ ] 频繁 Full GC 是系统性能下降/内存泄漏的信号
4. [ ] 可能触发 OOM：如果 Full GC 后仍无法腾出空间


### 1.4 遇到Full GC我们应该怎么办?
#### 1.4.1 先观察GC日志
👉 -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log  
👉 看 Full GC 发生频率、时间、前后内存情况，是否伴随 Metaspace、老年代溢出  

#### 1.4.2 检查内存分配情况
👉 如果发现年轻代空间不足，调整年轻代的大小（-Xmn 参数）以减少对象晋升到老年代的速度  
👉 如果老年代空间不足，适当增加堆内存大小（-Xms 和 -Xmx 参数）  

#### 1.4.3 看是否可以优化代码逻辑
👉 避免创建过多的大对象，尽量将大对象拆分为小对象  
👉 减少对象的生命周期，让短生命周期的对象尽可能在 Minor GC 中被回收，而不是晋升到老年代   

#### 1.4.4 选择合适的垃圾回收器
👉 根据应用的特性选择合适的垃圾回收器，例如：
👉 对于低延迟需求的应用，可以使用 G1 或 ZGC。
👉 对于吞吐量优先的应用，可以使用 Parallel GC。
👉 调整垃圾回收器的相关参数，例如 CMS 的并发线程数或 G1 的区域大小 

#### 1.4.5 排查内存泄漏
👉 如果 Full GC 后老年代仍有大量存活对象，可能存在内存泄漏问题。
👉 使用工具（如 MAT、JProfiler 或 VisualVM）分析堆内存快照，定位内存泄漏的根源并修复问题 



## 二 Java 中如何避免频繁 GC？如何减少临时对象创建？
### 2.1 为什么频繁 GC 会成为问题
每次 GC 都会有性能损耗(尤其是 Full GC，会 STW)  
频繁 GC 是内存抖动的信号，可能引发系统抖动或 OOM    

### 2.2 Java 中如何避免频繁 GC
#### 2.2.1 减少临时对象的创建
##### 2.2.1.1 重用对象/工具类:
复用同一个对象,而不是每次都创建一个对象:  
```
// 不推荐: 每次循环都创建新对象
for (int i = 0; i < 10000; i++) {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
    sdf.format(new Date());
}

// 推荐：复用同一个对象
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
for (int i = 0; i < 10000; i++) {
    sdf.format(new Date());
}
```


##### 2.2.1.2 使用对象池
对于需要频繁创建和销毁的对象（如线程、数据库连接等），可以使用对象池技术来复用对象，避免频繁分配和回收内存  
常见的对象池库包括 Apache Commons Pool 和 HikariCP  

##### 2.2.1.3 使用基本数据类型代替包装类
包装类（如 Integer、Double）是对象，而基本数据类型（如 int、double）不是对象。尽量使用基本数据类型以减少对象的创建  
```
// 不推荐：使用包装类
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    list.add(Integer.valueOf(i)); // 每次都会创建新的 Integer 对象
}

// 推荐：使用基本数据类型
int[] array = new int[1000];
for (int i = 0; i < 1000; i++) {
    array[i] = i;
}
```

##### 2.2.1.4 使用 Stream API 和 Lambda 表达式
Java 8 引入了 Stream API 和 Lambda 表达式，可以更高效地处理集合数据，避免频繁创建临时对象   
```
// 不推荐：传统方式
List<String> filteredList = new ArrayList<>();
for (String item : list) {
    if (item.startsWith("A")) {
        filteredList.add(item);
    }
}

// 推荐：使用 Stream API
List<String> filteredList = list.stream()
   .filter(item -> item.startsWith("A"))
   .collect(Collectors.toList());
```

#### 2.2.2 合理配置 JVM 参数
JVM 的内存配置对 GC 的频率和性能有直接影响。通过调整 JVM 参数，可以有效减少 GC 的发生频率  
##### 2.2.2.1 调整新生代大小  
新生代（Young Generation）是临时对象的主要存储区域。如果新生代空间不足，会导致频繁的 Minor GC  
优化方式：通过 -Xmn 参数增大新生代的大小，降低 Minor GC 的频率   

##### 2.2.2.2 增大堆内存  
如果堆内存过小，老年代可能会快速填满，从而触发 Full GC  
优化方式：通过 -Xms 和 -Xmx 参数增大堆内存大小，减少 GC 的压力  

##### 2.2.2.3 选择合适的垃圾回收器
不同的垃圾回收器适用于不同的应用场景。例如：  
* G1 GC ：适合低延迟需求的应用。
* Parallel GC ：适合吞吐量优先的应用。
* ZGC ：适合超大规模内存的应用。
优化方式：根据应用需求选择合适的垃圾回收器，并调整相关参数

#### 2.2.3 优化对象的生命周期
尽量缩短对象的生命周期，使对象尽快被回收  
优化方式：避免将短生命周期对象存储到全局变量或静态变量中，以免延长其生命周期  
```
// 不推荐：延长对象生命周期
public static List<String> globalList = new ArrayList<>();

public void method() {
    for (int i = 0; i < 1000; i++) {
        globalList.add("data" + i);
    }
}

// 推荐：局部变量
public void method() {
    List<String> localList = new ArrayList<>();
    for (int i = 0; i < 1000; i++) {
        localList.add("data" + i);
    }
}
```

#### 2.2.4 避免内存泄漏
内存泄漏会导致老年代空间被占用，从而增加 Full GC 的频率。
优化方式：定期检查代码中的内存泄漏问题，例如未关闭的流、未释放的资源等  
```
// 不推荐：未关闭流
InputStream in = new FileInputStream("file.txt");

// 推荐：使用 try-with-resources 自动关闭流
try (InputStream in = new FileInputStream("file.txt")) {
    // 处理文件
} catch (IOException e) {
    e.printStackTrace();
}
```