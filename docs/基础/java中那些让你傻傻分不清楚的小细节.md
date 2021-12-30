## 前言
最近我们通过sonar静态代码检测，同时配合人工代码review，发现了项目中很多代码问题。除了常规的bug和安全漏洞之外，还有几处方法用法错误，引起了我极大的兴趣。

我为什么会对这几个方法这么感兴趣呢？因为它们极具迷惑性，可能会让我们傻傻分不清楚。

## 1. replace会替换所有字符？
很多时候我们在使用字符串时，想把字符串比如：ATYSDFA*Y中的字符A替换成字符B，第一个想到的可能是使用replace方法。

如果想把所有的A都替换成B，很显然可以用replaceAll方法，因为非常直观，光从方法名就能猜出它的用途。

那么问题来了：replace方法会替换所有匹配字符吗？

jdk的官方给出了答案。

![](https://pic.imgdb.cn/item/610fe6655132923bf81517b5.jpg)

该方法会替换每一个匹配的字符串。

既然replace和replaceAll都能替换所有匹配字符，那么他们有啥区别呢？

replace有两个重载的方法。
其中一个方法的参数：char oldChar 和 char newChar，支持字符的替换。
```java
source.replace('A', 'B')
```
另一个方法的参数是：CharSequence target 和 CharSequence replacement，支持字符串的替换。
```java
source.replace("A", "B")
```
replaceAll方法的参数是：String regex 和 String replacement，基于正则表达式的替换。普通字符串替换：
```java
source.replaceAll("A", "B")
```
正则表达替换（将*替换成C）：
```java
source.replaceAll("\\*", "C")
```
顺便说一下，将*替换成C使用replace方法也可以实现：
```java
source.replace("*", "C")
```
无需对特殊字符进行转义。

不过，千万注意，切勿使用如下写法：
```java
source.replace("\\*", "C")
```
这种写法会导致字符串无法替换。

还有个小问题，如果我只想替换第一个匹配的字符串该怎么办?

这时可以使用replaceFirst方法：
```java
source.replaceFirst("A", "B")
```

## 2. Integer不能用==判断相等？
不知道你在项目中有没有见过，有些同事对Integer类型的两个参数使用==比较是否相等？

反正我见过的，那么这种用法对吗？

我的回答是看具体场景，不能说一定对，或不对。

有些状态字段，比如：orderStatus有：-1(未下单)，0（已下单），1（已支付），2（已完成），3（取消），5种状态。

这时如果用==判断是否相等：
```java
Integer orderStatus1 = new Integer(1);
Integer orderStatus2 = new Integer(1);
System.out.println(orderStatus1 == orderStatus2);
```
返回结果会是true吗？

答案：是false。

有些同学可能会反驳，Integer中不是有范围是：-128-127的缓存吗？

为什么是false？

先看看Integer的构造方法：
![](https://pic.imgdb.cn/item/610fe6d05132923bf81605cd.jpg)

它其实并没有用到缓存。

那么缓存是在哪里用的？

答案在valueOf方法中：

![](https://pic.imgdb.cn/item/610fe6f15132923bf8165237.jpg)

如果上面的判断改成这样：
```java
String orderStatus1 = new String("1");
String orderStatus2 = new String("1");
System.out.println(Integer.valueOf(orderStatus1) == Integer.valueOf(orderStatus2));
```
返回结果会是true吗？

答案：还真是true。

我们要养成良好编码习惯，尽量少用==判断两个Integer类型数据是否相等，只有在上述非常特殊的场景下才相等。

而应该改成使用equals方法判断：
```java
Integer orderStatus1 = new Integer(1);
Integer orderStatus2 = new Integer(1);
System.out.println(orderStatus1.equals(orderStatus2));
```

## 3. 使用BigDecimal就不丢失精度？
通常我们会把一些小数类型的字段（比如：金额），定义成BigDecimal，而不是Double，避免丢失精度问题。

使用Double时可能会有这种场景：
```java
double amount1 = 0.02;
double amount2 = 0.03;
System.out.println(amount2 - amount1);
```
正常情况下预计amount2 - amount1应该等于0.01

但是执行结果，却为：
```java
0.009999999999999998
```
实际结果小于预计结果。

Double类型的两个参数相减会转换成二进制，因为Double有效位数为16位这就会出现存储小数位数不够的情况，这种情况下就会出现误差。

常识告诉我们使用BigDecimal能避免丢失精度。

但是使用BigDecimal能避免丢失精度吗？

答案是否定的。

为什么？
```java
BigDecimal amount1 = new BigDecimal(0.02);
BigDecimal amount2 = new BigDecimal(0.03);
System.out.println(amount2.subtract(amount1));
```
这个例子中定义了两个BigDecimal类型参数，使用构造函数初始化数据，然后打印两个参数相减后的值。

结果：
```java
0.0099999999999999984734433411404097569175064563751220703125
```
不科学呀，为啥还是丢失精度了？

jdk中BigDecimal的构造方法上有这样一段描述：

![](https://pic.imgdb.cn/item/610fe72f5132923bf816deee.jpg)

大致的意思是此构造函数的结果可能不可预测，可能会出现创建时为0.1，但实际是0.1000000000000000055511151231257827021181583404541015625的情况。

由此可见，使用BigDecimal构造函数初始化对象，也会丢失精度。

那么，如何才能不丢失精度呢？
```java
BigDecimal amount1 = new BigDecimal(Double.toString(0.02));
BigDecimal amount2 = new BigDecimal(Double.toString(0.03));
System.out.println(amount2.subtract(amount1));
```
使用Double.toString方法对double类型的小数进行转换，这样能保证精度不丢失。

其实，还有更好的办法：
```java
BigDecimal amount1 = BigDecimal.valueOf(0.02);
BigDecimal amount2 = BigDecimal.valueOf(0.03);
System.out.println(amount2.subtract(amount1));
```
使用BigDecimal.valueOf方法初始化BigDecimal类型参数，也能保证精度不丢失。在新版的阿里巴巴开发手册中，也推荐使用这种方式创建BigDecimal参数。

## 4. 字符串拼接不能用String？
String类型的字符串被称为不可变序列，也就是说该对象的数据被定义好后就不能修改了，如果要修改则需要创建新对象。
```java
String a = "123";
String b = "456";
String c = a + b;
System.out.println(c);
```
在大量字符串拼接的场景中，如果对象被定义成String类型，会产生很多无用的中间对象，浪费内存空间，效率低。

这时，我们可以用更高效的可变字符序列：StringBuilder和StringBuffer，来定义对象。

那么，StringBuilder和StringBuffer有啥区别？

StringBuffer对各主要方法加了synchronized关键字，而StringBuilder没有。所以，StringBuffer是线程安全的，而StringBuilder不是。

其实，我们很少会出现需要在多线程下拼接字符串的场景，所以StringBuffer实际上用得非常少。一般情况下，拼接字符串时我们推荐使用StringBuilder，通过它的append方法追加字符串，它只会产生一个对象，而且没有加锁，效率较高。
```java
String a = "123";
String b = "456";
StringBuilder c = new StringBuilder();
c.append(a).append(b);
System.out.println(c);
```
接下来，关键问题来了：字符串拼接时使用String类型的对象，效率一定比StringBuilder类型的对象低？

答案是否定的。

为什么？

使用javap -c StringTest命令反编译：

![](https://pic.imgdb.cn/item/610fe7805132923bf81795cb.jpg)

从图中能看出定义了两个String类型的参数，又定义了一个StringBuilder类的参数，然后两次使用append方法追加字符串。

如果代码是这样的：
```java
String a = "123";
String b = "789";
String c = a + b;
System.out.println(c);
```
使用javap -c StringTest命令反编译的结果会怎样呢？

![](https://pic.imgdb.cn/item/610fe7a75132923bf817ea2e.jpg)

我们会惊讶的发现，同样定义了两个String类型的参数，又定义了一个StringBuilder类的参数，然后两次使用append方法追加字符串。跟上面的结果是一样的。

其实从jdk5开始，java就对String类型的字符串的+操作做了优化，该操作编译成字节码文件后会被优化为StringBuilder的append操作。

## 5. isEmpty和isBlank的区别
我们在对字符串进行操作的时候，需要经常判断该字符串是否为空。如果没有借助任何工具，我们一般是这样判断的：
```java
if (null != source && !"".equals(source)) {
    System.out.println("not empty");
}
```
但是如果每次都这样判断，会有些麻烦，所以很多jar包都对字符串判空做了封装。目前市面上主流的工具有：

- spring中的StringUtils
- jdbc中的StringUtils
- apache common3中的StringUtils

不过spring中的StringUtils类只有isEmpty方法，没有isNotEmpty方法。

jdbc中的StringUtils类只有isNullOrEmpty方法，也没有isNotNullOrEmpty方法。

所以在这里强烈推荐一下apache common3中的StringUtils类，它里面包含了很多实用的判空方法：isEmpty、isBlank、isNotEmpty、isNotBlank等，还有其他字符串处理方法。

问题来了，isEmpty和isBlank有啥区别？

使用isEmpty方法判断：
```java
 StringUtils.isEmpty(null)      = true
 StringUtils.isEmpty("")        = true
 StringUtils.isEmpty(" ")       = false
 StringUtils.isEmpty("bob")     = false
 StringUtils.isEmpty("  bob  ") = false
``` 
使用isBlank方法判断：
```java
StringUtils.isBlank(null)      = true
StringUtils.isBlank("")        = true
StringUtils.isBlank(" ")       = true
StringUtils.isBlank("bob")     = false
StringUtils.isBlank("  bob  ") = false
```
两个方法关键的区别在于这种" "空字符串的情况，isNotEmpty返回false，而isBlank返回true。

## 6. mapper查询结果要判空？
有次代码review的时候，当时有个同事说这里的判空可以去掉，让我记忆犹新：
```java
List<User> list = userMapper.query(search);
if(CollectionUtils.isNotEmpty(list)) {
    List<Long> idList = list.stream().map(User::getId).collect(Collectors.toList());
}
```
因为按常理，一般调用方法查询出来的集合，可能为null，需要判空的。但是，这里比较特殊，我查了一下mybatis的源码，这个判空的代码还真的可以去掉。

怎么回事呢？

mybatis的查询方法最终都会调到DefaultResultSetHandler类的handleResultSets方法：
![](https://pic.imgdb.cn/item/610fe81c5132923bf818e568.jpg)

![](https://pic.imgdb.cn/item/610fe82a5132923bf81903d2.jpg)

该方法会返回一个multipleResultsList集合对象，在方法刚开始就new出来了，肯定是不会为空。

所以，如果你在项目的代码中看到有人直接使用查询出的结果，不判空也不要惊讶：
```java
List<User> list = userMapper.query(search);
List<Long> idList = list.stream().map(User::getId).collect(Collectors.toList());
```
因为mapper底层已经处理过的，它不会出现空指针异常。

## 7. indexOf方法的正确用法
有次在review别人代码的时候，看到有个地方indexOf使用了这种写法，让我印象比较深刻：
```java
String source = "#ATYSDFA*Y";
if(source.indexOf("#") > 0) {
    System.out.println("do something");
}
```
你们说这段代码会打印出do something吗？

答案是否定的。

为什么呢？

jdk官方说了不存在的情况会返回-1图片indexOf方法返回的是指定元素在字符串中的位置，从0开始。而上面的例子#在字符串的第一个位置，所以调用indexOf方法后的值其实是0。所以，条件是false，不会打印do something。

如果想通过indexOf判断某个元素是否存在时，要用：
```java
if(source.indexOf("#") > -1) {
    System.out.println("do something");
}
```
其实，还有更优雅的contains方法：
```java
if(source.contains("#")) {
   System.out.println("do something");
}
```

