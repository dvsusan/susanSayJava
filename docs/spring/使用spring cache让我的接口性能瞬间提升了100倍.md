## 前言
笔者之前做商城项目时，做过商城首页的商品分类功能。当时考虑分类是放在商城首页，以后流量大，而且不经常变动，为了提升首页访问速度，我考虑使用缓存。对于java开发而言，首先的缓存当然是redis。

优化前系统流程图：

![](https://pic.imgdb.cn/item/61152b715132923bf857438d.jpg)

我们从图中可以看到，分类功能分为生成分类数据 和 获取分类数据两个流程，生成分类数据流程是有个JOB每隔5分钟执行一次，从mysql中获取分类数据封装成首页需要展示的分类数据结构，然后保存到redis中。获取分类数据流程是商城首页调用分类接口，接口先从redis中获取数据，如果没有获取到再从mysql中获取。

一般情况下从redis就都能获取数据，因为相应的key是没有设置过期时间的，数据会一直都存在。以防万一，我们做了一次兜底，如果获取不到数据，就会从mysql中获取。



本以为万事大吉，后来，在系统上线之前，测试对商城首页做了一次性能压测，发现qps是100多，一直上不去。我们仔细分析了一下原因，发现了两个主要的优化点：去掉多余的接口日志打印 和 分类接口引入redis cache做一次二级缓存。日志打印我在这里就不多说了，不是本文的重点，我们重点说一下redis cache。

优化后的系统流程图：

![](https://pic.imgdb.cn/item/61152b915132923bf8579b2c.jpg)

我们看到，其他的流程都没有变，只是在获取分类接口中增加了先从spring cache中获取分类数据的功能，如果获取不到再从redis中获取，再获取不到才从mysql中获取。


经过这样一次小小的调整，再重新压测接口，性能一下子提升了N倍，满足了业务要求。如此美妙的一次优化经验，有必要跟大家分析一下。


我将从以下几个方面给大家分享一下spring cache。

1. 基本用法
2. 项目中如何使用
3. 工作原理

## 一、基本用法
SpringCache缓存功能的实现是依靠下面的这几个注解完成的。

- @EnableCaching：开启缓存功能
- @Cacheable：获取缓存
- @CachePut：更新缓存
- @CacheEvict：删除缓存
- @Caching：组合定义多种缓存功能
- @CacheConfig：定义公共设置，位于类之上

@EnableCaching注解是缓存的开关，如果要使用缓存功能，就必要打开这个开关，这个注解可以定义在Configuration类或者springboot的启动类上面。

@Cacheable、@CachePut、@CacheEvict 这三个注解的用户差不多，定义在需要缓存的具体类或方法上面。
```java
   @Cacheable(key="'id:'+#id")
   public User getUser(int id) {
        return userService.getUserById(id);
   }

   @CachePut(key="'id:'+#user.id")
   public User insertUser(User user) {
       userService.insertUser(user);
       return user;
   }

   @CacheEvict(key="'id:'+#id")
   public int deleteUserById(int id) {
        userService.deleteUserById(id);
        return id;
   }
```

需要注意的是@Caching注解跟另外三个注解不同，它可以组合另外三种注解，自定义新注解。

```java
@Caching(
        cacheable = {@Cacheable(/*value = "emp",*/key = "#lastName")
        put = {@CachePut(/*value = "emp",*/key = "#result.id")}
)
public Employee getEmpByLastName(String lastName){
    return  employeeMapper.getEmpByLastName(lastName);
}
```

@CacheConfig一般定义在配置类上面，可以抽取缓存的公共配置，可以定义这个类全局的缓存名称，其他的缓存方法就可以不配置缓存名称了。
```java
@CacheConfig(cacheNames = "emp")
@Service
public class EmployeeService 
```



## 二、项目中如何使用
### 1. 引入caffeine的相关jar包

我们这里使用caffeine，而非guava，因为Spring Boot 2.0中取代了guava
```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.6.0</version>
</dependency>
```
## 2. 配置CacheManager

开启EnableCaching
```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(){
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        //Caffeine配置
        Caffeine<Object, Object> caffeine = Caffeine.newBuilder()
                //最后一次写入后经过固定时间过期
                .expireAfterWrite(10, TimeUnit.SECONDS)
                //缓存的最大条数
                .maximumSize(1000);
        cacheManager.setCaffeine(caffeine);
        return cacheManager;
    }
}
```
### 3.使用Cacheable注解获取数据
```java
@Service
public class CategoryService {
    
    //category是缓存名称,#type是具体的key，可支持el表达式
    @Cacheable(value = "category", key = "#type")
    public CategoryModel getCategory(Integer type) {
        return getCategoryByType(type);
    }

    private CategoryModel getCategoryByType(Integer type) {
        System.out.println("根据不同的type:" + type + "获取不同的分类数据");
        CategoryModel categoryModel = new CategoryModel();
        categoryModel.setId(1L);
        categoryModel.setParentId(0L);
        categoryModel.setName("电器");
        categoryModel.setLevel(3);
        return categoryModel;
    }
}
```
### 4.测试
```java
@Api(tags = "category", description = "分类相关接口")
@RestController
@RequestMapping("/category")
public class CategoryController {

    @Autowired
    private CategoryService categoryService;

    @GetMapping("/getCategory")
    public CategoryModel getCategory(@RequestParam("type") Integer type) {
        return categoryService.getCategory(type);
    }
}
```
在浏览器中调用接口：

![](https://pic.imgdb.cn/item/61152c5d5132923bf8599be6.jpg)

可以看到，有数据返回。

再看看控制台的打印。

![](https://pic.imgdb.cn/item/61152c6b5132923bf859c00a.jpg)

有数据打印，说明第一次请求进入了categoryService.getCategory方法的内部。

然后再重新请求一次，

![](https://pic.imgdb.cn/item/61152c5d5132923bf8599be6.jpg)

还是有数据，返回。但是控制台没有重新打印新数据，还是以前的数据，说明这一次请求走的是缓存，没有进入categoryService.getCategory方法的内部。在5分钟以内，再重复请求该接口，一直都是直接从缓存中获取数据。

![](https://pic.imgdb.cn/item/61152c6b5132923bf859c00a.jpg)

说明缓存生效了，下面我介绍一下spring cache的工作原理

## 三、工作原理

通过上面的例子，相当朋友们对spring cache在项目中的用法有了一定的认识。那么它的工作原理是什么呢？

相信聪明的朋友们，肯定会想到，它用了AOP。

没错，它就是用了AOP。那么具体是怎么用的？

我们先看看EnableCaching注解
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {

  // false JDK动态代理 true cglib代理   
  boolean proxyTargetClass() default false;
  
  //通知模式 JDK动态代理 或 AspectJ
  AdviceMode mode() default AdviceMode.PROXY;
 
  //排序
  int order() default Ordered.LOWEST_PRECEDENCE;

}
```

这个数据很简单，定义了代理相关参数，引入了CachingConfigurationSelector类。再看看该类的getProxyImports方法

```java
private String[] getProxyImports() {
    List<String> result = new ArrayList<>(3);
    result.add(AutoProxyRegistrar.class.getName());
    result.add(ProxyCachingConfiguration.class.getName());
    if (jsr107Present && jcacheImplPresent) {
      result.add(PROXY_JCACHE_CONFIGURATION_CLASS);
    }
    return StringUtils.toStringArray(result);
  }
```  
该方法引入了AutoProxyRegistrar和ProxyCachingConfiguration两个类

AutoProxyRegistrar是让spring cache拥有AOP的能力（至于如何拥有AOP的能力，这个是单独的话题，感兴趣的朋友可以自己阅读一下源码。或者关注一下我的公众账号，后面会有专门AOP的专题）。

重点看看ProxyCachingConfiguration
```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

  @Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor() {
    BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
    advisor.setCacheOperationSource(cacheOperationSource());
    advisor.setAdvice(cacheInterceptor());
    if (this.enableCaching != null) {
      advisor.setOrder(this.enableCaching.<Integer>getNumber("order"));
    }
    return advisor;
  }

  @Bean
  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  public CacheOperationSource cacheOperationSource() {
    return new AnnotationCacheOperationSource();
  }

  @Bean
  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  public CacheInterceptor cacheInterceptor() {
    CacheInterceptor interceptor = new CacheInterceptor();
    interceptor.setCacheOperationSources(cacheOperationSource());
    if (this.cacheResolver != null) {
      interceptor.setCacheResolver(this.cacheResolver);
    }
    else if (this.cacheManager != null) {
      interceptor.setCacheManager(this.cacheManager);
    }
    if (this.keyGenerator != null) {
      interceptor.setKeyGenerator(this.keyGenerator);
    }
    if (this.errorHandler != null) {
      interceptor.setErrorHandler(this.errorHandler);
    }
    return interceptor;
  }

}
```

哈哈哈，这个类里面定义了AOP的三大要素：advisor、interceptor和Pointcut，只是Pointcut是在BeanFactoryCacheOperationSourceAdvisor内部定义的。

![](https://pic.imgdb.cn/item/61152cde5132923bf85ad979.jpg)

另外定义了CacheOperationSource类，该类封装了cache方法签名注解的解析工作，形成CacheOperation的集合。它的构造方法会实例化SpringCacheAnnotationParser，现在看看这个类的parseCacheAnnotations方法。

```java
private Collection<CacheOperation> parseCacheAnnotations(
    DefaultCacheConfig cachingConfig, AnnotatedElement ae, boolean localOnly) {

  Collection<CacheOperation> ops = null;
  //找@cacheable注解方法
  Collection<Cacheable> cacheables = (localOnly ? AnnotatedElementUtils.getAllMergedAnnotations(ae, Cacheable.class) :
      AnnotatedElementUtils.findAllMergedAnnotations(ae, Cacheable.class));
  if (!cacheables.isEmpty()) {
    ops = lazyInit(null);
    for (Cacheable cacheable : cacheables) {
      ops.add(parseCacheableAnnotation(ae, cachingConfig, cacheable));
    }
  }
  
 //找@cacheEvict注解的方法
  Collection<CacheEvict> evicts = (localOnly ? AnnotatedElementUtils.getAllMergedAnnotations(ae, CacheEvict.class) :
      AnnotatedElementUtils.findAllMergedAnnotations(ae, CacheEvict.class));
  if (!evicts.isEmpty()) {
    ops = lazyInit(ops);
    for (CacheEvict evict : evicts) {
      ops.add(parseEvictAnnotation(ae, cachingConfig, evict));
    }
  }
  
 //找@cachePut注解的方法
  Collection<CachePut> puts = (localOnly ? AnnotatedElementUtils.getAllMergedAnnotations(ae, CachePut.class) :
      AnnotatedElementUtils.findAllMergedAnnotations(ae, CachePut.class));
  if (!puts.isEmpty()) {
    ops = lazyInit(ops);
    for (CachePut put : puts) {
      ops.add(parsePutAnnotation(ae, cachingConfig, put));
    }
  }

 //找@Caching注解的方法
  Collection<Caching> cachings = (localOnly ? AnnotatedElementUtils.getAllMergedAnnotations(ae, Caching.class) :
      AnnotatedElementUtils.findAllMergedAnnotations(ae, Caching.class));
  if (!cachings.isEmpty()) {
    ops = lazyInit(ops);
    for (Caching caching : cachings) {
      Collection<CacheOperation> cachingOps = parseCachingAnnotation(ae, cachingConfig, caching);
      if (cachingOps != null) {
        ops.addAll(cachingOps);
      }
    }
  }

  return ops;
}
```

我们看到这个类会解析@cacheable、@cacheEvict、@cachePut 和 @Caching注解的参数，封装到CacheOperation集合中。

此外，spring cache 功能的关键就是上面的拦截器：CacheInterceptor，它最终会调到这个方法：


```java
@Nullable
private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
  // Special handling of synchronized invocation
  if (contexts.isSynchronized()) {
    CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
    if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
      Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
      Cache cache = context.getCaches().iterator().next();
      try {
        return wrapCacheValue(method, cache.get(key, () -> unwrapReturnValue(invokeOperation(invoker))));
      }
      catch (Cache.ValueRetrievalException ex) {
        // The invoker wraps any Throwable in a ThrowableWrapper instance so we
        // can just make sure that one bubbles up the stack.
        throw (CacheOperationInvoker.ThrowableWrapper) ex.getCause();
      }
    }
    else {
      // No caching required, only call the underlying method
      return invokeOperation(invoker);
    }
  }


  // 执行@CacheEvict的逻辑，这里是当beforeInvocation为true时清缓存
  processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
      CacheOperationExpressionEvaluator.NO_RESULT);

  // 获取命中的缓存对象
  Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

  //如果没有命中，则生成一个put的请求
  List<CachePutRequest> cachePutRequests = new LinkedList<>();
  if (cacheHit == null) {
    collectPutRequests(contexts.get(CacheableOperation.class),
        CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
  }

  Object cacheValue;
  Object returnValue;

  if (cacheHit != null && !hasCachePut(contexts)) {
    // If there are no put requests, just use the cache hit
    cacheValue = cacheHit.get();
    returnValue = wrapCacheValue(method, cacheValue);
  }
  else {
    // 如果没有获得缓存对象，则调用业务方法获得返回对象，这是关键代码
    returnValue = invokeOperation(invoker);
    cacheValue = unwrapReturnValue(returnValue);
  }

  // 收集@CachePuts数据
  collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

  // 执行cachePut或没有命中的Cacheable请求，将返回对象放到缓存中
  for (CachePutRequest cachePutRequest : cachePutRequests) {
    cachePutRequest.apply(cacheValue);
  }

  // 执行@CacheEvict的逻辑，这里是当beforeInvocation为false时清缓存
  processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);

  return returnValue;
}
```

也行有些朋友看到这里会有一个疑问：

既然spring cache的增删改查都有了，为啥还要 @Caching 注解呢？

其实是这样的：spring考虑如果除了增删改查之外，如果用户需要自定义自己的注解，或者有些比较复杂的功能需要增删改查的情况，这时就可以用@Caching 注解来实现。

还要一个问题：

上面的例子中使用的缓存key是#type，但是如果有些缓存key比较复杂，是实体中的几个字段组成的，这种情况要如何定义呢？

一起看看下面的例子：
```java
@Data
public class QueryCategoryModel {

    /**
     * 系统编号
     */
    private Long id;

    /**
     * 父分类编号
     */
    private Long parentId;

    /**
     * 分类名称
     */
    private String name;

    /**
     * 分类层级
     */
    private Integer level;

    /**
     * 类型
     */
    private Integer type;

}

@Cacheable(value = "category", key = "#type")
public CategoryModel getCategory2(QueryCategoryModel queryCategoryModel) {
    return getCategoryByType(queryCategoryModel.getType());
}
```

这个例子中需要用到QueryCategoryModel实体对象的所有字段，作为一个key，这种情况要如何定义呢？

1.自定义一个类实现KeyGenerator接口
```java
public class CategoryGenerator  implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        return target.getClass().getSimpleName() + "_"
                + method.getName() + "_"
                + StringUtils.arrayToDelimitedString(params, "_");
    }
}
```
2.在CacheConfig类中定义CategoryGenerator的bean实例
```java
@Bean
public CategoryGenerator categoryGenerator() {
    return new CategoryGenerator();
}
```

3.修改之前定义的key
```java
@Cacheable(value = "category", key = "categoryGenerator")
public CategoryModel getCategory2(QueryCategoryModel queryCategoryModel) {
    return getCategoryByType(queryCategoryModel.getType());
}
```