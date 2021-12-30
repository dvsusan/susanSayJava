
![](https://pic.imgdb.cn/item/61151caf5132923bf833b7a6.jpg)
你有几年没回老家了？

我有三年。

今年怕是又回不去了，有些想家了。。。

你呢？

## 前言
前几篇文章本打算写spring aop的，但是强忍着没有写（旁白：也有可能是没想好该怎么写😝），就是为了今天整个专题，因为它是spring中最核心的技术之一，实在太重要了。

关于spring aop的文章网上一搜一大堆，但我想写点不一样的东西，尝试一种全新的写作风格，希望您会喜欢。

![](https://pic.imgdb.cn/item/61151ce25132923bf834219e.jpg)

## 从实战出发
很多文章讲spring aop的时候，一开始就整一堆概念，等我们看得差不多要晕的时候，才真正进入主题。。。

我却相反，没错，先从实战出发。

在spring aop还没出现之前，想要在目标方法之前先后加上日志打印的功能，我们一般是这样做的：
```java
@Service
public class TestService {

    public void doSomething1() {
        beforeLog();
        System.out.println("==doSomething1==");
        afterLog();
    }

    public void doSomething2() {
        beforeLog();
        System.out.println("==doSomething1==");
        afterLog();
    }

    public void doSomething3() {
        beforeLog();
        System.out.println("==doSomething1==");
        afterLog();
    }

    public void beforeLog() {
        System.out.println("打印请求日志");
    }

    public void afterLog() {
        System.out.println("打印响应日志");
    }
}
```
如果加了新doSomethingXXX方法，就需要在新方法前后手动加beforeLog和afterLog方法。

原本相安无事的，但长此以往，总有会出现几个刺头青。

刺头青A说：每加一个新方法，都需要加两行重复的代码，是不是很麻烦？

刺头青B说：业务代码和公共代码是不是耦合在一起了？

刺头青C说：如果有几千个类中加了公共代码，而有一天我需要删除，是不是要疯了？

spring大师们说：我们提供一套spring的aop机制，你们可以闭嘴了。

下面看看用spring aop（偷偷说一句，还用了aspectj）是如何打印日志的：
```java
@Service
public class TestService {

    public void doSomething1() {
        System.out.println("==doSomething1==");
    }

    public void doSomething2() {
        System.out.println("==doSomething1==");
    }

    public void doSomething3() {
        System.out.println("==doSomething1==");
    }
}
@Component
@Aspect
public class LogAspect {

    @Pointcut("execution(public * com.sue.cache.service.*.*(..))")
    public void pointcut() {
    }

    @Before("pointcut()")
    public void beforeLog() {
        System.out.println("打印请求日志");
    }

    @After("pointcut()")
    public void afterLog() {
        System.out.println("打印响应日志");
    }
}
```
增加了LogAspect类，在类上加了@Aspect注解。先在类中使用@Pointcut注解定义了pointcut方法，然后将beforeLog和afterLog方法移到这个类中，分别加上@Before和@After注解。

改造后，业务方法在TestService类中，而公共方法在LogAspect类中，是分离的。如果要新加一个业务方法，直接加就好，LogAspect类不用改任何代码，新加的业务方法就自动拥有打印日志的功能，是不是很神奇？

![](https://pic.imgdb.cn/item/61151d1d5132923bf8349bc1.jpg)

spring aop其实是一种横切的思想，通过动态代理技术将公共代码织入到业务方法中。

这里出于5毛钱的友情，有必要温馨提醒一下。aop是一种思想，不是spring独有的，目前市面上比较出名的有：

- aspectj
- spring aop
- jboss aop

我们现在主流的做法是将spring aop和aspectj结合使用，spring借鉴了AspectJ的切面，以提供注解驱动的AOP。
此时，一个黑影一闪而过。

刺头青D问：你说的“横切”，“动态代理”，“织入” 是什么东东？

## 几个重要的概念
根据上面spring aop的代码，用一张图聊聊几个重要的概念：![](https://pic.imgdb.cn/item/61151d385132923bf834d3ef.jpg)

- 连接点（Joinpoint) 程序执行的某个特定位置，如某个方法调用前，调用后，方法抛出异常后，这些代码中的特定点称为连接点。
- 切点（Pointcut） 每个程序的连接点有多个，如何定位到某个感兴趣的连接点，就需要通过切点来定位。
- 通知（Advice） 增强是织入到目标类连接点上的一段程序代码。
- 切面（Aspect） 切面由切点和通知组成，它既包括了横切逻辑的定义，也包括了连接点的定义，SpringAOP就是将切面所定义的横切逻辑织入到切面所制定的连接点中。
- 目标对象（Target） 需要被增强的业务对象
- 代理类（Proxy） 一个类被AOP织入增强后，就产生了一个代理类。
- 织入（Weaving） 织入就是将增强添加到对目标类具体连接点上的过程。

还是那个刺头青D说（旁边：这位仁兄比较好学）：spring aop概念弄明白了，挺简单的。@Pointcut注解的execution表达式刚刚看得我一脸懵逼，可以再说说吗，我请你吃饭？

## 切入点表达式
@Pointcut注解的execution切入点表达，看似简单，里面还是有些内容的。为了更直观一些，还是用张图来总结一下：
[](https://pic.imgdb.cn/item/61151d6c5132923bf83543d7.jpg)

该表达式的含义是：匹配访问权限是public，任意返回值，包名为：com.sue.cache.service，下面的所有类所有方法和所有参数类型。图中所有用*表示，比如图中类名用.*表示的是所有类。如果具体匹配某个类，比如：TestService，则表达式可以换成：
```java
@Pointcut("execution(public * com.sue.cache.service.TestService.*(..))")
```
其实spring支持9种表达式，execution只是其中一种。

[](https://pic.imgdb.cn/item/61151d8e5132923bf8358be3.jpg)

## 有哪些入口？
先说说我为什么会问这样一个问题？

spring aop有哪些入口？说人话就是在问：spring中有哪些场景需要调用aop生成代理对象，难道你不好奇吗？

### 入口1
AbstractAutowireCapableBeanFactory类的createBean方法中，有这样一段代码：
[](https://pic.imgdb.cn/item/61151db75132923bf835e3f4.jpg)

它通过BeanPostProcessor提供了一个生成代理对象的机会。具体逻辑在AbstractAutoProxyCreator类的postProcessBeforeInstantiation方法中：

[](https://pic.imgdb.cn/item/61151de95132923bf8365523.jpg)

说白了，需要实现TargetSource才有可能会生成代理对象。该接口是对Target目标对象的封装，通过该接口可以获取到目标对象的实例。


不出意外，这时，又会冒出一个黑影。

刺头青F说：这里生成代理对象有什么用呢？

有时我们想自己控制bean的创建和初始化，而不需要通过spring容器，这时就可以通过实现TargetSource满足要求。只是创建单纯的实例还好，如果我们想使用代理该怎么办呢？这时候，入口1的作用就体现出来了。

## 入口2
AbstractAutowireCapableBeanFactory类的doCreateBean方法中，有这样一段代码：

[](https://pic.imgdb.cn/item/61151e095132923bf83696f2.jpg)

它主要作用是为了解决对象的循环依赖问题，核心思路是提前暴露singletonFactory到缓存中。


通过getEarlyBeanReference方法生成代理对象：

[](https://pic.imgdb.cn/item/61151e2f5132923bf836e50d.jpg)

它又会调用wrapIfNecessary方法：

[](https://pic.imgdb.cn/item/61151e455132923bf8371555.jpg)

这里有你想看到的生成代理的逻辑。


这时。。。。，你猜错了，黑影去吃饭了。。。

### 入口3
AbstractAutowireCapableBeanFactory类的initializeBean方法中，有这样一段代码：
[](https://pic.imgdb.cn/item/61151e635132923bf837513c.jpg)

它会调用到AbstractAutoProxyCreator类postProcessAfterInitialization方法：
[](https://pic.imgdb.cn/item/61151e7c5132923bf837885d.jpg)

该方法中能看到我们熟悉的面孔：wrapIfNecessary方法。从上面得知该方法里面包含了真正生成代理对象的逻辑。

这个入口，是为了给普通bean能够生成代理用的，是spring最常见并且使用最多的入口。

下面为了加深印象，用一张图总结一下：
[](https://pic.imgdb.cn/item/61151e945132923bf837bb24.jpg)

## jdk动态代理 vs cglib
我猜你们对jdk动态代理和cglib是知道的（即使猜错了也不会少块肉😃），但为了照顾一下新朋友，还是有必要把这两种生成代理的方式拿出来说说。

### jdk动态代理
jdk动态代理是通过反射技术实现的，生成代理的代码如下：
```java
public interface IUser {
    void add();
}

public class User implements IUser{
    @Override
    public void add() {
        System.out.println("===add===");
    }
}

public class JdkProxy implements InvocationHandler {

    private Object target;

    public Object getProxy(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(this.getClass().getClassLoader(),target.getClass().getInterfaces(),this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target,args);
        after();
        return result;
    }

    private void before() {
        System.out.println("===before===");
    }

    private void after() {
        System.out.println("===after===");
    }
}

public class Test {
    public static void main(String[] args) {
        User user = new User();
        JdkProxy jdkProxy = new JdkProxy();
        IUser proxy = (IUser)jdkProxy.getProxy(user);
        proxy.add();
    }
}
```
首先要定义一个接口IUser，然后定义接口实现类User，再定义类JdkProxy实现InvocationHandler接口，重写invoke方法，该方法中实现额外的逻辑。当然，别忘了在getProxy方法中，用Proxy.newProxyInstance方法创建一个代理对象。

jdk动态代理三个要素：

- 定义一个接口
- 实现InvocationHandler接口
- 使用Proxy创建代理对象

### cglib
cglib底层是通过asm字节码技术实现的，生成代理的代码如下：
```java
public class User {
    public void add() {
        System.out.println("===add===");
    }
}

public class CglibProxy implements MethodInterceptor {

    private Object target;

    public Object getProxy(Object target) {
        this.target = target;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        Object result = method.invoke(target,objects);
        after();
        return result;
    }

    private void before() {
        System.out.println("===before===");
    }

    private void after() {
        System.out.println("===after===");
    }
}

public class Test {
    public static void main(String[] args) {
        User user = new User();
        CglibProxy cglibProxy = new CglibProxy();
        IUser proxy = (IUser)cglibProxy.getProxy(user);
        proxy.add();
    }
}
```
这里不需要定义接口，直接定义目标类User，然后实现MethodInterceptor接口，重写intercept方法，该方法中实现额外的逻辑。当然，别忘了在getProxy方法中，通过Enhancer创建代理对象。

cglib两个要素：

- 实现MethodInterceptor接口
- 使用Enhancer创建代理对象

### spring中如何用的？
DefaultAopProxyFactory类的createAopProxy方法中，有这样一段代码：

[](https://pic.imgdb.cn/item/61151f815132923bf839a40e.jpg)

它里面包含：
- JdkDynamicAopProxy jdk动态代理生成类
- ObjenesisCglibAopProxy cglib代理生成类
- JdkDynamicAopProxy类的invoke方法生成的代理对象。而ObjenesisCglibAopProxy类的父类：CglibAopProxy，它的getProxy方法生成的代理对象。

### 哪个更好？
我猜，不是刺头青，是你，可能会来自灵魂深处的一问：jdk动态代理和cglib哪个更好？

其实这个问题没有标准答案，要看具体的业务场景：

1. 没有定义接口，只能使用cglib，不说它好不行。
2. 定义了接口，需要创建单例或少量对象，调用多次时，可以使用jdk动态代理，因为它创建时更耗时，但调用时速度更快。
3. 定义了接口，需要创建多个对象时，可以使用cglib，因为它创建速度更快。

> 随着jdk版本不断迭代更新，jdk动态代理创建耗时不断被优化，8以上的版本中，跟cglib已经差不多。所以spring官方默认推荐使用jdk动态代理，因为它调用速度更快。


出于人道主义关怀，免费赠送一条有用经验：如果要强制使用cglib，可以通过以下两种方式：

- spring.aop.proxy-target-class=true
- @EnableAspectJAutoProxy(proxyTargetClass = true)

## 五种通知
spring默认提供了五种通知：

[](https://pic.imgdb.cn/item/61151ff75132923bf83a9172.jpg)

按照国际惯例，不，按照我个人习惯，先看看他们是怎么用的。

### 前置通知
该通知在方法执行之前执行，只需在公共方法上加@Before注解，就能定义前置通知：
```kava
@Before("pointcut()")
public void beforeLog(JoinPoint joinPoint) {
    System.out.println("打印请求日志");
}
```

### 后置通知
该通知在方法执行之后执行，只需在公共方法上加@After注解，就能定义后置通知：
```java
@After("pointcut()")
public void afterLog(JoinPoint joinPoint) {
    System.out.println("打印响应日志");
}
```

### 环绕通知
该通知在方法执行前后执行，只需在公共方法上加@Round注解，就能定义环绕通知：
```java
@Around("pointcut()")
public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
    System.out.println("打印请求日志");
    Object result = joinPoint.proceed();
    System.out.println("打印响应日志");
    return result;
}
```

### 结果通知
该通知在方法结束后执行，能够获取方法返回结果，只需在公共方法上加@AfterReturning注解，就能定义结果通知：
```java
@AfterReturning(pointcut = "pointcut()",returning = "retVal")
public void afterReturning(JoinPoint joinPoint, Object retVal) {
    System.out.println("获取结果："+retVal);
}
```

### 异常通知
该通知在方法抛出异常之后执行，只需在公共方法上加@AfterThrowing注解，就能定义异常通知：
```java
@AfterThrowing(pointcut = "pointcut()", throwing = "e")
public void afterThrowing(JoinPoint joinPoint, Throwable e) {
    System.out.println("异常："+e);
}
```
spring aop给这五种通知，分别分配了一个xxxAdvice类。在ReflectiveAspectJAdvisorFactory类的getAdvice方法中可以看得到：

[](https://pic.imgdb.cn/item/611520415132923bf83b2a9b.jpg)

下面用一张图总结一下对应关系：

[](https://pic.imgdb.cn/item/611520555132923bf83b562b.jpg)

这五种xxxAdvice类都实现了Advice接口，但是有些差异。

下面三个xxxAdvice类实现了MethodInterceptor接口：
[](https://pic.imgdb.cn/item/611520735132923bf83b98e2.jpg)

而另外两个类：AspectJMethodBeforeAdvice 和 AspectJAfterReturningAdvice 没有实现上面的接口，这是为什么？

（这里留点悬念，后面的文章会揭晓谜题，敬请期待。）

一个猝不及防，依然是那个刺头青D，放下碗冲过来问了句：这五种通知的执行顺序是怎么样的？

#### 单个切面正常情况
[](https://pic.imgdb.cn/item/6115209c5132923bf83becd0.jpg)

#### 单个切面异常情况
[](https://pic.imgdb.cn/item/611520c05132923bf83c37a7.jpg)

#### 多个切面正常情况
[](https://pic.imgdb.cn/item/611520d15132923bf83c5906.jpg)

#### 多个切面异常情况
[](https://pic.imgdb.cn/item/611520e25132923bf83c7fe4.jpg)


> 当有多有切面时，按照可以通过@Order(n)指定执行顺序，n值越小越先执行。


## 为什么使用链式调用？
这个问题没人问，是我自己想聊聊（旁白：因为我长得帅，有点自恋了）。

先看看spring是如何使用链式调用的，在ReflectiveMethodInvocation的proceed方法中，有这样一段代码：
[](https://pic.imgdb.cn/item/611521075132923bf83ccaa6.jpg)

下面用一张图捋一捋上面的逻辑：

[](https://pic.imgdb.cn/item/611521345132923bf83d280d.jpg)

图中包含了一个递归的链式调用，为什么要这样设计呢？


假如不这样设计，我们代码中是不是需要写很多if...else，根据不同的切面和通知单独处理？

而spring巧妙的使用责任链模式消除了原本需要大量的if...else判断，让代码的扩展性更好，很好的体现了开闭原则：对扩展开放，对修改关闭。

## 缓存中存的原始还是代理对象？
我们知道spring中为了性能考虑是有缓存的，通常说包含了三级缓存：

[](https://pic.imgdb.cn/item/611521515132923bf83d6b6d.jpg)

说时迟那时快，刺头青D的兄弟，刺头青F忍不住赶过来问了句：缓存中存的原始还是代理对象？

我竟然被问得一时语塞，仔细捋了捋，要从三个方面回答：

### singletonFactories（三级缓存）
AbstractAutowireCapableBeanFactory类的doCreateBean方法中，有这样一段代码：

[](https://pic.imgdb.cn/item/611521785132923bf83dc34a.jpg)

其实之前已经说过，它是为了解决循环依赖问题。这次要说的是addSingletonFactory方法：

[](https://pic.imgdb.cn/item/611521885132923bf83de740.jpg)

它里面保存的是singletonFactory对象，所以是原始对象。


### earlySingletonObjects（二级缓存）
AbstractBeanFactory类的doGetBean方法中，有这样一段代码：
[](https://pic.imgdb.cn/item/611521b25132923bf83e4585.jpg)

在调用getBean方法获取bean实例时，会调用getSingleton尝试先从缓存中看能否获取到，如果能获取到则直接返回。
[](https://pic.imgdb.cn/item/6115220d5132923bf83f095e.jpg)

这段代码会先从一级缓存中获取bean，如果没有再从二级缓存中获取，如果还是没有则从三级缓存中获取singletonFactory，通过getObject方法获取实例，将该实例放入到二级缓存中。

答案的谜底就聚焦在getObject方法中，而这个方法又是在哪来定义的呢？

其实就是上面的getEarlyBeanReference方法，我们知道这个方法生成的是代理对象，所以二级缓存中存的是代理对象。

### singletonObjects（一级缓存）
DefaultSingletonBeanRegistry类的getSingleton方法中，有这样一段代码：
[](https://pic.imgdb.cn/item/611522315132923bf83f5738.jpg)

此时的bean创建、注入和初始化完成了，判断是如果新的单例对象，则会加入到一级缓存中，具体代码如下：

[](https://pic.imgdb.cn/item/6115224c5132923bf83f9627.jpg)

出于一块钱的友谊，有必要温馨提醒一下：这里是DefaultSingletonBeanRegistry类的getSingleton方法，跟上面说的AbstractBeanFactory类getSingleton方法不一样。

## 几个常见的坑
我是一个乐于分享的人，虽说有时话比较少（旁边：属于人狠话不多的角色，别惹我）。为了表现我的share精神，给大家总结几个我之前使用spring aop遇过的坑。

我们几乎每天都在用spring aop。

“什么？我怎么不知道？” 你可能会问。

如果你每天在用spring事务的话，就是每天在用spring aop，因为spring事务的底层就用到了spring aop。

### 坑1：直接方法调用
使用spring事务时，直接方法调用：
```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public void add(UserModel userModel) {
        userMapper.queryUser(userModel);
        save(userModel);
    }

    @Transactional
    public void save(UserModel userModel) {
        System.out.println("保存数据");
    }
}
```
这种情况直接方法调用spring aop无法生成代理对象，事务会失效。这个问题的解决办法有很多：

1. 使用TransactionTemplate手动开启事务
2. 将事务方法save放到新加的类UserSaveService中，通过userSaveService.save调用事务方法。
3. UserService类中@Autowired注入自己的实例userService，通过userService.save调用事务方法。
4. 通过AopContext类获取代理对象：((UserService)AopContext.currentProxy()).save(user);

### 坑2：访问权限错误
@Service
public class UserService {
    @Autowired
    private UserService userService;
    @Autowired
    private UserMapper userMapper;

    public void add(UserModel userModel) {
        userMapper.queryUser(userModel);
        userService.save(userModel);
    }

    @Transactional
    private void save(UserModel userModel) {
        System.out.println("保存数据");
    }
}
上面用 UserService类中@Autowired注入自己的实例userService的方式解决事务失效问题，如果不出意外的话，是可以的。

但是恰恰出现了意外，save方法被定义成了private的，这时也无法生成代理对象，事务同样会失效。

所以，我们应该拿个小本本记一下，目标方法一定不能定义成private的。

### 坑3：目标类用final修饰
```java
@Service
public class UserService {
    @Autowired
    private UserService userService;
    @Autowired
    private UserMapper userMapper;

    public void add(UserModel userModel) {
        userMapper.queryUser(userModel);
        userService.save(userModel);
    }

    @Transactional
    public final void save(UserModel userModel) {
        System.out.println("保存数据");
    }
}
```
这种情况spring aop生成代理对象，重写save方法时，发现的final的，重写不了，也会导致事务失效。

小本本需要再加一条，目标方法一定不能定义成final的。

### 坑4：循环依赖问题
在使用@Async注解开启异步功能的场景，它会通过AOP自动生成代理对象。
```java
@Service
public class TestService1 {

    @Autowired
    private TestService2 testService2;

    @Async
    public void test1() {
    }
}

@Service
public class TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```
启动服务会报错：
```java
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'testService1': Bean with name 'testService1' has been injected into other beans [testService2] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.
```
至于为什么会报错，我在这里不过多解释了，在我的《[spring：我是如何解决循环依赖的？](https://mp.weixin.qq.com/s?__biz=MzUxODkzNTQ3Nw==&amp;mid=2247485600&amp;idx=1&amp;sn=0c49b94e7fbd35c88c4470e936023e3e&amp;chksm=f9800e7acef7876ca05ab45ce9420ea140f188e84153f23d0af9d044f475458ad38d49a6546a&token=2142272128&lang=zh_CN#rd)》这篇文章中写的很详细。

## 唠唠家常
我最近一直在思考，读者喜欢看什么类型的文章，因为最近阅读量有些惨淡，也许是快要过年的原因吧。我并非是一个高产的写作者，不像有些大佬每日更新，我写一篇文章可能要花一周左右的时间，所有的demo都亲自测试过的，所有的图片都自己画的，可以说是百分百原创。我近期还是挺用心的，尽可能保证文章质量（虽说我的水平有限），想把我知道的很多知识点，分享给大家。但我并非是一个积极的人，需要更多读者正面的反馈。我其实最近同时也在思考另外一个问题：我写的文章到底有没有价值，有没有必要坚持写下去。

我娃最近生病住院了，这个事情让我感到很愧疚。因为最近加班比较多，而且花了很多时间写文章 和 推广文章，陪他的时间比较少。也许，人生很多事情都没法鱼和熊掌都兼得吧。

好了，不说了，2021年大家一起加油，未来一定可期。