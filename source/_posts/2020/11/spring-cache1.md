---
title: SpringCache的使用
date: 2020-11-02 21:26:28
tags:
- 缓存
- SpringCache
categories:
- Spring
- SpringCache
---

### 前言

   在开发一些配置相关的业务时，由于数据变化频率很低，很适合在查询的时候使用缓存。但对于一些复杂的数据结构，使用redis存取又觉得比较麻烦。
   幸好，Spring提供了注解式的缓存方案-SpringCache。所以本章就来总结下SpringCache的用法。
<!-- more -->

### 注解
   
   SpringCache借助注解来实现对数据的缓存及删除。所以，这里以每个注解为入口依次了解他们的使用。

#### @EnableCaching
   
   @EnableCaching注解用来开启SpringCache的缓存功能。
   首先来看看他的结构：
{% codeblock lang:java %}
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
	// 开启基于CGLIB代理的开关，默认为基于JDK的代理
	boolean proxyTargetClass() default false;
	// AOP的实现方式，默认为基于JDK的，也可以基于AspectJ实现
	AdviceMode mode() default AdviceMode.PROXY;
	// 一个切点存在多个通知时的执行顺序，默认最低
	int order() default Ordered.LOWEST_PRECEDENCE;
}
{% endcodeblock %}
   接着就开始正式的配置。
   一般来说，为了实现个性化配置，我们会单独建一个配置类：
{% codeblock lang:java %}
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        // 这里方便测试，使用最简单的CacheManager
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        // 使用CurrentMapCache作为缓存容器，default为默认的key值
        cacheManager.setCaches(Arrays.asList(new ConcurrentMapCache("default")));
        return cacheManager;
    }
}
{% endcodeblock %}
   关于CacheManager的使用，SpringCache提供了对应各种缓存工具实现类。
   例如针对redis的RedisCacheManager，还有针对encache的EhCacheCacheManager的等。
   这里我们使用的是SimpleCacheManager，通常是用于测试使用，不需要太多配置。

#### @Cacheable

   @Cacheable用来新增缓存，每次会先判断有无缓存，有则取缓存，无则执行方法并将结果放入缓存
   源码如下：
{% codeblock lang:java %}
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Cacheable {
    // 缓存的名字 与cacheNames属性互为别名，两者都是配置缓存名字
    @AliasFor("cacheNames")
    String[] value() default {};
    @AliasFor("value")
    String[] cacheNames() default {};
    // 定义缓存的key值，使用SpEl，可以动态取入参的值
    // "" 表示所有入参的值都被当作key，前提是没有配置keyGenerator
    String key() default "";
    // 指定key的生成策略
    // 与key属性互斥，二者只能配置一个
    String keyGenerator() default "";
    // 自定义的CacheManager(当定义了多个时使用)
    String cacheManager() default "";
    // 自定义的CacheResolver
    // 与CacheManager互斥
    String cacheResolver() default "";
    // 设置使用缓存的条件，同样使用SpEl语法
    String condition() default "";
    // 设置不适用缓存的条件，使用SpEl语法
    String unless() default "";
    // 是否使用异步模式，为true时无法使用unless属性
    boolean sync() default false;
}
{% endcodeblock %}


#### @CachePut

   @CachePut用来更新缓存，每次都会执行方法并将结果放入缓存
   源码如下：
{% codeblock lang:java %}
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface CachePut {
    // 缓存名称
    @AliasFor("cacheNames")
    String[] value() default {};
    @AliasFor("value")
    String[] cacheNames() default {};
    // 缓存key
    String key() default "";
    // 缓存key的生成策略
    String keyGenerator() default "";
    // 自定义的CacheResolver
    String cacheManager() default "";
    // 自定义的CacheResolver
    String cacheResolver() default "";
    // 设置使用缓存的条件
    String condition() default "";
    // 设置不使用缓存的条件
    String unless() default "";
}
{% endcodeblock %}
   @CachePut的属性和@Cacheable基本一样，除了没有异步开关属性。

#### @CahceEvict

   @CacheEvice用来清除缓存，执行方法后会将对应缓存清除掉
   源码如下：
{% codeblock lang:java %}
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface CacheEvict {
    // 缓存名称
    @AliasFor("cacheNames")
    String[] value() default {};
    @AliasFor("value")
    String[] cacheNames() default {};
    // 缓存key
    String key() default "";
    // 缓存key的生成策略
    String keyGenerator() default "";
    // 自定义的CacheResolver
    String cacheManager() default "";
    // 自定义的CacheResolver
    String cacheResolver() default "";
    // 设置删除缓存的条件
    String condition() default "";
    // 是否删除缓存内所有内容，当为true时，不能指定key
    boolean allEntries() default false;
    // 是否在方法执行前删除缓存
    // 若为true则表示在方法执行前就删除缓存，方法执行成功与否不影响删除操作
    boolean beforeInvocation() default false;
}
{% endcodeblock %}

#### @Caching
   
   @Cacheing用作在同一方法上配置多个缓存操作
   源码如下：
{% codeblock lang:java %}
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Caching {
    // 多个缓存操作
    Cacheable[] cacheable() default {};
    // 多个更新缓存操作
    CachePut[] put() default {};
    // 多个缓存删除操作
    CacheEvict[] evict() default {};
}
{% endcodeblock %}

#### @CacheConfig
    
   @CacheConfig提供了公用配置的实现，避免多次重复配置
   源码如下：
{% codeblock lang:java %}
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CacheConfig {
    // 缓存名称
    String[] cacheNames() default {};
    // 缓存key的生成策略
    String keyGenerator() default "";
    // 自定义的CacheResolver
    String cacheManager() default "";
    // 自定义的CacheResolver
    String cacheResolver() default "";
}
{% endcodeblock %}

### 使用示例
	
   虽然SpringCache简化了对缓存的操作，但相应的也有了很多限制。

#### 基本用法
   三个缓存的操作注解分别对应查询，更新，删除操作，但更新操作需要方法也返回对应的结果集。
   而更新的接口通常只会返回操作结果，成功或者失败。所以这里使用@CacheEvice直接删除缓存会更方便些。
{% codeblock lang:java %}
@Cacheable(key = "#key", value = "default")
public Object getInfo(String key) {
    // find data from DB
    return result;
}
@CacheEvict(key = "#key", value = "default")
public String update(String key) {
    // update data
    return result;
}
// 每次更新操作后都会清除缓存，这样下次执行查询操作就会直接走库，达到缓存更新的目的。
{% endcodeblock %}

#### 多个CacheManager
   前面有说到，我们可以配置多个不同的CacheManager,然后在每个操作注解上指定想要的CacheManager。
   这里简单举一个例子：
{% codeblock lang:java %}
// @Primary指定默认的CacheManager
@Primary
@Bean
public CacheManager cacheManager() {
    SimpleCacheManager cacheManager = new SimpleCacheManager();
    cacheManager.setCaches(Arrays.asList(new ConcurrentMapCache("default")));
    return cacheManager;
}
@Bean
public CacheManager slaveCacheManager() {
    ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
    List<String> list = Lists.newArrayList();
    list.add("salve1");
    list.add("salve2");
    cacheManager.setCacheNames(list);
    return cacheManager;
}

// 指定了slaveCacheManager 相应的缓存的名称也改为对应的salve1
@Override
@Cacheable(key = "#key", cacheManager = "slaveCacheManager",value = "salve1")
public String cacheAbleTest(String key) {
    log.info("cache able:{}", key);
    this.value = "cache able";
    return value;
}
// 没有指定CacheManager所以会使用默认的
@Override
@CachePut(key = "#key", value = "default")
public String cachePutTest(String key) {
    log.info("cache put:{}", key);
    this.value = "cache put";
    return value;
}
{% endcodeblock %}


#### CacheResolver的使用
   
  先看看默认实现的CacheResolver：

{% codeblock lang:java %}
// 实现了CacheResolver的抽象类，默认实现的CacheResolver也是继承自该抽象类
public abstract class AbstractCacheResolver implements CacheResolver, InitializingBean {

  @Nullable
  private CacheManager cacheManager;

  protected AbstractCacheResolver() {}
  protected AbstractCacheResolver(CacheManager cacheManager) {
    this.cacheManager = cacheManager;
  }

  public void setCacheManager(CacheManager cacheManager) {
    this.cacheManager = cacheManager;
  }

  public CacheManager getCacheManager() {
    Assert.state(this.cacheManager != null, "No CacheManager set");
    return this.cacheManager;
  }

  @Override
  public void afterPropertiesSet()  {
    Assert.notNull(this.cacheManager, "CacheManager is required");
  }

  // 实现自接口的方法，返回包含缓存对象的集合
  @Override
  public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {
    Collection<String> cacheNames = getCacheNames(context);
    if (cacheNames == null) {
      return Collections.emptyList();
    }
    Collection<Cache> result = new ArrayList<>(cacheNames.size());
    // 通过缓存名称获取不同的cacheManager
    for (String cacheName : cacheNames) {
      Cache cache = getCacheManager().getCache(cacheName);
      if (cache == null) {
        throw new IllegalArgumentException("Cannot find cache named '" +
            cacheName + "' for " + context.getOperation());
      }
      result.add(cache);
    }
    return result;
  }

  @Nullable
  protected abstract Collection<String> getCacheNames(CacheOperationInvocationContext<?> context);
}
// 继承了抽象类AbstractCacheResolver
public class SimpleCacheResolver extends AbstractCacheResolver {

  public SimpleCacheResolver() {}
  // 构造方法，设置指定的CacheManager
  public SimpleCacheResolver(CacheManager cacheManager) {
    super(cacheManager);
  }

  @Override
  protected Collection<String> getCacheNames(CacheOperationInvocationContext<?> context) {
    return context.getOperation().getCacheNames();
  }

  @Nullable
  static SimpleCacheResolver of(@Nullable CacheManager cacheManager) {
    return (cacheManager != null ? new SimpleCacheResolver(cacheManager) : null);
  }
}
{% endcodeblock %}
   从上述代码可以看出，CacheResolver的作用是管理CacheManager，可以根据特定规则选择CacheManager。
   下面是根据上述例子实现的CacheResolver，根据缓存名称返回CacheManager:
{% codeblock lang:java %}
public class MyCacheResolver implements CacheResolver {
    private static final Logger logger = LoggerFactory.getLogger(MyCacheResolver.class);

    private List<CacheManager> cacheManagerList;

    public MyCacheResolver(List<CacheManager> cacheManagerList) {
        this.cacheManagerList = cacheManagerList;
    }

    @Override
    public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {
        // context包含的内容
        logger.info("method name :{}", context.getMethod().getName());
        logger.info("args:{}", context.getArgs());
        logger.info("target:{}", context.getTarget());
        // 获取当前执行方法设置的缓存名称
        String[] values = context.getMethod().getAnnotation(Cacheable.class).value();
        logger.info("cacheName:{}", values);
        Collection<Cache> caches = Lists.newArrayList();
        for (CacheManager cacheManager : cacheManagerList) {
            // 根据缓存名称从集合中选择CacheManager
            if (cacheManager.getCacheNames().contains(values[0])) {
                logger.info("cache manager:{}", cacheManager);
                caches.add(cacheManager.getCache(values[0]));
            }
        }
        return caches;
    }
}

// 然后在配置中新建对应的CacheResolver方法
@Configuration
@EnableCaching
public class CacheConfig {

    @Primary
    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(Arrays.asList(new ConcurrentMapCache("default")));
        return cacheManager;
    }


    @Bean
    public CacheManager slaveCacheManager() {
        ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
        List<String> list = Lists.newArrayList();
        list.add("salve1");
        list.add("salve2");
        cacheManager.setCacheNames(list);
        return cacheManager;
    }

    @Bean
    public CacheResolver cacheResolver() {
        List<CacheManager> list = Lists.newArrayList();
        list.add(cacheManager());
        list.add(slaveCacheManager());
        return new MyCacheResolver(list);
    }
}
{% endcodeblock %}

### 总结

      不得不说，SpringCache极大的简化了项目中对缓存的使用。
      不过，这也意味着限制了对缓存的操作，就目前来看，SpringCache适合于获取变化较少的数据的
    查询接口上，例如一些配置信息的查询等。
      当然，SpringCache也提供了CacheManager和CacheResolver共开发者扩展，不过本章只是简单
    展示了他们的用法，所以暂时不了解他们更多的运用技巧。
      总体来说，SpringCache的优势在于快速实现，代码规范化及简化，兼容性高，扩展性都比较高。
    如果不考虑对缓存生命周期设置或者一些精细操作上的需求，SpringCache必然是最好的选择。