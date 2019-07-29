---
title: spring-boot-cache 缓存提升系统响应速度
date: 2019-07-29 11:39:18
tags:
---
### 1. 缓存简介
缓存：缓存就是数据交换的缓冲区（称作Cache），当某一硬件要读取数据时，会首先从缓存中查找需要的数据，如果找到了则直接执行，找不到的话则从内存中找。为什么使用缓存？究其原因就是缓存的读写速度远快与磁盘，从减轻I/O开销和加快运行速度方便都有很好的效果。那么我们缓存什么？哪些经常读取而又不经常修改的数据，那些数据量较大又很少修改的数据。缓存策略三要素：缓存命中率、缓存更新策略、最大缓存容量。

cache 可以说是后端提高响应速度、承载能力的标准套路了
spring boot中提供spring boot starter cache 组件 配合spring boot starter redis 或者其他缓存组件 可以很简单的使用缓存。

### 2. spring cache 支持的缓存类型
- Generic
- JCache (JSR-107)
- EhCache 2.x
- Hazelcast
- Infinispan
- Redis
- Guava
- Simple
如果不满足上述的缓存方案 可以自实现 cacheManager。

#### 注解介绍
- @Cacheable

获取缓存 如果有缓存 直接返回。
`@Cacheable `可以标记在一个方法上，也可以标记在一个类上。当标记在一个方法上时表示该方法是支持缓存的，当标记在一个类上时则表示该类所有的方法都是支持缓存的。对于一个支持缓存的方法，Spring会在其被调用后将其返回值缓存起来，以保证下次利用同样的参数来执行该方法时可以直接从缓存中获取结果，而不需要再次执行该方法。Spring在缓存方法的返回值时是以键值对进行缓存的，值就是方法的返回结果，至于键的话，Spring又支持两种策略，默认策略和自定义策略。

![](https://upload-images.jianshu.io/upload_images/11462107-b1a29edb1461fd2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

value：缓存的名称，在 spring 配置文件中定义，必须指定至少一个。如@Cacheable(value=”mycache”) 或者@Cacheable(value={”cache1”,”cache2”} 
key：缓存的 key，可以为空，如果指定要按照 SpEL 表达式编写，如果不指定，则缺省按照方法的所有参数进行组合。如@Cacheable(value=”testcache”,key=”#userName”) 
condition：缓存的条件，可以为空，使用 SpEL 编写，返回 true 或者 false，只有为 true 才进行缓存。如@Cacheable(value=”testcache”,condition=”#userName.length()>2”) 

- @CachePut
执行并且更新缓存相关 不管如何 肯定会执行方法 然后返回 这样可以更新缓存的内容，@CachePut也可以标注在类上和方法上。使用@CachePut时我们可以指定的属性跟@Cacheable是一样的。

在支持Spring Cache的环境下，对于使用@Cacheable标注的方法，Spring在每次执行前都会检查Cache中是否存在相同key的缓存元素，如果存在就不再执行该方法，而是直接从缓存中获取结果进行返回，否则才会执行并将返回结果存入指定的缓存中。@CachePut也可以声明一个方法支持缓存功能。与@Cacheable不同的是使用@CachePut标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果，而是每次都会执行该方法，并将执行结果以键值对的形式存入指定的缓存中。 

- @CacheEvict
删除缓存相关，

@CacheEvict是用来标注在需要清除缓存元素的方法或类上的。当标记在一个类上时表示其中所有的方法的执行都会触发缓存的清除操作。@CacheEvict可以指定的属性有value、key、condition、allEntries和beforeInvocation。其中value、key和condition的语义与@Cacheable对应的属性类似。即value表示清除操作是发生在哪些Cache上的（对应Cache的名称）；key表示需要清除的是哪个key，如未指定则会使用默认策略生成的key；condition表示清除操作发生的条件。allEntries和beforeInvocation。 
allEntries：是否清空所有缓存内容，缺省为 false，如果指定为 true，则方法调用后将立即清空所有缓存。如：@CachEvict(value=”testcache”,allEntries=true) 。
beforeInvocation：是否在方法执行前就清空，缺省为 false，如果指定为 true，则在方法还没有执行的时候就清空缓存，缺省情况下，如果方法执行抛出异常，则不会清空缓存。如：@CachEvict(value=”testcache”，beforeInvocation=true) 
allEntries指定为true时，则会清楚所有缓存。 

- @Caching

@Caching注解可以让我们在一个方法或者类上同时指定多个Spring Cache相关的注解。其拥有三个属性：cacheable、put和evict，分别用于指定@Cacheable、@CachePut和@CacheEvict。使用如下
```java
@Caching(cacheable = {@Cacheable(value = "user", key = "#id", condition = "#id != '123'"),
            @Cacheable(value = "user", key = "#id", condition = "#id != '321'")}
    )
    public User findById(String id) {
        System.out.println("执行数据库查询方法");
        return userDao.findById(id);
    }
```

### 实践
- 引入依赖
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
```

- 配置
```
spring:
  redis:
    host: <ip>
    port: <port>
    password: <password>
  cache:
    # spring cache 缓存类型为redis  也可以是其他的实现 
    type: redis
```
- 使用cache

模拟带缓存的service
```java
package com.ming;

import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
//公共配置  可以在类上注释 注释本类的 缓存相关公共配置
//@CacheConfig(cacheNames = TestCacheService.CACHE_KEY)
public class TestCacheService {

    public static final String CACHE_KEY = "test-cache";

    /**
     * 获取信息  第二次访问会取缓存
     *
     * @author ming
     * @date 2018-07-11 17:41:47
     */
    @Cacheable(cacheNames = CACHE_KEY ,key = "#id")
    public String testCache(String id) {
        return getString(id);
    }


    /**
     * 更新信息   更新缓存
     *
     * @author ming
     * @date 2018-07-12 09:50:53
     */
    @CachePut(cacheNames = CACHE_KEY ,key = "#id")
    public String testCachePut(String id) {
        return getString(id + "update");
    }

    /**
     * 清除缓存
     *
     * @author ming
     * @date 2018-07-12 09:51:22
     */
    @CacheEvict(cacheNames = CACHE_KEY ,key = "#id")
    public void removeCache(String id) {
        System.out.println("删除缓存 ");
    }


    /**
     * 获取string 模拟调用方法
     *
     * @author ming
     * @date 2018-07-11 17:41:58
     */
    private String getString(String id) {
        try {
            Thread.sleep(3000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return id + "load";
    }


}
```
- 测试用例
```java
package com.ming;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = Start.class)
public class TestCache {

    @Autowired
    private TestCacheService testCacheService;

    @Test
    public void test() {
        String id = "ming";
        System.out.println("第一次访问没有缓存--------");
        long oneNow = System.currentTimeMillis();
        System.out.println(testCacheService.testCache(id));
        System.out.println("耗时:" + (System.currentTimeMillis() - oneNow) + "ms");


        System.out.println("第二次访问有缓存--------");
        long twoNow = System.currentTimeMillis();
        System.out.println(testCacheService.testCache(id));
        System.out.println("耗时:" + (System.currentTimeMillis() - twoNow) + "ms");


        System.out.println("更新缓存信息--------");
        long threeNow = System.currentTimeMillis();
        System.out.println(testCacheService.testCachePut(id));
        System.out.println("耗时:" + (System.currentTimeMillis() - threeNow) + "ms");


        System.out.println("获取更新后的缓存信息-------");
        long fourNow = System.currentTimeMillis();
        System.out.println(testCacheService.testCache(id));
        System.out.println("耗时:" + (System.currentTimeMillis() - fourNow) + "ms");


        System.out.println("移除缓存------并且调用testCache方法");
        testCacheService.removeCache(id);
        long fiveNow = System.currentTimeMillis();
        System.out.println(testCacheService.testCache(id));
        System.out.println("耗时:" + (System.currentTimeMillis() - fiveNow) + "ms");
    }
}
```
- 注意事项

@Cacheable 、@CachePut、@CacheEvict 必须要有 cacheNames或 value
注解必须放在public修饰的方法上。
如果只是获取缓存使用@Cacheable即可 如果要更新数据库并且更新缓存一定要使用@CachePut 否则@Cacheable会出现脏读。

### 总结
spring cache 为缓存提供了一套简单快捷的方案 可以很快速添加上缓存
具体缓存的实现 也有更多的选择 也可以自己实现spring cache的缓存管理器 来实现自定义的缓存。



