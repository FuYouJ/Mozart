## JDK动态代理的实现原理和失效的场景

> SpringAOP有两种CgLib和JdkProxy实现，CgLib适用于没有接口的类，JdkProxy适用于实现了接口的类。这里解析AOP的实现原理是为了更直观的解释某些场景下AOP失效的原因。比如事务，日志统计，请求拦截和过滤。

## 代码准备

因为JDKProxy只能用于有接口的类。这里定义一个接口模拟业务类，定义了两个注解。

```Java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
//打印日志
public @interface LogPrint {
}


//自动开启事务
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface Transaction {
}
public interface IService {

    /**
     * 执行事务的函数
     * System.out.println("update db_test set name = '付有杰' where id = 1");
     */
    @Transaction
    void transactionFunc();

    /**
     * 执行日志统计的函数
     *  System.out.println("-----------logFunc-----------");
     */
    @LogPrint
    void logFunc();

    /**
     * 事务函数调日志
     *  System.out.println("update db_test set name = '付有杰' where id = 1");
     *  this.logFunc();
     */
    @Transaction
    void transaction2Log();

    /**
     *  日志调用事务函数
     *  System.out.println("-----------logFunc-----------");
     *  this.transactionFunc();
     */
    @LogPrint
    void log2Transaction();
}
public class ServiceImpl implements IService {


    @Override
    public void transactionFunc() {

        System.out.println("update db_test set name = '付有杰' where id = 1");
    }


    @Override
    public void logFunc() {

        System.out.println("-----------logFunc-----------");
    }


    @Override
    public void transaction2Log() {

        System.out.println("update db_test set name = '付有杰' where id = 1");

        this.logFunc();
    }

    @Override
    public void log2Transaction() {

        System.out.println("-----------logFunc-----------");

        this.transactionFunc();
    }
}
```

## 实现AOP

利用JDKProxy代理实现AOP比如要实现一个`InvocationHandler`,在里面写入自己的AOP逻辑。

```Java
public class AOPHandler implements InvocationHandler {
    private final Object object;

    public AOPHandler(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        LogPrint logPrint = method.getAnnotation(LogPrint.class);
        Transaction transaction = method.getAnnotation(Transaction.class);

        if (logPrint != null) {
            System.out.println("【logBefore】");
        }
        if (transaction != null) {
            System.out.println("【start transaction;】");
        }

        //执行真正的原始方法
        Object result = method.invoke(object, args);

        if (logPrint != null) {
            System.out.println("【logAfter】");
        }
        if (transaction != null) {
            System.out.println("【transaction commit;】");
        }

        
        return result;
    }
}
```

## 将AOP增强过的类放入到容器

```Java
//模拟的spring的容器
private static Map<Class<?>, Object> springApplicationContext() {

    Map<Class<?>, Object> container = new HashMap<>();
    //原始对象
    ServiceImpl service = new ServiceImpl();

    ClassLoader classLoader = service.getClass().getClassLoader();
    Class<?>[] interfaces = service.getClass().getInterfaces();
    //JDK提供的构造代理类的静态方法
    Object proxyInstance = Proxy.newProxyInstance(
    //类加载器
    classLoader,
    //这个类实现的接口们 
    interfaces, 
    //handler
    new AOPHandler(service));

    //把代理类放进容器
    container.put(IService.class, proxyInstance);

    return container;
}
```

## 测试复现问题

### **测试事务方法**

```Java
@Override
@Transaction
public void transactionFunc() {

    System.out.println("update db_test set name = '付有杰' where id = 1");
}

//增强的aop逻辑
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    LogPrint logPrint = method.getAnnotation(LogPrint.class);
    Transaction transaction = method.getAnnotation(Transaction.class);

    if (logPrint != null) {
        System.out.println("【logBefore】");
    }
    if (transaction != null) {
        System.out.println("【start transaction;】");
    }

    //执行真正的原始方法
    Object result = method.invoke(object, args);

    if (logPrint != null) {
        System.out.println("【logAfter】");
    }
    if (transaction != null) {
        System.out.println("【transaction commit;】");
    }


    return result;
}
```

执行方法

```Java
 public static void main(String[] args) {

     Map<Class<?>, Object> springApplicationContext = springApplicationContext();
     //模拟从spring容器拿到的被代理增强的service
     IService service = (IService) springApplicationContext.get(IService.class);
     //只有事务注解
     service.transactionFunc();
   }
```

执行结果

![img](http://www.xiewenshi.fans/blogimg/java/jdkproxy1.png)

### 测试日志记录

**原始逻辑**

```Java
@Override
@LogPrint
public void logFunc() {
    System.out.println("-----------logFunc-----------");
}
```

**执行结果**

![img](http://www.xiewenshi.fans/blogimg/java/jdkproxy2.png)

### 失效场景1

原始逻辑

```Java
@Override
@Transaction
public void transaction2Log() {

    System.out.println("update db_test set name = '付有杰' where id = 1");

    this.logFunc();
}
```

执行结果

![img](http://www.xiewenshi.fans/blogimg/java/jdkproxy3.png)

### 失效场景2

原始逻辑

```Java
@Override
@LogPrint
public void log2Transaction() {

    System.out.println("-----------logFunc-----------");

    this.transactionFunc();
}
```

执行结果

![img](http://www.xiewenshi.fans/blogimg/java/jdkproxy4.png)

## 分析原因

JDKProxy实现动态代理类每次执行方法时委托了AOPHandler执行，Handler执行完毕最外层的AOP逻辑再委托了真正的原始类执行，真正的原始类里使用this指针或者调用自己的方法，就不会像前面说的再绕一圈执行AOP的逻辑。

![img](http://www.xiewenshi.fans/blogimg/java/jdkproxy5.png)

**以下是反编译看到的自动生成的代码是什么样子，可以看到实际上就是调用InvocationHandler去调用真实类**

```Java
public final class ServiceImplProxy extends Proxy implements IService {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m6;
    private static Method m4;
    private static Method m5;
    private static Method m0;

    public ServiceImplProxy(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void logFunc() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void transaction2Log() throws  {
        try {
            super.h.invoke(this, m6, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void log2Transaction() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void transactionFunc() throws  {
        try {
            super.h.invoke(this, m5, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("demo.proxy.service.IService").getMethod("logFunc");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m6 = Class.forName("demo.proxy.service.IService").getMethod("transaction2Log");
            m4 = Class.forName("demo.proxy.service.IService").getMethod("log2Transaction");
            m5 = Class.forName("demo.proxy.service.IService").getMethod("transactionFunc");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    LogPrint logPrint = method.getAnnotation(LogPrint.class);
    Transaction transaction = method.getAnnotation(Transaction.class);

    if (logPrint != null) {
        System.out.println("【logBefore】");
    }
    if (transaction != null) {
        System.out.println("【start transaction;】");
    }

    //执行真正的原始方法
    Object result = method.invoke(object, args);

    if (logPrint != null) {
        System.out.println("【logAfter】");
    }
    if (transaction != null) {
        System.out.println("【transaction commit;】");
    }


    return result;
}
```

## 总结

分析一下几种情况

```Java
class Test{
    //A的事务会失效吗
    @Transaction
    void A(){
         this.B();
    }
    
    //单独执行B会有事务吗
    void B();
}

class Test{
    //这么调用
   void A(){
         this.B();
    }
    
    //B的事务会失效吗
    @Transaction
    void B();
}

@Transaction
class Test{
    
    //A B的事务会失效吗
    void A(){
         this.B();
    }
    
    void B();
}


class Test{
    
    @Transaction
    void A(){
    //在这里的b会异步执行吗 = 同步
         this.B();
    }
    
    @Async
    void B();
}
```

在同一个类里调用自身方法时，被调用方的AOP会失效。

解决办法

- 拆分方法到其他的类
- 调用自身方法时，走容器代理，不走自身。
  - 在启动配置类中加上@EnableAspectJAutoProxy(exposeProxy = true)注解
  -  ((XXXService)AopContext.currentProxy()).DoSomeThing();
  - ![img](http://www.xiewenshi.fans/blogimg/java/jdkproxy6.png)
- 特殊情况：事务的传递性：事务调事务虽然内部事务失效了，但是依赖于外部事务，也就还是有事务的效果。

## 加餐

多个AOP的执行顺序

![img](http://www.xiewenshi.fans/blogimg/java/mutiAop1.png)

![img](http://www.xiewenshi.fans/blogimg/java/multiaop2.png)

```Java
 private static Map<Class<?>, Object> springApplicationContext() {

        Map<Class<?>, Object> container = new HashMap<>();
        //原始对象
        ServiceImpl service = new ServiceImpl();

        ClassLoader classLoader = service.getClass().getClassLoader();
        Class<?>[] interfaces = service.getClass().getInterfaces();
        //JDK提供的构造代理类的静态方法
        Object proxyInstance = Proxy.newProxyInstance(
                //类加载器
                classLoader,
                //这个类实现的接口们
                interfaces,
                //handler
                new AOPHandler(service));

       proxyInstance =  Proxy.newProxyInstance(classLoader,
       interfaces,
       new AOPHandler2(proxyInstance));

        //把代理类放进容器
        container.put(IService.class, proxyInstance);

        return container;
    }
```

运行

![img](http://www.xiewenshi.fans/blogimg/java/multiAop3.png)