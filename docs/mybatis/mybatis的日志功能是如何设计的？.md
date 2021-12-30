## 引言
我们在使用mybatis时，如果出现sql问题，一般会把mybatis配置文件中的logging.level参数改成debug，这样就能在日志中看到某个mapper最终执行sql、入参和影响数据行数。我们拿到sql和入参，手动拼接成完整的sql，然后将该sql在数据库中执行一下，就基本能定位到问题原因。mybatis的日志功能使用起来还是非常方便的，大家有没有想过它是如何设计的呢？

## 从logging目录开始
我们先看一下mybatis的logging目录，该目录的功能决定了mybatis使用什么日志工具打印日志。

logging目录结构如下：

![](https://pic.imgdb.cn/item/610fe9705132923bf81bc1a7.jpg)

它里面除了jdbc目录，还包含了7个子目录，每一个子目录代表一种日志打印工具，目前支持6种日志打印工具和1种非日志打印工具。我们用一张图来总结一下

![](https://pic.imgdb.cn/item/610fe9835132923bf81bf1c9.jpg)

除了上面的8种日志工具之外，它还抽象出一个Log接口，所有的日志打印工具必须实现该接口，后面可以面向接口编程。定义了LogException异常，该异常是日志功能的专属异常，如果你有看过mybatis其他源码的话，不难发现，其他功能也定义专属异常，比如：DataSourceException等，这是mybatis的惯用手法，主要是为了将异常细粒度的划分，以便更快定位问题。

此外，它还定义了LogFactory日志工厂，以便于屏蔽日志工具实例的创建细节，让用户使用起来更简单。

### 如果是你该如何设计这个功能？
我们按照上面目录结构的介绍其实已经有一些思路：

1. 定义一个Log接口，以便于统一抽象日志功能，这8种日志功能都实现Log接口，并且重写日志打印方法。
2. 定义一个LogFactory日志工厂，它会根据我们项目中引入的某个日志打印工具jar包，创建一个具体的日志打印工具实例。

看起来，不错。但是，再仔细想想，LogFactory中如何判断项目中引入了某个日志打印工具jar包才创建相应的实例呢？我们第一个想到的可能是用if...else判断不就行了，再想想感觉用if...else不好，7种条件判断太多了，并非优雅的编程。这时候，你会想一些避免太长if...else判断的方法，当然如果你看过我之前写的文章《消除if...else的9条锦囊妙计》，可能已经学到了几招，但是mybatis却用了一个新的办法。

### mybatis是如何设计这个功能的？
### 1.从Log接口开始
![](https://pic.imgdb.cn/item/610fea0b5132923bf81d0b7f.jpg)

它里面抽象了日志打印的5种方法和2种判断方法。

### 2.再分析LogFactory的代码
![](https://pic.imgdb.cn/item/610fea3d5132923bf81d66ab.jpg)

它里面定义了一个静态的构造器logConstructor，没有用if...else判断，在static代码块中调用了6个tryImplementation方法，该方法会启动一个执行任务去调用了useXXXLogging方法，创建日志打印工具实例。

![](https://pic.imgdb.cn/item/610fea545132923bf81d8c56.jpg)

当然tryImplementation方法在执行前会判断构造器logConstructor为空才允许执行任务中的run方法。下一步看看useXXXLogging方法：
![](https://pic.imgdb.cn/item/610fea915132923bf81e1ab5.jpg)

看到这里，聪明的你可能会有这样的疑问，从上图可以看出mybatis定义了8种useXXXLogging方法，但是在前面的static静态代码块中却只调用了6种，这是为什么？

对比后发现：useCustomLogging 和 useStdOutLogging 前面是没调用的。useStdOutLogging它里面使用了StdOutImpl类

![](https://pic.imgdb.cn/item/610feb005132923bf81f19f4.jpg)

该类其实就是通过JDK自带的System类的方法打印日志的，无需引入额外的jar包，所以不参与static代码块中的判断。

而useCustomLogging方法需要传入一个实现了Log接口的类，如果mybatis默认提供的6种日志打印工具不满足要求，以便于用户自己扩展。

而这个方法是在Configuration类中调用的，如果用户有自定义logImpl参数的话。
![](https://pic.imgdb.cn/item/610feb1a5132923bf81f5338.jpg)

![](https://pic.imgdb.cn/item/610feb295132923bf81f7093.jpg)

具体是在XMLConfigBuilder类的settingsElement方法中调用
![](https://pic.imgdb.cn/item/610feb3e5132923bf81f9d03.jpg)

再回到前面LogFactory的setImplementation方法
![](https://pic.imgdb.cn/item/610feb505132923bf81fc540.jpg)

它会先找到实现了Log接口的类的构造器，返回将该构造器赋值给全局的logConstructor。

这样一来，就可以通过getLog方法获取到Log实例。
![](https://pic.imgdb.cn/item/610feb645132923bf81feec5.jpg)

然后在业务代码中通过下面这种方式获取Log对象，调用它的方法打印日志了。
![](https://pic.imgdb.cn/item/610feb785132923bf8201be7.jpg)

梳理一下LogFactory的流程：

- 在static代码块中根据逐个引入日志打印工具jar包中的日志类，先判断如果全局变量logConstructor为空，则加载并获取相应的构造器，如果可以获取到则赋值给全局变量logConstructor。
- 如果全局变量logConstructor不为空，则不继续获取构造器。
- 根据getLog方法获取Log实例
- 通过Log实例的具体日志方法打印日志

在这里还分享一个知识点，如果某个工具类里面都是静态方法，那么要把该工具类的构造方法定义成private的，防止被疑问调用，LogFactory就是这么做的。
![](https://pic.imgdb.cn/item/610feba35132923bf8207819.jpg)

### 3.适配器模式
日志模块除了使用工厂模式之外，还是有了适配器模式。

> 适配器模式会将所需要适配的类转换成调用者能够使用的目标接口

涉及以下几个角色：

- 目标接口（ Target ）
- 需要适配的类（ Adaptee ）
- 适配器（ Adapter)
![](https://pic.imgdb.cn/item/610fec0a5132923bf8216ede.jpg)

mybatis是怎么用适配器模式的?
![](https://pic.imgdb.cn/item/610fec2f5132923bf821c297.jpg)

上图中标红的类对应的是Adapter角色，Log是Target角色。
![](https://pic.imgdb.cn/item/610fec3f5132923bf821e500.jpg)

而LogFactory就是Adaptee，它里面的getLog方法里面包含是需要适配的对象。

## sql执行日志打印原理
从上面已经能够确定使用哪种日志打印工具，但在sql执行的过程中是如何打印日志的呢？这就需要进一步分析logging目录下的jdbc目录了。

![](https://pic.imgdb.cn/item/610fec6b5132923bf8224afc.jpg)

看看这几个类的关系图：

![](https://pic.imgdb.cn/item/610fec7a5132923bf8226b75.jpg)

ConnectionLogger、PreparedStatementLogger、ResultSetLogger和StatementLogger都继承了BaseJdbcLogger类，并且实现了InvocationHandler接口。从类名非常直观的看出，这4种类对应的数据库jdbc功能。

![](https://pic.imgdb.cn/item/610fecbf5132923bf822fe95.jpg)

它们实现了InvocationHandler接口意味着它用到了动态代理，真正起作用的是invoke方法，我们以ConnectionLogger为例：
![](https://pic.imgdb.cn/item/610feceb5132923bf8235cf0.jpg)

如果调用了prepareStatement方法，则会打印debug日志。
![](https://pic.imgdb.cn/item/610fed045132923bf8239103.jpg)

上图中传入的original参数里面包含了\n\t等分隔符，需要将分隔符替换成空格，拼接成一行sql。

最终会在日志中打印sql、入参和影响行数：
![](https://pic.imgdb.cn/item/610fed1c5132923bf823c49e.jpg)

上图中的sql语句是在ConnectionLogger类中打印的

那么入参和影响行数呢？

入参在PreparedStatementLogger类中打印的
![](https://pic.imgdb.cn/item/610fed3e5132923bf82409e6.jpg)

影响行数在ResultSetLogger类中打印的
![](https://pic.imgdb.cn/item/610fed4f5132923bf8242d91.jpg)

大家需要注意的一个地方是：sql、入参和影响行数只打印了debug级别的日志，其他级别并没打印。所以需要在mybatis配置文件中的logging.level参数配置成debug，才能打印日志。

## 彩蛋
不知道大家有没有发现这样一个问题：

在LogFactory的代码中定义了很多匿名的任务执行器
![](https://pic.imgdb.cn/item/610fed715132923bf824756a.jpg)

但是在实际调用时，却没有在线程中执行，而是直接调用的，这是为什么？

![](https://pic.imgdb.cn/item/610fed825132923bf82498c3.jpg)

答案是为了保证顺序执行，如果所有的日志工具jar包都有，加载优先级是：slf4j 》commonsLog 》log4j2 》log4j 》jdkLog 》NoLog

还有个问题，顺序执行就可以了，为什么要把匿名内部类定义成Runnable的呢？

这里非常有迷惑性，因为它没创建Thread类，并不会多线程执行。我个人认为，这里是mybatis的开发者的一种偷懒，不然需要定义一个新类代替这种执行任务的含义，还不如就用已有的。