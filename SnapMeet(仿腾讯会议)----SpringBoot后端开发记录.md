(来自老罗的项目:[EasyMeeting](https://www.bilibili.com/video/BV1XMKJzgEh8))
⚒️技术栈：SpringBoot、Mysql、Redis、Rabbitmq、netty
🧠架构图：
![](assets/SnapMeet(仿腾讯会议)----SpringBoot后端开发记录/file-20251201215200120.png)
# 工具类

## Redis 读写规则配置类
**核心作用是：**防止存入 Redis 的数据变成“乱码”，让数据在 Redis 客户端中看起来是清晰可读的 普通字符串和JSON 格式。
```java
@Configuration  
public class RedisConfig<V> {  
    @Bean("redisTemplate")  
    public RedisTemplate<String, V> redisTemplate(RedisConnectionFactory factory){  
        RedisTemplate<String, V> template = new RedisTemplate<>();  
        template.setConnectionFactory(factory);  
        //设置key的序列化方式  
        template.setKeySerializer(RedisSerializer.string());  
        //设置value的序列化方式  
        template.setValueSerializer(RedisSerializer.json());  
        //设置hash的key的序列化方式  
        template.setHashKeySerializer(RedisSerializer.string());  
        //设置hash的key的序列化方式  
        template.setHashValueSerializer(RedisSerializer.json());  
        template.afterPropertiesSet();  
        return template;  
    }  
}
```
配置和不配置所保存内容的区别：

| **场景** | **存入 Redis 后的 Key**             | **存入 Redis 后的 Value**           | **评价**                   |
| ------ | ------------------------------- | ------------------------------- | ------------------------ |
| 不配置    | `\xAC\xED\x00\x05t\x00\x04user` | `\xAC\xED\x00\x05sr\x00\x0E...` | **完全看不懂**，调试极其痛苦，且数据体积大。 |
| 配置     | `"user"`                        | `{"name": "admin", "age": 18}`  | **清晰明了**，方便维护。           |
## Redis 工具类
将 Spring 提供的原生 `RedisTemplate` 操作进行二次封装，简化代码，让我们在写业务逻辑时，不需要每次都处理异常、序列化或复杂的 API 调用。
### 1.通用管理（增删查、过期）
**`delete(String... key)`**:

- **智能删除**：支持删除单个 key，也支持一次性删除多个 key（变长参数）。
    
- **实现细节**：如果是一个，直接删；如果是多个，转成 `List` 批量删。
```java
public void delete(String... key) {  
    if (key != null && key.length > 0) {  
        if (key.length == 1) {  
            redisTemplate.delete(key[0]);  
        } else {  
            redisTemplate.delete((Collection<String>) CollectionUtils.arrayToList(key));  
        }  
    }  
}
```
**`keyExists(String key)`**:

- 判断 Redis 中是否有这个 Key。
```java
public V get(String key) {  
    return key == null ? null : redisTemplate.opsForValue().get(key);  
}
```
**`expire(String key, long time)`**:
- 设置过期时间,存入时间为毫秒。

```java
public boolean expire(String key, long time) {  
    try {  
        if (time > 0) {  
            redisTemplate.expire(key, time, TimeUnit.MILLISECONDS);  
        }  
        return true;  
    } catch (Exception e) {  
        e.printStackTrace();  
        return false;  
    }  
}
```
### String 类型操作
对应 Redis 的 String 结构（Key-Value）。
**`get(String key)`**: 获取值，做了空指针保护。
```java
public V get(String key) {  
    return key == null ? null : redisTemplate.opsForValue().get(key);  
}
```
**`set(String key, V value)`**: 存入值。内部加了 `try-catch`，如果 Redis 挂了，不会导致整个程序崩溃，而是打印错误日志并返回 `false`。
```java
public boolean set(String key, V value) {  
    try {  
        redisTemplate.opsForValue().set(key, value);  
        return true;  
    } catch (Exception e) {  
        logger.error("设置redisKey:{},value:{}失败", key, value);  
        return false;  
    }  
}
```
**`setex(String key, V value, long time)`**:

- **存入并设置过期时间**。
    
- 逻辑：如果时间 `>0`，设置过期（毫秒）；如果不设置时间，就调用普通 `set`（永久保存）。
```java
public boolean setex(String key, V value, long time) {  
    try {  
        if (time > 0) {  
            redisTemplate.opsForValue().set(key, value, time, TimeUnit.MILLISECONDS);  
        } else {  
            set(key, value);  
        }  
        return true;  
    } catch (Exception e) {  
        logger.error("设置redisKey:{},value:{}失败", key, value);  
        return false;  
    }  
}
```
### 4. 计数器逻辑（原子性操作）

# 登录注册
## 数据库
### 表名：user_info (用户信息表)

| 字段名 | 数据类型 | 长度 | 非空 | 默认值 | 键 | 备注 |
| :--- | :--- | :--- | :---: | :---: | :---: | :--- |
| **user_id** | varchar | 12 | 是 | - | PRI | 用户ID |
| **email** | varchar | 50 | 是 | - | UNI | 邮箱 |
| **nick_name** | varchar | 20 | 是 | - | | 昵称 |
| **sex** | tinyint | 1 | 是 | - | | 0:女 1:男 2:保密 |
| **password** | varchar | 32 | 是 | - | | 密码 |
| **status** | tinyint | 1 | 是 | - | | 状态 |
| **create_time** | datetime | - | 否 | NULL | | 创建时间 |
| **last_login_time** | bigint | 20 | 否 | NULL | | 最后登录时间 |
| **last_off_time** | bigint | 20 | 否 | NULL | | 最后离开时间 |
| **meeting** | varchar | 10 | 否 | NULL | | 个人会议号 |
#### 索引信息 (Indexes)

- **主键 (Primary Key):** `user_id`
    
- **唯一索引 (Unique Index):** `idx_key_email` (字段: `email`, 算法: BTREE)
### 验证码
**思路**：使用 [EasyCaptcha](https://mvnrepository.com/artifact/com.github.whvcse/easy-captcha/1.6.2)这个库生成验证码，
