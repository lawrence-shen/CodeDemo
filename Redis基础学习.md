---

---

# Redis基础

Redis是一个基于内存的key-value结构数据库。

## 简介

Redis是用C语言开发的一个开源的高性能键值对(key-value)数据库，官方提供的数据是可以达到100000+的QPS（每秒内查询次数）。它存储的value类型比较丰富，也被称为结构化的NoSql数据库。

关系型数据库(RDBMS)

- Mysql
- Oracle
- DB2
- SQLServer

非关系型数据库(NoSql)

- Redis
- Mongo db
- MemCached

### Redis 应用场景

- 缓存

- 任务队列

- 消息队列

- 分布式锁

  

## 安装

### 安装环境

docker、docker-compose

### 安装方法

可以直接使用docker-compose方式，进入配置文件目录下，运行docker-compose.yaml（用docker安装配置其他数据库或服务器也可以使用类似的方法）

```
version: '3'
services:
  redis:
    image: redis:latest
    hostname: redis
    container_name: redis
    restart: always
    ports:
      # 端口映射
      - '6379:6379'
    volumes:
      # 目录映射
      - './etc/redis.conf:/usr/local/etc/redis/redis.conf'
      - './db:/data'
    # 在容器中执行的命令
    command: redis-server /usr/local/etc/redis/redis.conf
```



## 使用

### 准备工作

在pom.xml中添加依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

配置数据库连接信息

```
spring:
  redis:
    host: ${SPRING_REDIS_DEFAULT_URL:172.23.16.196}
    port: ${SPRING_REDIS_PORT:6379}
    password: ${SPRING_REDIS_PASSWORD:}
    database: ${SPRING_REDIS_DATABASE:1}
```

注入redisTemplate对象

```
@Autowired
private RedisTemplate<String, String> redisTemplate;
```

### 常用数据结构

- string（字符串）—— 普通字符串，常用

  ```
  @GetMapping("/str")
  public Object str() {
      String key = packageKey("str");
      redisTemplate.opsForValue().set(key, "hello world");
      return redisTemplate.opsForValue().get(key);
  ```

- hash（哈希）—— hash 适合存储对象

  ```
  @GetMapping("/hash")
  public Object hash() {
      String key = packageKey("hash");
      redisTemplate.opsForHash().put(key, "hello", "world");
      redisTemplate.opsForHash().put(key, "nice", "redis");
      return redisTemplate.opsForHash().get(key, "nice");
  }
  ```

- list（列表）—— list 按照插入顺序排序，可以有重复元素

  ```
  @GetMapping("/list")
  public Object list() {
      String key = packageKey("list");
      redisTemplate.opsForList().set(key, 1L, "world");
      redisTemplate.opsForList().set(key, 0L, "hello");
      redisTemplate.opsForList().set(key, 3L, "redis");
      return redisTemplate.opsForList().index(key, 1L);
  }
  ```

- set（集合）—— set 无序集合，没有重复元素

  ```
  @GetMapping("/set")
  public Object set() {
      String key = packageKey("set");
      redisTemplate.opsForSet().add(key, "hello", "world");
      return redisTemplate.opsForSet().members(key);
  }
  ```

- zset（有序集合）—— sorted set 有序集合，没有重复元素

  ```
  @GetMapping("/zset")
  public Object zset() {
      String key = packageKey("zset");
      redisTemplate.opsForZSet().add(key, "world", 2);
      redisTemplate.opsForZSet().add(key, "hello", 1);
      return redisTemplate.opsForSet().members(key);
  }
  ```

  #### 运行结果查询

![](C:\学习笔记\培训期第一周\Redis\截图素材\1658466365861.png)

![](C:\学习笔记\培训期第一周\Redis\截图素材\微信截图_20220722130907.png)

![](C:\学习笔记\培训期第一周\Redis\截图素材\微信截图_20220722130920.png)

![](C:\学习笔记\培训期第一周\Redis\截图素材\微信截图_20220722130925.png)

### 缓存

开启缓存
	@EnableCaching

配置缓存

- 构建缓存@Cacheable

  ```
  @GetMapping("/{key}")
      @Cacheable(key = "#key", cacheNames = "test")
      public String query(@PathVariable String key) {
          // 这一步可以返回查询数据库的结果
          LOGGER.debug("查询数据库");
          return "hello world";
      }
  ```

- 更新缓存@CachePut

  ```
  @PostMapping("/{key}")
      @CachePut(key = "#key", cacheNames = "test")
      public String update(@PathVariable String key) {
          // 这一步可以直接更新数据库
          LOGGER.debug("更新数据库");
          return "hello world update";
      }
  ```

- 删除缓存@CacheEvict

  ```
  @DeleteMapping("/{key}")
      @CacheEvict(key = "#key", cacheNames = "test")
      public void delete(@PathVariable String key) {
          // 这一步可以直接删除数据库对应的数据
          LOGGER.debug("删除数据库");
      }
  ```

### 监听

#### 发布消息

```
@GetMapping("/{topic}")
    public void t1(@PathVariable String topic) {
        redisTemplate.convertAndSend(topic, "hello world");
        LOGGER.debug("发布消息");

    }
```

#### 接收消息

```
@Bean
    public RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory,
                                                   @Qualifier("catAdapter") MessageListenerAdapter adapter) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(adapter, new PatternTopic("test"));
        return container;
    }
```

```
@Bean
    public MessageListenerAdapter catAdapter() {
        return new MessageListenerAdapter((MessageListener) (message, pattern) -> {
            String topic = new String(message.getChannel());
            String body = new String(message.getBody());
            LOGGER.debug("topic: {}, body: {}", topic, body);
        });
    }
```

### 接收测试

![](C:\学习笔记\培训期第一周\Redis\截图素材\微信截图_20220722140807.png)

![](C:\学习笔记\培训期第一周\Redis\截图素材\微信截图_20220722140847.png)

![](C:\学习笔记\培训期第一周\Redis\截图素材\微信截图_20220722140859.png)