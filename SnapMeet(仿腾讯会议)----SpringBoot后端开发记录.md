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
