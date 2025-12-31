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
---
# 登录与注册
### 数据库
#### 表名：user_info (用户信息表)

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
1. 用户输入正确的邮箱、密码和验证码。需要检查账户状态、是否已登录(**如果上一次登录时间小于登出时间则没重复登录**)。
2. 更新登录时间，保存tokenUserInfoDto数据到Redis,给客户端返回userInfoVO。
#### AccountController.java
```java
@RequestMapping("/login")  
public ResponseVO login(@NotEmpty String checkCodeKey,  
                        @NotEmpty @Email String email,  
                        @NotEmpty @Size(max=32)String password,  
                        @NotEmpty String checkCode){  
    try {  
        if(!checkCode.equalsIgnoreCase(redisComponent.getCheckCode(checkCodeKey))){  
            throw new BusinessException("图片验证码不正确");  
        }  
        UserInfoVO userInfoVO = this.userInfoService.login(email,password);  
        return getSuccessResponseVO(userInfoVO);  
    }finally {  
        redisComponent.cleanCheckCode(checkCodeKey);  
    }  
}
```
#### UserInfoServiceImpl.java
```java
@Override  
public UserInfoVO login(String email, String password) {  
    // 检查邮箱是否已存在  
    UserInfo existUser = this.getOne(new LambdaQueryWrapper<UserInfo>().eq(UserInfo::getEmail, email));  
    if(existUser == null || !existUser.getPassword().equals(password)){  
        throw new BusinessException("邮箱或密码不正确！");  
    }  
    // 检查账号状态  
    if(UserStatusEnum.DISABLE.getStatus().equals(existUser.getStatus())){  
        throw new BusinessException("账号已被禁用！");  
    }  
    // 检查账号是否已登录  
    if(existUser.getLastLoginTime() != null && existUser.getLastOffTime() <= existUser.getLastLoginTime()){  
        throw new BusinessException("此账号已在别处登录！");  
    }  
    //更新登录时间  
    existUser.setLastLoginTime(System.currentTimeMillis());  
    this.updateById(existUser);  
    // TokenUserInfoDto是保存到Redis的  
    TokenUserInfoDto tokenUserInfoDto = StringTools.copy(existUser,TokenUserInfoDto.class);  
    String token = StringTools.encodeByMD5(tokenUserInfoDto.getUserId()+StringTools.getRandomString(Constants.LENGTH_20));  
    tokenUserInfoDto.setToken(token);  
    tokenUserInfoDto.setMyMeetingNo(existUser.getMeeting());  
    tokenUserInfoDto.setAdmin(appConfig.getAdminEmails().contains(email));  
    redisComponent.saveTokenUserInfoDto(tokenUserInfoDto);  
  
    // 返回给客户端的VO  
    UserInfoVO userInfoVO = StringTools.copy(existUser,UserInfoVO.class);  
    userInfoVO.setToken(token);  
    userInfoVO.setAdmin(tokenUserInfoDto.getAdmin());  
    return userInfoVO;  
}
```
#### RedisComponent.java
```java
//保存TokenUserInfoDto  
public void saveTokenUserInfoDto(TokenUserInfoDto tokenUserInfoDto){  
    redisUtils.setex(Constants.REDIS_KEY_WS_TOKEN+tokenUserInfoDto.getToken(),tokenUserInfoDto,Constants.REDIS_KEY_EXPIRES_DAY);  
    redisUtils.setex(Constants.REDIS_KEY_WS_TOKEN_USERID+tokenUserInfoDto.getUserId(),tokenUserInfoDto.getToken(),Constants.REDIS_KEY_EXPIRES_DAY);  
}
```

---
# Netty
### Netty WebSocket 服务端
Netty 服务端启动时，通常会调用 `channel().closeFuture().sync()` 来保持服务运行。**这行代码是阻塞的**，意味着程序会卡在这里一直等待，直到服务关闭。所以需要**开辟新线程**来避免阻塞主程序。
#### InitRun.java
**`ApplicationRunner`**：这是 Spring Boot 提供的一个接口。它的 `run` 方法会在 **Spring 容器完全初始化完成**之后被回调。
```java
@Component  
public class InitRun implements ApplicationRunner {  
  
    @Resource  
    private NettyWebSocketStarter nettyWebSocketStarter;  
  
    @Override  
    public void run(ApplicationArguments args) throws Exception {  
        new Thread(nettyWebSocketStarter).start();  
    }  
}
```
#### NettyWebSocketStarter.java
下面是Netty WebSocket 服务端启动逻辑。
##### 1.核心组件：EventLoopGroup
```java
serverBootstrap.group(boosGroup, workerGroup);
```
- **`boosGroup` (Boss)**: 负责“接待”。它只做一件事：监听端口，接受客户端的连接请求。一旦连接建立，就扔给 Worker 处理。
- **`workerGroup` (Worker)**: 负责“干活”。处理具体的 IO 操作，比如读取数据、解码、业务逻辑执行、发送数据。
##### 2. 流水线 (ChannelPipeline) —— 最关键的部分
Netty 处理网络数据就像流水线一样，`initChannel` 方法定义了这条流水线上的工序。数据进来后，会依次经过这些 Handler。
###### A. HTTP 基础支持层
````java
pipeline.addLast(new HttpServerCodec()); 
````
**作用**: 翻译官，将字节流解码成 HTTP 请求对象，或将 HTTP 响应编码成字节流。
```java
pipeline.addLast(new HttpObjectAggregator(64*1024));
```
**作用**: 组装员，HTTP 请求可能会被拆分成很多片段（Header, Chunk 1, Chunk 2...）。这个处理器把它们聚合成一个完整的 `FullHttpRequest` 对象，方便后续处理。参数 `64*1024` 限制了最大内容长度为 64KB，防止大包攻击。
###### B. 跳与保活层
```java
 pipeline.addLast(new IdleStateHandler(6,0,0));  
```
**作用**: 计时器。如果 6 秒内没有**读**到客户端发来的数据，它会触发一个 `IdleStateEvent` 事件。
```java
pipeline.addLast(new HandlerHeartBeat());  
```
**作用**: 医生。**自定义**的类。它会捕获上面抛出的 `IdleStateEvent`。如果检测到超时事件，就判定客户端掉线，关闭连接。
###### C. 业务校验层
```java
pipeline.addLast(handlerTokenValidation);  
```
**作用**: 门卫。**自定义**的类。在 WebSocket 握手成功前，校验用户的 Token。如果校验失败，会直接关闭连接，防止非法用户进入后续流程。
###### D.WebSocket 协议层
```java
 pipeline.addLast(new WebSocketServerProtocolHandler("/ws",null,true,6553,true,true,10000L));  
```
**作用**: WebSocket 总管。它帮你处理了所有复杂的 WebSocket 协议细节。
**参数详解**:

- `"ws"`: 访问路径。客户端连接地址类似于 `ws://localhost:6061/ws/?token=1a5575159602929842cbe4c81dfe0cf5`。
    
- `null`: 不指定子协议。
    
- `true`: 允许 WebSocket 扩展。
    
- `6553`: **MaxFrameSize**。允许的最大帧载荷长度。
    
- `true` (`allowMaskMismatch`): 允许掩码不匹配（宽松模式）。
    
- `true` (`checkStartsWith`): 检查路径是否以 "ws" 开头。
    
- `10000L`: 握手超时时间 10秒。
###### E. 最终业务层
```java
pipeline.addLast(handlerWebSocket);  
```
**作用**: 具体的业务员。**自定义**的类。
##### 完整代码
```java
@Component  
@Slf4j  
public class NettyWebSocketStarter implements Runnable{  
    // boos线程，用于处理连接  
    private EventLoopGroup boosGroup = new NioEventLoopGroup();  
  
    // work线程,用于处理消息。  
    private EventLoopGroup workerGroup = new NioEventLoopGroup();  
  
    @Resource  
    private HandlerTokenValidation handlerTokenValidation;  
  
    @Resource  
    private HandlerWebSocket handlerWebSocket;  
  
    @Resource  
    private AppConfig appConfig;  
  
    @Override  
    public void run() {  
        try {  
            ServerBootstrap serverBootstrap = new ServerBootstrap();  
            serverBootstrap.group(boosGroup,workerGroup);  
            serverBootstrap.channel(NioServerSocketChannel.class).handler(new LoggingHandler(LogLevel.DEBUG)).  
                    childHandler(new ChannelInitializer<Channel>() {  
                        @Override  
                        protected void initChannel(Channel channel) throws Exception {  
                            ChannelPipeline pipeline = channel.pipeline();  
                            //对http协议的支持，使用http的编码器和解码器  
                            pipeline.addLast(new HttpServerCodec());  
                            //http消息聚合器，将http消息聚合成完整FullHttpRequest  
                            pipeline.addLast(new HttpObjectAggregator(64*1024));  
                            //  
                            pipeline.addLast(new IdleStateHandler(6,0,0));  
                            //心跳处理  
                            pipeline.addLast(new HandlerHeartBeat());  
                            //token校验 拦截 channelRead事件  
                            pipeline.addLast(handlerTokenValidation);  
                            /*  
                                websocket协议处理器  
                            * String websocketPath, 路径  
                            * String subprotocols,  指定支持的子协议  
                            * boolean allowExtensions, 是否允许websocket扩展  
                            * int maxFrameSize, 设置最大帧数 6553                            * boolean allowMaskMismatch, 是否允许掩码不匹配  
                            * boolean checkStartsWith, 是否严格检查路径开头  
                            * long handshakeTimeoutMillis 握手超时  
                            * */                            pipeline.addLast(new WebSocketServerProtocolHandler("/ws",null,true,6553,true,true,10000L));  
  
                            pipeline.addLast(handlerWebSocket);  
                        }  
                    });  
                    Channel channel = serverBootstrap.bind(appConfig.getWsProt()).sync().channel();  
                    log.info("Netty服务启动成功，端口:{}",appConfig.getWsProt());  
                    channel.closeFuture().sync();  
        }catch (Exception e){  
            log.error("netty启动失败：",e);  
        }finally {  
            boosGroup.shutdownGracefully();  
            workerGroup.shutdownGracefully();  
        }  
    }  
    @PreDestroy  
    public void close(){  
        boosGroup.shutdownGracefully();  
        workerGroup.shutdownGracefully();  
    }  
}
```

#### HandlerHeartBeat.java
```java
@Slf4j  
public class HandlerHeartBeat extends ChannelDuplexHandler {  
    @Override  
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {  
        if(evt instanceof IdleStateEvent){  
            IdleStateEvent e = (IdleStateEvent) evt;  
            if(e.state() == IdleState.READER_IDLE){  
                Attribute<String> attribute = ctx.channel().attr(AttributeKey.valueOf(ctx.channel().id().toString()));  
                String userId = attribute.get();  
                log.info("用户{}没有发送心跳,断开连接",userId);  
                ctx.close();  
            }else if(e.state() == IdleState.WRITER_IDLE){  
                ctx.writeAndFlush("heart");  
            }  
        }  
    }  
}
```
#### HandlerTokenValidation.java
这个类用来在 WebSocket 握手之前拦截 HTTP 请求，进行 **Token 身份校验**。
##### 核心注解
**`@Component`**: 将该类交给 Spring 管理，这样就可以在里面使用 `@Resource` 注入 Redis 组件。
**`@ChannelHandler.Sharable`**:标记这个 Handler 实例是**线程安全**的，可以被多个 Channel（多个客户端连接）共享。
##### 主要逻辑
###### A. 提取参数
```java
String uri = fullHttpRequest.uri();
QueryStringDecoder queryStringDecoder = new QueryStringDecoder(uri);
List<String> tokens = queryStringDecoder.parameters().get("token");
```
- WebSocket 连接为：`ws://localhost:6061/ws/?token=xxxxxx`。
    
- 这里使用 `QueryStringDecoder` 解析 URL 里的参数，尝试获取名为 `token` 的值。
###### B. 空值校验
```java
if(tokens==null){
    senErrorResponse(channelHandlerContext);
    return;
}
String token = tokens.get(0);
```
- 如果 URL 里根本没带 `token` 参数，直接调用 `senErrorResponse` 发送错误响应。
###### C. Redis 业务校验
```java
TokenUserInfoDto tokenUserInfoDto = checkToken(token);
if(tokenUserInfoDto==null){
    log.info("校验token失败{}",token);
    senErrorResponse(channelHandlerContext);
    return;
}
```
- 调用 `checkToken` 去 Redis 查询这个 token 是否有效。如果无效（返回 null），则记录日志并拒绝连接。
###### D. 放行请求
```java
channelHandlerContext.fireChannelRead(fullHttpRequest.retain());
```
##### 辅助方法
**`checkToken(String token)`**:
- 简单的业务封装，去 Redis 查用户信息。

**`senErrorResponse(ChannelHandlerContext ctx)`**:
- 构建一个 HTTP 403 响应，告诉客户端 Token 无效。
	
- **`.addListener(ChannelFutureListener.CLOSE)`**: 这是一个回调。意味着“当响应数据发送完毕后，**立即断开 TCP 连接**”。这对于安全性很重要，不给非法连接留活口。
##### 完整代码：
```java
@Component  
@ChannelHandler.Sharable  
@Slf4j  
public class HandlerTokenValidation extends SimpleChannelInboundHandler<FullHttpRequest> {  
  
    @Resource  
    private RedisComponent redisComponent;  
  
    @Override  
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, FullHttpRequest fullHttpRequest) throws Exception {  
        String uri = fullHttpRequest.uri();  
        QueryStringDecoder queryStringDecoder = new QueryStringDecoder(uri);  
        List<String> tokens = queryStringDecoder.parameters().get("token");  
        if(tokens==null){  
            senErrorResponse(channelHandlerContext);  
            return;
        }  
        String token = tokens.get(0);  
        TokenUserInfoDto tokenUserInfoDto = checkToken(token);  
        if(tokenUserInfoDto==null){  
            log.info("校验token失败{}",token);  
            senErrorResponse(channelHandlerContext);  
            return;
        }  
        channelHandlerContext.fireChannelRead(fullHttpRequest.retain());  
        //TODO 连接成功后初始化工作  
  
    }  
  
    private TokenUserInfoDto checkToken(String token){  
        if(StringTools.isEmpty(token)){  
            return null;  
        }  
        return redisComponent.getTokenUserInfoDto(token);  
    }  
  
    private void senErrorResponse(ChannelHandlerContext ctx){  
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1,HttpResponseStatus.FORBIDDEN, Unpooled.copiedBuffer("token无效", CharsetUtil.UTF_8));  
        response.headers().set(HttpHeaderNames.CONTENT_TYPE,"text/plain ;charset=UTF-8");  
        response.headers().set(HttpHeaderNames.CONTENT_LENGTH,response.content().readableBytes());  
        ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);  
    }  
}
```

#### HandlerWebSocket.java
```java
@Component  
@ChannelHandler.Sharable  
@Slf4j  
public class HandlerWebSocket extends SimpleChannelInboundHandler<TextWebSocketFrame> {  
    @Override  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        log.info("有新的连接加入");  
    }  
  
    @Override  
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {  
        log.info("有连接断开");  
        // TODO 处理连接断开的逻辑  
    }  
  
    @Override  
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, TextWebSocketFrame textWebSocketFrame) throws Exception {  
        String text = textWebSocketFrame.text();  
        log.error("收到消息：{}",text);  
    }  
}
```

### Netty 连接管理器
**代码意图**：实现了一个标准的 WebSocket 会话管理器，支持单发和会议群发。

**将 Netty 的物理连接（Channel）与用户（UserId）以及业务场景（MeetingId）绑定起来**，以便后续能通过 UserId 或 MeetingId 找到对应的连接去推送消息。

**整体设计思路：**
- **握手阶段**：用户连接 -> `HandlerTokenValidation` 校验 Token。
    
- **绑定阶段**：校验通过 -> 调用 `ChannelContextUtils.addContext`。
    
    - 把 `UserId` 贴在 `Channel` 上（打标签）。
        
    - 把 `UserId` 和 `Channel` 的映射关系存入全局 Map。
        
    - 如果用户正在会议中，把 `Channel` 加入对应的会议组（ChannelGroup）。
        
- **通信阶段**：
    
    - **单聊**：通过 UserId -> 查 Map 拿到 Channel -> 发送。
        
    - **群聊/会议**：通过 MeetingId -> 查 Map 拿到 ChannelGroup -> 群发。
#### HandlerTokenValidation.java
连接成功后执行，`channelContextUtils.addContext()`。
```java
//channelHandlerContext.fireChannelRead(fullHttpRequest.retain());  
  
//连接成功后初始化工作  
channelContextUtils.addContext(tokenUserInfoDto.getUserId(),channelHandlerContext.channel());
```

#### ChannelContextUtils.java
##### 1.全局容器
```java
public static final ConcurrentHashMap<String, Channel> USER_CONTEXT_MAP = new ConcurrentHashMap<>();
public static final ConcurrentHashMap<String, ChannelGroup> MEETING_ROOM_CONTEXT_MAP = new ConcurrentHashMap<>();
```
**USER_CONTEXT_MAP**: 维护 `UserId <-> Channel` 的关系。这是实现“点对点”推送的基础。

**MEETING_ROOM_CONTEXT_MAP**: 维护 `MeetingId <-> ChannelGroup` 的关系。
- `ChannelGroup` 是 Netty 自带的一个非常强大的集合，它自动管理里面的 Channel。**重点：当 Channel 关闭（用户断线）时，ChannelGroup 会自动把它踢出去，不需要手动去 Group 里删。**

##### 2.`addContext` (核心绑定方法)
这一步做了很多事，既完成了内存中的连接注册，也同步了数据库状态，还处理了业务上的“恢复会议上下文”
```java
public void addContext(String userId, io.netty.channel.Channel channel){
    try {
        // 1. 【重点问题】给 Channel 打上 UserId 的标签
        String channelId = channel.id().toString();

        channel.attr(attributeKey).set(userId);

        // 2. 存入全局用户表
        USER_CONTEXT_MAP.put(userId, channel);

        // 3. 更新数据库：记录用户最后登录时间
        UserInfo userInfo = new UserInfo();
        userInfo.setLastLoginTime(System.currentTimeMillis());
        userInfoMapper.update(userInfo, new LambdaQueryWrapper<UserInfo>().eq(UserInfo::getUserId, userId));

        // 4. “断线重连”逻辑：检查 Redis，看用户之前是否还在某个会议里
        TokenUserInfoDto tokenUserInfoDto = redisComponent.getTokenUserInfoDtoByUserId(userId);
        if(tokenUserInfoDto.getCurrentMeetingId() == null){
            return;
        }
        // 5. 如果在会议里，直接把这个新连接加入会议组
        addMeetingRoom(tokenUserInfoDto.getCurrentMeetingId(),userId);

    }catch (Exception e){
        log.error("初始化连接失败",e);
    }
}
```
##### 3. `addMeetingRoom` (加入会议)
```java
public void addMeetingRoom(String meetingId,String userId){
    // 1. 先找到这个人的物理连接
    Channel context = USER_CONTEXT_MAP.get(userId);
    if(null==context){ return; }

    // 2. 找会议室的组，如果没有就新建一个
    ChannelGroup group = MEETING_ROOM_CONTEXT_MAP.get(meetingId);
    if(group==null){
        // GlobalEventExecutor.INSTANCE 是全局事件执行器，用于自动管理 Group 里连接的生命周期
        group = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
        MEETING_ROOM_CONTEXT_MAP.put(meetingId,group);
    }
    // 3. 把人加进去
    group.add(context); // context 就是 channel
}
```
##### 4.`sendMessage` (消息路由)
```java
public void sendMessage(MessageSendDto messageSendDto) {
    // 根据类型分发：是发给人，还是发给群
    if(MessageSend2TypeEnum.USER.getType().equals(messageSendDto.getMessageSend2Type())){
        sendMsg2User(messageSendDto);
    }else {
        sendMsg2Group(messageSendDto);
    }
}
```
发到会议室
```java
private void sendMsg2Group(MessageSendDto messageSendDto){  
    if(messageSendDto.getMeetingId()==null){  
        return;  
    }  
    ChannelGroup group = MEETING_ROOM_CONTEXT_MAP.get(messageSendDto.getMeetingId());  
    if(group==null){  
        return;  
    }  
    group.writeAndFlush(new TextWebSocketFrame(JSON.toJSONString(messageSendDto)));  
}
```
发给个人
```java
private void sendMsg2User(MessageSendDto messageSendDto){  
    if(messageSendDto.getReceiveUserId()==null){  
        return;  
    }  
    Channel channel = USER_CONTEXT_MAP.get(messageSendDto.getReceiveUserId());  
    if(channel==null){  
        return;  
    }  
    channel.writeAndFlush(new TextWebSocketFrame(JSON.toJSONString(messageSendDto)));  
}
```
##### 5. 关闭会话
```java
public void closeContext(String userId){  
    if(StringTools.isEmpty(userId)){  
        return;  
    }  
    Channel channel = USER_CONTEXT_MAP.get(userId);  
    USER_CONTEXT_MAP.remove(userId);  
    if(channel!=null){  
        channel.close();  
    }  
}
```

### 会议室
#### 需要用到的数据库表
##### **meeting_info(会议信息表)**：

| 字段名                | 数据类型     | 长度  | 非空  | 默认值  |  键  | 备注  |
| :----------------- | :------- | :-- | :-: | :--: | :-: | :-- |
| **meeting_id**     | varchar  | 10  |  是  |  -   | PRI | -   |
| **meeting_no**     | varchar  | 10  |  否  | NULL |     | -   |
| **meeting_name**   | varchar  | 100 |  否  | NULL |     | -   |
| **create_time**    | datetime | -   |  否  | NULL |     | -   |
| **create_user_id** | varchar  | 12  |  否  | NULL |     | -   |
| **join_type**      | int      | 1   |  否  | NULL |     | -   |
| **join_password**  | varchar  | 5   |  否  | NULL |     | -   |
| **start_time**     | datetime | -   |  否  | NULL |     | -   |
| **end_time**       | datetime | -   |  否  | NULL |     | -   |
| **status**         | int      | 1   |  否  | NULL |     | -   |
##### **meeting_member(会议成员表)** 

| 字段名 | 数据类型 | 长度 | 非空 | 默认值 | 键 | 备注 |
| :--- | :--- | :--- | :---: | :---: | :---: | :--- |
| **meeting_id** | varchar | 10 | 是 | - | PRI | - |
| **user_id** | varchar | 12 | 是 | - | PRI | - |
| **nick_name** | varchar | 20 | 否 | NULL | | - |
| **last_join_time** | datetime | - | 否 | NULL | | - |
| **status** | int | 1 | 否 | NULL | | - |
| **member_type** | int | 1 | 否 | NULL | | - |
| **meeting_status** | int | 4 | 否 | NULL | | - |

#### 获取用户会议历史
##### MeetingInfo.java
在生成的类里加入，`memberCount` 统计人数字段。
```java
@TableField(exist = false)  
private Integer memberCount;
```
##### MeetingInfoController.java
`/meeting/loadMeeting` 接口，返回结果是用户参与过的会议。
```java
@RequestMapping("/loadMeeting")  
public ResponseVO loadMeeting(Integer pageNo){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    Page<MeetingInfo> page = meetingInfoServiceImpl.getMeetingInfoList(tokenUserInfoDto.getUserId(),pageNo);  
    return getSuccessResponseVO(page);  
}
```
##### MeetingInfoServiceImpl.java
SQL语句分析：
1. 在meeting_member表中找出`id=用户id` 和 `status=1` 的meeting_id字段的数据有哪些，再对比在meet_info表中的meeting_id字段与筛选出来的meeting_member里meeting_id字段相同的有哪些，然后把这些相同的数据列出来。
2. 增加memberCount字段(这个是表里没有的)，查找meeting_member的meeting_id和meeting_info的meeting_id有哪些是相同的，统计相同的个数来**知道会议有多少人**。

```java
public Page<MeetingInfo> getMeetingInfoList(String userId, Integer pageNo) {  
    Page<MeetingInfo> page = new Page<>(pageNo, 15);  
  
    QueryWrapper<MeetingInfo> wrapper = new QueryWrapper<>();  
    wrapper.select(  
            "meeting_id", "meeting_no", "meeting_name", "create_time", "create_user_id", "join_type","join_password","start_time","end_time","status",  
            "(SELECT count(1) FROM meeting_member mm WHERE mm.meeting_id = meeting_info.meeting_id) AS memberCount"  
    );  
    wrapper.inSql("meeting_id",  
            "SELECT meeting_id FROM meeting_member WHERE user_id = '" + userId + "' AND status = 1");  
  
    wrapper.orderByDesc("create_time");  
    meetingInfoMapper.selectPage(page, wrapper);  
    return page;
}
```

##### ABaseController.java
**作用：** 接收前端header头里的"token"参数。用token来获取存在Redis里的tokenUserInfoDto内容。前端的token数值在登录时获得。
```java
//添加这段代码
protected TokenUserInfoDto getTokenUserInfoDto(){  
    HttpServletRequest request = ((ServletRequestAttributes)RequestContextHolder.getRequestAttributes()).getRequest();  
    String token = request.getHeader("token");  
    TokenUserInfoDto tokenUserInfoDto = redisComponent.getTokenUserInfoDto(token);  
    return  tokenUserInfoDto;  
}
```
### AOP切面
**AOP 概念**：把“权限校验”这种通用的脏活累活，从每个业务方法里抽离出来，统一在这个类里处理。
#### GlobalInterceptor.java
这段代码定义了一个 **自定义注解 (Custom Annotation)**，名为 `@GlobalInterceptor`。
**`@Target({ElementType.METHOD, ElementType.TYPE})`**

- **作用**：决定了这个注解可以放在什么位置。
    
- **`ElementType.METHOD`**：表示可以放在**方法**上面。
    
- **`ElementType.TYPE`**：表示可以放在**类、接口或枚举**上面。

**`@Retention(RetentionPolicy.RUNTIME)`**

- **作用**：决定了这个注解的生命周期。
    
- **`RUNTIME`**：表示这个注解在代码编译后、程序运行期间**依然存在**。
```java
@Target({ElementType.METHOD,ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Mapping  
public @interface GlobalInterceptor {  
	//是否校验登录
    boolean checkLogin() default true;  
    //是否校验管理员身份
    boolean checkAdmin() default false;  
}
```
#### GlobalOperationAspect.java
这段代码是 **AOP（面向切面编程）** 的具体实现。
它的作用是：**在执行 Controller 的业务方法之前，自动拦截请求，进行“登录校验”和“管理员权限校验”**。
##### 1.定义切点与通知
```java
@Before("@annotation(com.snapmeet.annotation.GlobalInterceptor)")
```
**`@Before`**: 表示**前置通知**。意思是在目标方法执行**之前**，先运行这段代码。
`@GlobalInterceptor` 这个标签，我就拦截它。
**`@annotation(...)`**: 这是切点表达式。它的意思是：只要某个方法上打了
##### 2.获取注解配置
```java
Method method = ((MethodSignature)point.getSignature()).getMethod();
GlobalInterceptor interceptor = method.getAnnotation(GlobalInterceptor.class);
```
**反射机制：**
- 通过 `point` 拿到当前正要执行的方法对象。
    
- 通过 `method.getAnnotation` 拿到方法头上那个 `@GlobalInterceptor` 注解的实例。
    
- 这样就能读取你在 Controller 上配置的参数了（比如 `checkAdmin=true`）。
##### 4. 判断逻辑
```java
if(interceptor.checkLogin() || interceptor.checkAdmin()){ 
	checkLogin(interceptor.checkAdmin()); 
}
```
- 如果注解配置了需要校验登录（默认 true）**或者**需要校验管理员权限，就调用 `checkLogin` 方法。
##### 5. 核心校验逻辑
```java
private void checkLogin(Boolean checkAdmin){
    // 1. 获取当前 HTTP 请求
    HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
    
    // 2. 拿 Token
    String token = request.getHeader("token");
    
    // 3. 查 Redis 验真伪
    TokenUserInfoDto tokenUserInfoDto = redisComponent.getTokenUserInfoDto(token);
    
    // 4. 校验是否登录
    if(tokenUserInfoDto == null){
        // 抛出自定义异常：901 未登录
        throw new BusinessException(ResponseCodeEnum.CODE_901);
    }
    
    // 5. 校验是否是管理员 (如果注解要求查的话)
    if(checkAdmin && !tokenUserInfoDto.getAdmin()){
        // 抛出自定义异常：600 无权限
        throw new BusinessException(ResponseCodeEnum.CODE_600);
    }
}
```
### 创建会议信息
#### MeetingInfoController.java
1. 前端传入 会议类型、会议名字、是否有密码和会议密码。
2. 通过之前在ABaseController写的`getTokenUserInfoDto()` 获取token信息，如果token记录了会议的id，则说明有进行的会议未结束。
3. 把需要记录的信息写到meetingInfo里并保存到数据库，然后通过 `resetTokenUserInfo()` 方法更新现有的token到Redis。
4. 把**会议的ID号**返回给前端。
```java
@RequestMapping("/quickMeeting")  
@GlobalInterceptor  
public ResponseVO quickMeeting(@NotNull Integer meetingNoType,  
                               @NotEmpty @Size(max = 100) String meetingName,  
                               @NotNull Integer joinType, @Max(5) String joinPassword){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    if(tokenUserInfoDto.getCurrentMeetingId() != null){  
        throw new BusinessException("你有未结束的会议,无法创建新的会议");  
    }  
    MeetingInfo meetingInfo = new  MeetingInfo();  
    meetingInfo.setMeetingName(meetingName);  
    meetingInfo.setMeetingNo(meetingNoType==0?tokenUserInfoDto.getMyMeetingNo(): StringTools.getMeetingNoOrMeetingId());  
    meetingInfo.setJoinType(joinType);  
    meetingInfo.setJoinPassword(joinPassword);  
    meetingInfo.setCreateUserId(tokenUserInfoDto.getUserId());  
    meetingInfoService.qucikMeeting(meetingInfo,tokenUserInfoDto.getNickName());  
  
    tokenUserInfoDto.setCurrentMeetingId(meetingInfo.getMeetingId());  
    tokenUserInfoDto.setCurrentNickName(tokenUserInfoDto.getNickName());  
    resetTokenUserInfo(tokenUserInfoDto);  
    return getSuccessResponseVO(meetingInfo.getMeetingId());  
}
```

#### MeetingInfoServiceImpl.java
```java
@Override  
public void qucikMeeting(MeetingInfo meetingInfo, String nickName) {  
    LocalDateTime curDate = LocalDateTime.now();  
    meetingInfo.setCreateTime(curDate);  
    meetingInfo.setMeetingId(StringTools.getMeetingNoOrMeetingId());  
    meetingInfo.setStartTime(curDate);  
    meetingInfo.setStatus(MeetingStatusEnum.RUNING.getStatus());  
    this.save(meetingInfo);  
}
```
#### ABaseController.java
```java
protected void resetTokenUserInfo(TokenUserInfoDto tokenUserInfoDto){  
    redisComponent.saveTokenUserInfoDto(tokenUserInfoDto);  
}
```

### 加入会议
接口`/meeting/joinMeeting` 参数：要加入的会议Id、用户Id、用户昵称、用户性别、是否打开摄像头。
```java
@RequestMapping("/joinMeeting")  
@GlobalInterceptor  
public ResponseVO joinMeeting(@NotNull Boolean videOpen){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingInfoService.joinMeeting(tokenUserInfoDto.getCurrentMeetingId(),tokenUserInfoDto.getUserId(),tokenUserInfoDto.getNickName(),tokenUserInfoDto.getSex(),videOpen);  
    return  getSuccessResponseVO(null);  
}
```
