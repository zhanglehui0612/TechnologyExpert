## 一 什么是 Spring AOP？它的作用是什么？
AOP是面向切面编程，将公共的一些非核心功能封装成切面，在不修改代码的情况下，应用到业务逻辑中  

## 二 AOP有哪些角色？切面、切点、连接点？它们之间有什么区别？
### 2.1 AOP有哪些角色
✅ 切面: 切面，指的是横切关注点模块化表示，一个切面可以定义多个切点和通知  
✅ 连接点: 需要进行增强逻辑的方法执行点，表示可插入通知的点  
✅ 切点: 定义哪些连接点需要被拦截  
✅ 通知: 要执行的功能，包括前置通知、后置通知，环绕通知等  
✅ 织入: 将切面中的通知应用到定义的切点上  

### 2.2 切面、切点、连接点 它们之间有什么区别？
#### 2.2.1 连接点
✅ 需要进行增强逻辑的方法执行点，表示可插入通知的点  
✅ 比如UserService.createUser() 方法的调用就是一个连接点  
✅ 比如UserService.updateUser() 方法的调用就是一个连接点  
```java
@Service
public class UserService {

    public void saveUser(String name) {
        // 模拟耗时
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        System.out.println("保存用户: " + name);
    }

    public void updateUser(String name) {
        // 模拟耗时
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        System.out.println("更新用户: " + name);
    }
}
```

#### 2.2.2 切点
✅ 定义哪些连接点需要被拦截，通过表达式配置  
```java
// ✅ 这就是【切点】：定义哪些连接点需要被拦截
@Pointcut("execution(* com.example.service..*(..))")
public void logPointcut() {
    // 方法体可以为空，只是一个标记
}
```

#### 2.2.3 切面
✅ 切面，指的是横切关注点模块化表示，一个切面可以定义多个切点和通知  
✅ 每一个切面需要添加@Aspect注解  
```java
@Aspect  // 表示这是一个切面类
@Component  // 让 Spring 扫描它
public class LogExecutionTimeAspect {

    // ✅ 这就是【切点】：定义哪些连接点需要被拦截
    @Pointcut("execution(* com.example.service..*(..))")
    public void logPointcut() {
        // 方法体可以为空，只是一个标记
    }

    // ✅ 织入: 将切面中的通知应用到定义的切点上
    @Around("logPointcut()")  // 引用上面的切点表达式
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {

        long start = System.currentTimeMillis();

        // ✅ 这就是【连接点】：具体执行的方法，比如 userService.saveUser()
        Object result = joinPoint.proceed();  // 执行目标方法

        long timeTaken = System.currentTimeMillis() - start;
        System.out.println("[AOP日志] " + joinPoint.getSignature() + " 执行耗时：" + timeTaken + "ms");

        return result;
    }
}
```


## 三 如何自定义一个切面？并应用在指定类/包？

## 四  Spring AOP 是如何实现代理的？
### 4.1 Spring AOP 是如何实现代理的？
🎯 **步骤 1：Spring 启动时解析切面**  
✅ 扫描 @Aspect 注解的类  
✅ 读取其中的通知(@Around、@Before)和切点(@Pointcut)  

🎯 **步骤 2：Spring 创建目标 Bean 时判断是否需要代理**  
✅ 当某个 Bean 正在被创建，Spring 会判断： 这个类的方法是否命中了某个切点表达式？  
✅ 如果命中，Spring 就会为这个 Bean 创建一个代理对象，就需要将这个实例的对象工厂放到三级缓存  


🎯 **步骤 3：创建代理对象**  
✅ 通过ObjectFactory.getObject的时候，会调用getEarlyBeanReference  
✅ 使用 ProxyFactory 创建代理对象，Bean实现了接口，采取JDK动态代理；没有实现接口通过CGLib动态代理  

🎯 **步骤 4：代理对象执行方法时，调用通知逻辑**  
✅ 代理对象执行方法时，调用通知逻辑

### 4.2 Spring AOP 什么时候使用 JDK 动态代理？什么时候用 CGLIB？
取决于Bean是否实现接口，如果有接口则走JDK 动态代理；没有接口则走CGLib动态代理  

## 五 JDK动态代理和CGLib动态代理比较? 
### 5.1 代理对象生成方式不同
✅ JDK动态代理: 基于接口实现，通过反射生成接口实现类  
✅ CGLib动态代理: 基于继承实现, 通过生成目标类的子类实现代理  

### 5.2 限制
✅ JDK动态代理: 只能代理接口  
✅ CGLib动态代理: 不能代理 final 类和 final 方法  

### 5.3 要求
✅ JDK动态代理: 目标类必须实现接口  
✅ CGLib动态代理: 目标类不需要实现接口  

## 六 AOP 对性能有影响吗？实际项目中有哪些场景要慎用 AOP？
### 6.1 AOP 对性能有影响吗
✅ 代理开销：Spring AOP 基于代理(JDK 动态代理或 CGLIB)，每次调用切点方法都会经过代理逻辑，增加额外的调用层  
✅ 反射调用: 部分场景中会涉及反射调用，性能略低于直接调用  
✅ 通知逻辑开销：切面中的通知(Advice)代码越复杂，执行时间越长，性能影响越大  

### 6.2 实际项目中有哪些场景要慎用 AOP?
🎯 **高性能要求的核心业务代码**  
频繁调用且性能敏感的方法，不建议使用复杂的 AOP 通知，避免额外开销  

🎯 **大规模循环或批量处理逻辑**  
在大量数据循环操作中使用 AOP，可能引起显著的性能瓶颈  

🎯 **递归调用场景**  
AOP 代理可能导致递归调用链条变复杂，增加调用深度和风险  

🎯 **大量切点匹配**  
切点表达式不精准，匹配范围过大，导致大量无关方法被代理，浪费性能  


## 七 如何避免 AOP 在并发场景下产生线程安全问题？
✅ 使用 ThreadLocal 存储线程隔离的数据  
✅ 避免使用实例变量存储状态

## 八 一个 Bean 被代理后，它的实际类型还是原始类型吗？instanceof 会成立吗？
**结论:**  
✅ 一个 Bean 被代理后，它的实际类型不是原始类型，而是代理类类型  
✅ 但是instanceof 判断 原始类型成立

**JDK 动态代理:**  
✅ 代理类是接口的实现类，不是目标类本身  
✅ 虽然代理对象 proxy 的类型 ≠ 目标类类型   
✅ 但因为代理类实现了接口，proxy instanceof 接口 成立，而 proxy instanceof 目标类 不成立  

**CGLIB 动态代理:**  
✅ 代理类是目标类的子类  
✅ 所以，proxy instanceof 目标类 是成立的  

## 九 Spring AOP 失效的场景有哪些？为什么会失效？如何解决？
✅ 自身方法调用: 调用没有经过代理对象  
✅ 方法非public: Spring AOP 只拦截 public 方法  
✅ 类未被Spring管理:	AOP 只拦截由 Spring 容器托管的 Bean  
✅ 被final、static修饰的方法:	CGLIB 无法代理 final/static 方法  
✅ 使用JDK动态代理但是未通过接口调用: JDK 代理只对接口有效  