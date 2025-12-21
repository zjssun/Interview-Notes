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
### 2.String 类型操作
> 对应 Redis 的 String 结构（Key-Value）。

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
### 3. 计数器逻辑（原子性操作）
> 这部分利用了 Redis 的原子递增特性，常用于**限流、点赞、库存扣减**。

**`increment(String key)`**: 简单的 +1。
```java
public Long increment(String key) {  
    Long count = redisTemplate.opsForValue().increment(key, 1);  
    return count;  
}
```

**`incrementex(String key, long milliseconds)`**
- **带过期的计数**。
    
- **场景**：比如“限制用户 1 分钟内只能发送 1 次验证码”。
    
- 逻辑：如果是第一次计数（count == 1），说明 Key 刚创建，立刻给它设个过期时间。
```
public Long incrementex(String key, long milliseconds) {  
    Long count = redisTemplate.opsForValue().increment(key, 1);  
    if (count == 1) {   
        expire(key, milliseconds);  
    }  
    return count;  
}
```
**`decrement(String key)`**：
- **减法逻辑**。
    
- **特殊逻辑**：如果减到 0 或更小，直接**删除该 Key**。这通常用于“库存扣减”场景，卖完了就清掉。
```java
public Long decrement(String key) {  
    Long count = redisTemplate.opsForValue().increment(key, -1);  
    if (count <= 0) {  
        redisTemplate.delete(key);  
    }  
    logger.info("key:{},减少数量{}", key, count);  
    return count;  
}
```
### 4.List 类型操作（队列/栈）
> 对应 Redis 的 List 结构。

**`lpush` (Left Push)**: 从左边（头部）塞入数据。

```java
public boolean lpush(String key, V value, Long time) {  
    try {  
        redisTemplate.opsForList().leftPush(key, value);  
        if (time != null && time > 0) {  
            expire(key, time);  
        }  
        return true;  
    } catch (Exception e) {  
        e.printStackTrace();  
        return false;  
    }  
}
```
**`rpop` (Right Pop)**: 从右边（尾部）取出数据。
 - **组合效果**：`lpush` + `rpop` = **FIFO 队列（先进先出）**，就像排队买票。
```java
public V rpop(String key) {  
    try {  
        return redisTemplate.opsForList().rightPop(key);  
    } catch (Exception e) {  
        e.printStackTrace();  
        return null;  
    }  
}
```
**`lpushAll`** ：批量入队，由于是 `Left Push`（从左边/头部塞入），**Redis 里的顺序会和 Java List 的顺序相反（栈结构特性）**。
```java
public boolean lpushAll(String key, List<V> values, long time) {  
    try {  
        redisTemplate.opsForList().leftPushAll(key, values);  
        if (time > 0) {  
            expire(key, time);  
        }  
        return true;  
    } catch (Exception e) {  
        e.printStackTrace();  
        return false;  
    }  
}
```

**`getQueueList`**: 获取 List 里所有的元素（`0` 到 `-1` 代表全部）。
```java
public List<V> getQueueList(String key) {  
    return redisTemplate.opsForList().range(key, 0, -1);  
}
```
`remove` ：从列表的 **左边（头部）** 开始找，删除 **1** 个等于 `value` 的元素。
 - 如果需要改为从**右边(尾部)** 把`opsForList().remove()`参数里的1改为-1。改为0时，则是删除列表中 **所有** 等于 `value` 的元素
```java
public long remove(String key, Object value) {  
    try {  
        Long remove = redisTemplate.opsForList().remove(key, 1, value);  
        return remove;  
    } catch (Exception e) {  
        e.printStackTrace();  
        return 0;  
    }  
}
```
### 5.高级操作(批量获取与排行榜)
**`getByKeyPrefix(String keyPrifix)`**:

- **模糊查询**：获取所有以 `keyPrifix` 开头的 Key。
    
- **⚠️ 风险提示**：这里使用了 `keys` 命令。在生产环境（数据量大时）**极度危险**，会导致 Redis 阻塞卡死。建议改用 `scan` 命令。
```java
public Set<String> getByKeyPrefix(String keyPrifix) {  
    Set<String> keyList = redisTemplate.keys(keyPrifix + "*");  
    return keyList;  
}
```
**`getBatch(String keyPrifix)`**:

- **批量获取值**：先查出所有 Key，再用 `multiGet` 一次性把值都取出来，最后组装成 Map。同样受 `keys` 命令性能影响。
```java
public Map<String, V> getBatch(String keyPrifix) {  
    Set<String> keySet = redisTemplate.keys(keyPrifix + "*");  
    List<String> keyList = new ArrayList<>(keySet);  
    List<V> keyValueList = redisTemplate.opsForValue().multiGet(keyList);  
    Map<String, V> resultMap = keyList.stream().collect(Collectors.toMap(key -> key, value -> keyValueList.get(keyList.indexOf(value))));  
    return resultMap;  
}
```
**`zaddCount` / `getZSetList` (ZSet 有序集合)**:

- **排行榜功能**。
    
- `zaddCount`: 给某个元素分数 +1（比如“热搜词热度 +1”）。
```java
public void zaddCount(String key, V v) {  
    redisTemplate.opsForZSet().incrementScore(key, v, 1);  
}
```
    
- `getZSetList`: 取出分数最高的 `count` 个元素（`reverseRange` 代表倒序，分数高的排前面）。
```java
public List<V> getZSetList(String key, Integer count) {  
    Set<V> topElements = redisTemplate.opsForZSet().reverseRange(key, 0, count);  
    List<V> list = new ArrayList<>(topElements);  
    return list;  
}
```
---
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
**思路**：使用 [EasyCaptcha](https://mvnrepository.com/artifact/com.github.whvcse/easy-captcha/1.6.2)这个库生成验证码，验证码的答案保存在redis中，返回给用户的是验证码图片和保存这个答案的redis的key。

下面是获取验证码的接口代码：
#### **AccountContriller.java**
```java
@RequestMapping("/checkCode")  
public ResponseVO checkCode(){  
	//生成验证码 参数是长度和宽度
    ArithmeticCaptcha captcha = new ArithmeticCaptcha(100,42);  
    String code = captcha.text();  
    // 存到redis  
    String checkCodeKey = redisComponent.saveCheckCode(code);  
    // 验证码base64图片  
    String checkCodeBase64 = captcha.toBase64();  
    // 返回redis的key和验证码图片  
    CheckCodeVO checkCodeVO = new CheckCodeVO();  
    checkCodeVO.setCheckCode(checkCodeBase64).setCheckCodeKey(checkCodeKey);  
  
    return getSuccessResponseVO(checkCodeVO);  
}
```
#### **RedisComponent.java**
用UUID生成一段随机数作为key。返回的是这个随机数。
```java
@Resource  
private RedisUtils redisUtils;
//存储验证码答案
public String saveCheckCode(String code){  
    String checkCodeKey = UUID.randomUUID().toString();  
    redisUtils.setex(Constants.REDIS_KEY_CHECK_CODE+checkCodeKey,code,Constants.REDIS_KEY_EXPIRES_ONE_MIN);  
    return  checkCodeKey;  
}
//获取验证码答案
public String getCheckCode(String checkCodeKey){  
    return (String)redisUtils.get(Constants.REDIS_KEY_CHECK_CODE+checkCodeKey);  
}
//清除验证码  
public void cleanCheckCode(String checkCodeKey){  
    redisUtils.delete(Constants.REDIS_KEY_CHECK_CODE+checkCodeKey);  
}
```

接口返回内容：
![](assets/SnapMeet(仿腾讯会议)----SpringBoot后端开发记录/file-20251218171121420.png)
### 注册接口

**思路：** 
1. 前端需提交的参数：验证码的redis key、邮箱、用户名和密码。
2. 判断验证码是否正确，不正确则抛出异常。
3. 把数据存到数据库。
4. 清理redis中的验证码数据。
#### AccountController.java
```java
@RequestMapping("/register")  
public ResponseVO register(@NotEmpty String checkCodeKey,  
                           @NotEmpty @Email String email,  
                           @NotEmpty @Size(max=20)String password,  
                           @NotEmpty @Size(max=20) String nickName,  
                           @NotEmpty String checkCode){  
    try {  
        if(!checkCode.equalsIgnoreCase(redisComponent.getCheckCode(checkCodeKey))){  
            throw new BusinessException("图片验证码不正确");  
        }  
  
        this.userInfoService.register(email,password,nickName);  
        return getSuccessResponseVO(null);  
    }finally {  
        redisComponent.cleanCheckCode(checkCodeKey);  
    }  
}
```
#### UserInfoServiceImpl.java
```java
@Override  
public void register(String email, String nickName, String password) {  
  
    // 检查邮箱是否已存在  
    UserInfo existUser = this.getOne(new LambdaQueryWrapper<UserInfo>()  
            .eq(UserInfo::getEmail, email));  
    if (existUser != null) {  
        throw new BusinessException("该邮箱已被注册");  
    }  
	// 生成时间和用户id
    LocalDateTime curDate = LocalDateTime.now();  
    String userId = StringTools.getRandomNumber(Constants.LENGTH_12);  
    //准备数据实体  
    UserInfo userInfo = new UserInfo();  
    userInfo.setUserId(userId);  
    userInfo.setEmail(email);  
    userInfo.setNickName(nickName);  
    userInfo.setSex(2);
    //保存的密码进行md5加密
    userInfo.setPassword(StringTools.encodeByMD5(password));  
    userInfo.setCreateTime(curDate);  
    userInfo.setLastOffTime(curDate.toInstant(ZoneOffset.of("+8")).toEpochMilli());  
    userInfo.setStatus(UserStatusEnum.ENABLE.getStatus());  
    userInfo.setMeeting(StringTools.getMeetingNoOrMeetingId());
    // 保存到数据库
    this.save(userInfo);  
}
```

### 登录接口
**思路：**
1. 