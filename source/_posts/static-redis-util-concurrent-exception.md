---
title: springboot中redis工具类并发使用时异常
date: 2019-03-06 23:32:11
categories:
  - tech
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
    
    public static UserInfo getUserInfo() {
        if (StrUtil.isEmpty(getSessionId())) {
            return null;
        }
        Object o = getRedisValue(getSessionKey(getSessionId()));
        return UserInfo.parseJson(o);
    }
}
```

### 后续bug && 重点

在有些微服务中，使用的是`@Autowired`自动注入`RedisTemplate`，同时还使用了`WebUtils#getRedisValue()`。如：

```java
@RestController
public class DictionaryController {
    
    @Autowired
    private RedisTemplate redisTemplate;
    
    @ApiOperation("查询字典表")
    @RequestMapping(value = "/dicArray", method = RequestMethod.GET)
    public Object dicArray (String type) {
        // 使用的是redis database 1
        ValueOperations valueOperations = redisTemplate.opsForValue();
        return valueOperations.get(AmlBasicConstants.DICT_TYPE + type);
    }
    
    public List<SystemMenu> getMenuCount() {
        // 从redis中获取用户信息，使用的是redis database 0
        UserInfo userInfo = WebUtils.getUserInfo(); 
        
        ...
        
    }
}
```

当`getMenuCount()`调用出现异常后，会把`redisTemplate`的 database 重置为0，之后 查询字典表 接口就获取不到数据了（因为字典存的是database 1），一直返回的为 null.

monitor 命令查看 redis 请求日志: 

```
172.17.0.5:6379> monitor
OK

1554853703.046199 [1 10.10.121.39:62040] "GET" "aml-basic:dict:upload_url"

...
// 调用getMenuCount()出现异常后，使用的是database 0

1554853703.046199 [0 10.10.121.39:62040] "GET" "aml-basic:dict:upload_url"
```

后来我查了下，不同的应用一般用同一个数据库，比如 database 0，只有不同环境用不同的数据库，比如生产环境用 database 0，测试环境用 database 1. 

这里有个redis database相关的文章，大家可以参考下：[勿用 redis 的多库](http://blog.kankanan.com/article/52ff7528-redis-7684591a5e93.html)