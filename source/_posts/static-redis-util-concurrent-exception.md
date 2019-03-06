---
title: springboot中redis工具类并发使用时异常
date: 2019-03-06 23:32:11
tags:
  - redis
  - springboot
  - 并发
---

为了使用的方便，我把`redisTemplate`注入到静态属性中，同时还想动态的切换redis的`database`，不过在并发使用时，遇到了各种奇奇怪怪的bug。


<!-- more -->


### 异常：

并发使用WebUtils是，redis报错：
```
org.springframework.data.redis.RedisSystemException: Unknown redis exception; nested exception is java.util.concurrent.CancellationException

Caused by: io.lettuce.core.RedisException: Connection is closed
```

### 分析原因：

因为我在redis中get时，会先重置数据库连接，如下：

```java
@Slf4j
public class WebUtils extends org.springframework.web.util.WebUtils {

    private static RedisTemplate redisTemplate = SpringContextHolder.getBean("redisTemplate");
    /**
     * 当前服务redis 所用的database
     */
    public final static int DEFAULT_DATABASE = ((LettuceConnectionFactory) redisTemplate.getConnectionFactory()).getDatabase();

    /**
     * 从redis中获取值
     * @param key 键
     * @param database the index of the database 0-15
     * @return 值
     */
    public static Object getRedisValue(int database, String key) {
        resetRedisConnection(database);
        return StrUtil.isBlank(key) ? null : redisTemplate.opsForValue().get(key);
    }

    /**
     * 从redis中获取hash值
     * @param key 键
     * @param database the index of the database 0-15
     * @return 值
     */
    public static Map getRedisHashValue(int database, String key) {
        resetRedisConnection(database);
        return StrUtil.isBlank(key) ? null : transfer(redisTemplate.opsForHash().entries(key));
    }
    
    /**
     * 重置redis连接
     * @param database the index of the database 0-15
     */
    private static void resetRedisConnection(int database) {
        RedisConnectionFactory connectionFactory = redisTemplate.getConnectionFactory();
        LettuceConnectionFactory lettuceConnectionFactory = (LettuceConnectionFactory) connectionFactory;
        lettuceConnectionFactory.setDatabase(database);
        redisTemplate.setConnectionFactory(lettuceConnectionFactory);
        // 重置连接
        lettuceConnectionFactory.resetConnection();
    }
}
```

当`getRedisValue()`和`getRedisHashValue()`并发访问时，可能redis连接还没重置完，所以会出现上述错误

### 解决办法：

在静态方法`getRedisValue()`和`getRedisHashValue()`上加关键字`synchronized`，同时优化`resetRedisConnection()`方法

```java
@Slf4j
public class WebUtils extends org.springframework.web.util.WebUtils {

    private static RedisTemplate redisTemplate = SpringContextHolder.getBean("redisTemplate");
    /**
     * 当前服务redis 所用的database
     */
    public final static int DEFAULT_DATABASE = ((LettuceConnectionFactory) redisTemplate.getConnectionFactory()).getDatabase();

    /**
     * 从redis中获取值
     * @param key 键
     * @param database the index of the database 0-15
     * @return 值
     */
    public static synchronized Object getRedisValue(int database, String key) {
        resetRedisConnection(database);
        return StrUtil.isBlank(key) ? null : redisTemplate.opsForValue().get(key);
    }

    /**
     * 从redis中获取hash值
     * @param key 键
     * @param database the index of the database 0-15
     * @return 值
     */
    public static synchronized Map getRedisHashValue(int database, String key) {
        resetRedisConnection(database);
        return StrUtil.isBlank(key) ? null : transfer(redisTemplate.opsForHash().entries(key));
    }
    
    /**
     * 重置redis连接
     * @param database the index of the database 0-15
     */
    private static void resetRedisConnection(int database) {
        // 当前redis使用的数据库
        int currentDatabase = ((LettuceConnectionFactory) redisTemplate.getConnectionFactory()).getDatabase();
        if (database == currentDatabase) {
            return;
        }
        RedisConnectionFactory connectionFactory = redisTemplate.getConnectionFactory();
        LettuceConnectionFactory lettuceConnectionFactory = (LettuceConnectionFactory) connectionFactory;
        lettuceConnectionFactory.setDatabase(database);
        redisTemplate.setConnectionFactory(lettuceConnectionFactory);
        // 重置连接
        lettuceConnectionFactory.resetConnection();
    }
}
```