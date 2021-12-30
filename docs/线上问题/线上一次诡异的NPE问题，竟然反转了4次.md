## 前言
公司为了保证系统的稳定性，加了很多监控，比如：接口响应时间、cpu使用率、内存使用率、错误日志等等。如果系统出现异常情况，会邮件通知相关人员，以便于大家能在第一时间解决隐藏的系统问题。此外，我们这边有个不成文的规定，就是线上问题最好能够当日解决，除非遇到那种非常棘手的问题。


## 1.起因
有个周一的早上，我去公司上班，查看邮件，收到我们老大转发的一封邮件，让我追查线上的一个NPE（NullPointException）问题。

邮件是通过`sentry`发出来的，我们通过点击邮件中的相关链接，可以直接跳转到sentry的详情页面。在这个页面中，展示了很多关键信息，比如：操作时间、请求的接口、出错的代码位置、报错信息、请求经过了哪些链路等等。真是居家旅行，查bug的良药，有了这些，小case一眼就能查到原因。

我当时没费吹灰之力，就访问到了NPE的sentry报错页面（其实只用鼠标双击一下就搞定）。果然上面有很多关键信息，我一眼就看到了NPE的具体代码位置：
```java
notify.setName(CurrentUser.getCurrent().getUserName());
```
剧情发展得如此顺利，我都有点不好意思了。

根据类名和代码行号，我在idea中很快找到那行代码，不像是我写的，这下可以放心不用背锅了。于是接下来看了看那行的代码修改记录，最后修改人是XXX。

什么？是他？

他在一个月前已经离职了，看来这个无头公案已经无从问起，只能自己查原因。

我当时内心的OS是：`代码没做兼容处理`。

**为什么这么说呢？**

这行代码其实很简单，就是从当前`用户上下文`中获取用户名称，然后设置到notify实体的inUserName字段上，最终notify的数据会保存到数据库。

该字段表示那条`推送通知`的添加人，正常情况下没啥卵用，主要是为了出现线上问题扯皮时，有个地方可以溯源。如果出现冤案，可以还你清白。

> 顺便提一嘴，这里说的`推送通知`跟mq中的`消息`是两回事，前者指的是`websocket`长连接推送的实时通知，我们这边很多业务场景，在页面功能操作完之后，会实时推送通知给指定用户，以便用户能够及时处理相关单据，比如：您有一个审批单需要审批，请及时处理等。

`CurrentUser`内部包含了一个`ThreadLocal`对象，它负责保存当前线程的用户上下文信息。当然为了保证在线程池中，也能从用户上下文中获取到正确的用户信息，这里用了阿里的`TransmittableThreadLocal`。伪代码如下：
```java
@Data
public class CurrentUser {
    private static final TransmittableThreadLocal<CurrentUser> THREA_LOCAL = new TransmittableThreadLocal<>();
    
    private String id;
    private String userName;
    private String password;
    private String phone;
    ...
    
    public statis void set(CurrentUser user) {
      THREA_LOCAL.set(user);
    }
    
    public static void getCurrent() {
      return THREA_LOCAL.get();
    }
}
```
> 这里为什么用了阿里的`TransmittableThreadLocal`，而不是普通的`ThreadLocal`呢？在线程池中，由于线程会被多次复用，导致从普通的`ThreadLocal`中无法获取正确的用户信息。父线程中的参数，没法传递给子线程，而`TransmittableThreadLocal`很好解决了这个问题。

然后在项目中定义一个全局的`spring mvc拦截器`，专门设置用户上下文到ThreadLocal中。伪代码如下：
```java
public class UserInterceptor extends HandlerInterceptorAdapter {
   
   @Override  
   public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
      CurrentUser user = getUser(request);
      if(Objects.nonNull(user)) {
         CurrentUser.set(user);
      }
   } 
}
```

用户在请求我们接口时，会先触发该拦截器，它会根据用户cookie中的token，调用调用接口获取redis中的用户信息。如果能获取到，说明用户已经登录，则把用户信息设置到CurrentUser类的ThreadLocal中。

接下来，在api服务的下层，即business层的方法中，就能轻松通过`CurrentUser.getCurrent();`方法获取到想要的用户上下文信息了。
![](https://pic.imgdb.cn/item/610e92645132923bf8e9a302.jpg)

这套用户体系的想法是很good的，但深入使用后，发现了一个小插曲：

api服务和mq消费者服务都引用了business层，business层中的方法两个服务都能直接调用。

我们都知道在api服务中用户是需要登录的，而mq消费者服务则不需要登录。
![](https://pic.imgdb.cn/item/610e92745132923bf8e9bfa3.jpg)

如果business中的某个方法刚开始是给api开发的，在方法深处使用了`CurrentUser.getCurrent();`获取用户上下文。但后来，某位新来的帅哥在mq消费者中也调用了那个方法，并未发觉这个小机关，就会中招，出现找不到用户上下文的问题。
![](https://pic.imgdb.cn/item/610e92935132923bf8e9f4d6.jpg)

所以我当时的第一个想法是：`代码没做兼容处理`，因为之前这类问题偶尔会发生一次。

想要解决这个问题，其实也很简单。只需先判断一下能否从CurrentUser中获取用户信息，如果不能，则取配置的系统用户信息。伪代码如下：
```java
@Autowired
private BusinessConfig businessConfig;

CurrentUser user = CurrentUser.getCurrent();
if(Objects.nonNull(user)) {
   entity.setUserId(user.getUserId());
   entity.setUserName(user.getUserName());
} else {
   entity.setUserId(businessConfig.getDefaultUserId());
   entity.setUserName(businessConfig.getDefaultUserName());
}
```
这种简单无公害的代码，如果只是在一两个地方加还OK。

但如果有多个地方都在获取用户信息，难道在每个地方都需要把相同的判断逻辑写一遍？对于有追求的程序员来说，这种简单的重复是写代码的大忌，如何更优雅的解决问题呢？

答案将会在文章后面揭晓。

这个NPE问题表面上，已经有答案了。根据以往的经验，由于在代码中没有做兼容处理，在mq消费者服务中获取到的用户信息为空，对一个空对象，调用它的方法，就会出现NPE。


## 2.第一次反转
但这个答案显得有点草率，会不会还有什么机关？

于是我在项目工程中全局搜索`CurrentUser.set`关键字，竟然真的找到了一个机关。

**剧情出现第一次反转。**

有个地方写了一个`rocketmq`的`AOP拦截器`，伪代码如下：
```java
@Aspect
@Component
public class RocketMqAspect {

   @Pointcut("execution(* onMessage(..)&&@within(org.apache.rocketmq.spring.annotation.RocketMQMessageListener))")
   public void pointcut() {
   
   }
   ...

   @Around(value="pointcut")
   public void around(ProceedingJoinPoint point) throws Throwable {
      if(point.getArgs().length == 1 && point.getArgs()[0] instanceof MessageExt) {
         Message message = (Message)point.getArgs()[0];
         String userId = message.getUserProperty("userId");
         String userName = message.getUserProperty("userName");
         if(StringUtils.notEmpty(userId) && StringUtils.notEmpty(userName))  {
             CurrentUser user = new CurrentUser();
             user.setUserId(userId);
             user.setUserName(userName);
             CurrentUser.set(user);
         }
      }
      
      ...
   }
}
```
它会拦截所有mq消费者中的`onMessage`方法，在该方法执行之前，从`userProperty`中获取用户信息，并且创建用户对象，设置到用户上下文中。

> 温馨提醒一下，免得有些朋友依葫芦画瓢踩坑。上面的伪代码只给出了设置用户上下文的关键代码，用完后，删除用户上下文的代码没有给出，感兴趣的朋友可以找我私聊。

既然有获取用户信息的地方，我猜测必定有设置的地方。这时候突然发现自己有点当侦探的潜力，因为后面还真找到了。

意不意外，惊不惊喜？

另外一个同事自己自定义了一个`RocketMQTemplate`。伪代码如下：
```java
public class MyRocketMQTemplate extends RocketMQTemplate {
    
    @Override
    public void asyncSend(String destnation, Meassage<?> message, SendCallback sendCallback, long timeout, int delayLevel) {
      
      MessageBuilder builder = withPayload(message.getPayLoad());
      CurrentUser user = CurrentUser.getCurrent();
      builder.setHeader("userId", user.getUserId());
      builder.setHeader("userName", user.getUserName());
      
      super.asyncSend(destnation,message,sendCallback,timeout,delayLevel);
    }
}
```
这段代码的主要作用是在mq生产者在发送异步消息之前，先将当前用户上下文信息设置到mq消息的header中，这样在mq消费者中就能通过`userProperty`获取到，它的本质也是从header中获取到的。

![](https://pic.imgdb.cn/item/610e92ab5132923bf8ea1b7a.jpg)

这个设计比较巧妙，完美的解决了mq的消费者中通过`CurrentUser.getCurrent();`无法获取用户信息的问题。

此时线索一下子断了，没有任何进展。

我再去查了一下服务器的日志。确认了那条有问题的mq消息,它的header信息中确实没有userId和userName字段。

莫非是mq生产者没有往header中塞用户信息？这是需要重点怀疑的地方。

因为mq生产者是另外一个团队写的代码，在EOA（签报系统）回调他们系统时，会给我们发mq消息，通知我们签报状态。

而EOA是第三方的系统，用户体系没有跟我们打通。所以在另外一个团队的回调接口中，没法获取当前登录的用户信息，AOP的拦截器就没法自动往header中塞用户信息，这样在mq的消费者中自然就获取不到了。

![](https://pic.imgdb.cn/item/610e92c25132923bf8ea4142.jpg)

这样想来还真的是顺理成章。

## 3.第二次反转

但真的是这样的吗？

我们抱着很大的希望，给他们发了一封邮件，让他们帮忙查一下问题。

很快，他们回邮件了。

但他们说：已经本地测试过，功能正常。

**就这样剧情第二次反转了。**

我此时有点好奇，他们是怎么往header中塞用户信息的。带着“学习的心态”，于是找他们一起查看了相关代码。

他们在发送mq消息之前，会调用一个UserUtil工具`注入用户`。该工具类的伪代码如下：
```java
@Component
public class UserUtil{
    @Value("${susan.userId}")
    private String userId;

    @Value("${susan.userName}")
    private String userName;

    public void injectUser() {
        CurrentUser user = new CurrentUser();
        user.setUserId(userId);
        user.setUserName(userName);
        CurrentUser.set(user);
    }
}
```
好吧，不得不承认，这样做确实可以解决`header`传入用户信息的问题，比之前需要手动判断用户信息是否为空要优雅得多，因为注入之后的用户信息肯定是不为空的。

![](https://pic.imgdb.cn/item/610e92dc5132923bf8ea6885.jpg)

折腾了半天，NPE问题还是没有着落。

我回头再仔细看了那个自定义的`RocketMQTemplate`类，发现里面重写的方法：`asyncSend`，它包含了5个参数。而他们在给我们推消息时，调用的`asyncSend`却只传了3个参数。

一下子，问题又有了新的进展，有没有可能是他们调错接口了？

原本应该调用5个参数的方法，但实际上他们调用了3个参数的方法。

这样就能解释通了。


## 4.第三次反转
终于有点思路，我带着一份喜悦，准备开始证明刚刚的猜测。

但事实证明，我真的高兴的太早了，马上被啪啪打脸。

**这次是反转最快的一次。**

怎么回事呢？

原本我以为是另外一个团队的人，在发mq消息时调错方法了，应该调用5个参数的`asyncSend`方法，但他们的代码中实际上调用的是3个参数的同名方法。

为了防止出现冤枉同事的事情发生。我本着尽职尽责的态度，仔细看了看`RocketMQTemplate`类的所有方法，这个类是`rocketmq`框架提供的。

意外得发现了一些藕断丝连的关系，伪代码如下：
```java
public void asyncSend(String destination, Message<?> message, SendCallback sendCallback, long timeout, int delayLevel) {
  if (Objects.isNull(message) || Objects.isNull(message.getPayload())) {
      log.error("asyncSend failed. destination:{}, message is null ", destination);
      throw new IllegalArgumentException("`message` and `message.payload` cannot be null");
    }

    try {
        org.apache.rocketmq.common.message.Message rocketMsg = RocketMQUtil.convertToRocketMessage(objectMapper,
            charset, destination, message);
        if (delayLevel > 0) {
            rocketMsg.setDelayTimeLevel(delayLevel);
        }
        producer.send(rocketMsg, sendCallback, timeout);
    } catch (Exception e) {
        log.info("asyncSend failed. destination:{}, message:{} ", destination, message);
        throw new MessagingException(e.getMessage(), e);
    }
}
    

public void asyncSend(String destination, Message<?> message, SendCallback sendCallback, long timeout) {
    asyncSend(destination,message,sendCallback,timeout,0);
}

public void asyncSend(String destination, Message<?> message, SendCallback sendCallback) {
    asyncSend(destination, message, sendCallback, producer.getSendMsgTimeout());
}

public void asyncSend(String destination, Object payload, SendCallback sendCallback, long timeout) {
     Message<?> message = this.doConvert(payload, null, null);
     asyncSend(destination, message, sendCallback, timeout);
}

public void asyncSend(String destination, Object payload, SendCallback sendCallback) {
    asyncSend(destination, payload, sendCallback, producer.getSendMsgTimeout());
}
```
这个背后隐藏着一个天大的秘密，这些同名的方法殊途同归，竟然最终都会调用5个参数的asyncSend方法。

这样看来，如果在子类中重写了5个的asyncSend方法，相当于重写了所有的asyncSend方法。
![](https://pic.imgdb.cn/item/610e92f85132923bf8ea951b.jpg)
再次证明他们没错。
> 温馨提醒一下，有些类的重载方法会相互调用，如果在子类中重新了最底层的那个重载方法，等于把所有的重载方法都重写了。

头疼，又要回到原点了。


## 5.第四次反转
此时，我有点迷茫了。

不过，有个好习惯是：遇到线上问题不知道怎办时，会多查一下日志。

本来不报啥希望的，但是没想到通过再查日志。

**出现了第四次反转。**

这次抱着试一下的心态，根据messageID去查了mq生产者的日志，查到了一条消息的发送日志。

这次眼镜擦得雪亮，发现了一个小细节：`时间不对`。

这条日志显示的消息发送日期是2021-05-21，而实际上mq消费者处理的日期是2021-05-28。

**这条消息一个星期才消费完？**

显然不是。

我有点肃然起敬了。再回去用那个messageID查了mq消费者的日志，发现里面其实消费了6次消息。前5次竟然是同一天，都在2021-05-21，而且都处理失败了。另一次是2021-05-28，处理成功了。

**为什么同一条消息，会在同一天消费5次？**

如果你对rocketmq比较熟悉的话，肯定知道它支持重试机制。

如果mq消费者消息处理失败了，可以在业务代码中抛一个异常。然后框架层面捕获该异常返回ConsumeConcurrentlyStatus.RECONSUME_LATER，rocketmq会自动将该消息放到`重试队列`。
![](https://pic.imgdb.cn/item/610e93175132923bf8eac6da.jpg)

流程图如下：
![](https://pic.imgdb.cn/item/610e932f5132923bf8eae666.jpg)

这样mq消费者下次可以重新消费那条消息，直到达到一定次数（这里我们配置的5次），rocketmq会将那条消息发送到`死信队列`。![](https://pic.imgdb.cn/item/610e93425132923bf8eb01e1.jpg)

流程图如下：
![](https://pic.imgdb.cn/item/610e935d5132923bf8eb2ac6.jpg)
后面就不再消费了。


**最后为什么会多消费一次？**

最后的那条消息不可能是其他的mq生产者发出的，因为messageID是唯一的，其他的生产者不可能产生一样的messageID。

那么接下来，只有一种可能，那就是`人为发了条消息`。

> 查线上日志时，时间、messageID、traceID、记录条数 这几个维度至关重要。

## 6.真相
后来发现还真的是人为发的消息。

一周前，线上有个用户，由于EOA页面回调接口失败（重试也失败），导致审核状态变更失败。该审核单在EOA系统中审批通过了，但mq消费者去处理该审核单的时候，发现状态还是待审核，就直接返回了，没有走完后续的流程，从而导致该审核单数据数据异常。

为了修复这个问题，我们当时先修改了线上该审核单的状态。接下来，手动的在rocketmq后台发了条消息。由于当时在rocketmq后台看不到header信息，所以发消息时没有管header，直接往指定的topic中发消息了。

> 千万注意，大家在手动发mq消息时，一定要注意header中是否也需要设置相关参数，尤其是rocketmq，不然就可能会出问题。

mq消费者消费完那条消息之后，该审核单正常走完了流程，当时找测试一起测试过，数据库的状态都是正常的。

大家都以为没有问题了，但是所有人都忽略了一个小细节：就是在正常业务逻辑处理完之后，会发websocket通知给指定用户。
但这个功能是已经离职的那个同事加的新逻辑，其他人都不知道。站在手动发消息的那个人的角度来说，他没错，因为他根本不知道新功能的存在。

由于这行代码是最后一行代码，并且跟之前的代码不在同一个事物当中，即使出了问题也不会影响正常的业务逻辑。

所以这个NPE问题影响范围很小，只是那个商户没有收到某个通知而已。

> 有个好习惯，就是把跟核心业务逻辑无关的代码，放在事务之外，防止出现问题时，影响主流程。

说实话，有时候遇到线上问题，对于我们来说未必是一件坏事。通过这次线上问题定位，让我熟悉了公司更多新功能，学习了其他同事的一些好的思想，总结了一些经验和教训，是一次难得的提升自己的好机会。


