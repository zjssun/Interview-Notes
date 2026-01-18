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

# 会议
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
#### MeetingInfoController.java
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
#### 辅助方法 (MeetingInfoServiceImpl.java)
##### addMeetingMember
保存用户信息到`MeetingMember` 数据库表。
**关键点**：使用 `insertOrUpdate`。
- 如果是第一次加入，插入一条新记录。
    
- 如果用户掉线后重连，或者退出后再进，只是更新“最后加入时间”和状态，避免产生脏数据。
```java
private void addMeetingMember(String meetingId,String userId,String nickName,Integer memberType){  
    MeetingMember meetingMember = new MeetingMember();  
    meetingMember.setMeetingId(meetingId);  
    meetingMember.setUserId(userId);  
    meetingMember.setNickName(nickName);  
    LocalDateTime localDateTime = LocalDateTime.now();  
    meetingMember.setLastJoinTime(localDateTime);  
    meetingMember.setStatus(MeetingMemberStatusEnum.NORMAL.getStatus());  
    meetingMember.setMemberType(memberType);  
    meetingMember.setMeetingStatus(MeetingStatusEnum.RUNING.getStatus());  
    meetingMemberService.insertOrUpdate(meetingMember);  
}
```
##### add2Meeting
将成员信息存入 Redis。前端展示“参会人员墙”时，通常直接读 Redis，而不是查数据库。
```java
private void add2Meeting(String meetingId,String userId,String nickName,Integer sex,Integer memberType,Boolean videoOpen){  
    MeetingMemberDTO meetingMemberDTO = new MeetingMemberDTO();  
    LocalDateTime localDateTime = LocalDateTime.now();  
    meetingMemberDTO.setUserId(userId)  
            .setNickName(nickName)  
            .setJoinTime(localDateTime)  
            .setSex(sex)  
            .setMemberType(memberType)  
            .setVideoOpen(videoOpen)  
            .setStatus(MeetingMemberStatusEnum.NORMAL.getStatus());  
    redisComponent.add2Meeting(meetingId,meetingMemberDTO);  
}
```
##### checkMeetingJoin
防止被拉黑的用户再次进入。去 Redis 查这个人在这个会议里的状态，如果是 `BLACKLIST`，直接抛异常阻断流程。
```java
private void checkMeetingJoin(String meetingId,String userId){  
    MeetingMemberDTO meetingMemberDTO = redisComponent.getMeetingMember(meetingId,userId);  
    if(meetingMemberDTO!=null&&MeetingMemberStatusEnum.BLACKLIST.getStatus().equals(meetingMemberDTO.getStatus())){  
        throw new BusinessException("你已经被拉黑无法加入会议");  
    }  
}
```
#### `joinMeeting` (主流程)
判断id是否为空，判断会议状态，如果状态为结束，抛出异常。
```java
if(StringTools.isEmpty(meetingId)){  
    throw new BusinessException(ResponseCodeEnum.CODE_600);  
}  
MeetingInfo meetingInfo = this.getOne(new LambdaQueryWrapper<MeetingInfo>().eq(MeetingInfo::getMeetingId, meetingId));  
if(meetingInfo == null || MeetingStatusEnum.FINISHED.getStatus().equals(meetingInfo.getStatus())){  
    throw new BusinessException(ResponseCodeEnum.CODE_600);  
}
```
校验用户是否为黑名单，保存用户加入会议信息到`MeetingMember` 表，保存用户加入会议信息到Redis。
```java
//校验用户  
checkMeetingJoin(meetingId,userId);  
//加入成员  
MemberTypeEnum memberTypeEnum = meetingInfo.getCreateUserId().equals(userId) ? MemberTypeEnum.COMPERE : MemberTypeEnum.NORMAL;  
addMeetingMember(meetingId,userId,nickName,memberTypeEnum.getType());  
//加入会议  
add2Meeting(meetingId,userId,nickName,sex,memberTypeEnum.getType(),videoOpen);
```
建立 WebSocket 关联,获取最新的全员列表和新进来的这个人信息，封装消息包，执行群发。
```java
//加入ws 房间  
channelContextUtils.addMeetingRoom(meetingId,userId);  
//发送ws消息  
MeetingJoinDto meetingJoinDto = new MeetingJoinDto();  
meetingJoinDto.setMeetingMemberList(redisComponent.getMeetingMemberList(meetingId));  
meetingJoinDto.setNewMember(redisComponent.getMeetingMember(meetingId,userId));  
  
MessageSendDto messageSendDto = new MessageSendDto();  
messageSendDto.setMessageType(MessageTypeEnum.ADD_MEETING_ROOM.getType());  
messageSendDto.setMeetingId(meetingId);  
messageSendDto.setMessageSend2Type(MessageSend2TypeEnum.GROUP.getType());  
messageSendDto.setMessageContent(meetingJoinDto);  
channelContextUtils.sendMessage(messageSendDto);
```
### 入会前判断
- **用户输入**：会议号 9527，密码 123456。
    
- **系统查询**：找会议号 9527 且正在进行的会议。 -> **没找到**？报错“会议不存在”。
    
- **系统检查**：用户当前是否正在开别的会？ -> **是**？报错“你有未结束的会议”。
    
- **系统检查**：用户是否被拉黑？ -> **是**？报错“无法加入”。
    
- **系统检查**：会议需要密码吗？ -> **需要** -> 密码对不对？ -> **不对**？报错“密码错误”。
    
- **状态锁定**：在 Redis 里标记该用户“正在参加会议 X”。
    
- **通行**：返回会议 ID，允许前端进行下一步 WebSocket 连接。
```java
@Override  
public String preJoinMeeting(String meetingNo, TokenUserInfoDto tokenUserInfoDto, String password) {  
    String userId = tokenUserInfoDto.getUserId();  
    LambdaQueryWrapper<MeetingInfo> wrapper = new LambdaQueryWrapper<>();  
    wrapper.eq(MeetingInfo::getMeetingNo, meetingNo)  
            .eq(MeetingInfo::getStatus, MeetingStatusEnum.RUNING.getStatus())  
            .orderByDesc(MeetingInfo::getCreateTime);  
    List<MeetingInfo> meetingInfoList = this.list(wrapper);  
    if(meetingInfoList.isEmpty()){  
        throw new BusinessException("会议不存在");  
    }  
    MeetingInfo meetingInfo = meetingInfoList.get(0);  
    if(!MeetingStatusEnum.RUNING.getStatus().equals(meetingInfo.getStatus())){  
        throw  new BusinessException("会议结束");  
    }  
    if(!StringTools.isEmpty(tokenUserInfoDto.getCurrentMeetingId())&&!meetingInfo.getMeetingId().equals(tokenUserInfoDto.getCurrentMeetingId())){  
        throw  new BusinessException("你有未结束的会议");  
    }  
    checkMeetingJoin(meetingInfo.getMeetingId(),userId);  
    if(MeetingJoinTypeEnum.PASSWORD.getType().equals(meetingInfo.getJoinType()) && !meetingInfo.getJoinPassword().equals(password)){  
        throw new BusinessException("入会密码不正确");  
    }  
  
    tokenUserInfoDto.setCurrentMeetingId(meetingInfo.getMeetingId());  
    redisComponent.saveTokenUserInfoDto(tokenUserInfoDto);  
    return meetingInfo.getMeetingId();  
}
```
### RabbitMQ实现消息的订阅发布

实现一个 **基于 RabbitMQ 的消息处理组件**。这个类就是利用 RabbitMQ 的 **广播机制** 来实现这种跨服务器的消息同步。

```java
@ConditionalOnProperty(name = Constants.MESSAGEING_HANDLE_CHANNEL_KEY, havingValue = Constants.MESSAGEING_HANDLE_CHANNEL_RABBITMQ)
```
- **`@ConditionalOnProperty`**: 这是一个开关。只有当配置文件中 `Constants.MESSAGEING_HANDLE_CHANNEL_KEY` 的值等于 "rabbitmq" 时，Spring 才会加载这个类。这意味着系统可能支持多种消息中间件（比如 Redis 或 Kafka），可以通过配置灵活切换。
```java
private ConnectionFactory factory;  
private Connection connection;  
private Channel channel;
factory = new ConnectionFactory();  
factory.setHost(host);  
factory.setPort(port);
connection = factory.newConnection();  
channel = connection.createChannel();
channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT);
```
- 这里使用了 **`FANOUT` (扇型)** 交换机。
    
- **作用**：广播。无论消息发给谁，只要绑定到这个交换机的队列，都会收到一份完整的拷贝。这是实现“群发”或“多服务器同步”的关键。
```java
String queueName = channel.queueDeclare().getQueue();
```
- **含义**：创建一个 **匿名、排他、自动删除** 的队列。
```java
channel.queueBind(queueName, EXCHANGE_NAME, "")
```
- 将这个临时队列绑定到广播交换机上。
```java
Boolean autoAck = false; // 关闭自动确认
DeliverCallback deliverCallback = (consumerTag, dellivery) -> {
    try {
        // 1. 解析消息
        String message = new String(dellivery.getBody(), "UTF-8");
        // 2. 将消息分发给 WebSocket 客户端
        channelContextUtils.sendMessage(JSON.parseObject(message, MessageSendDto.class));
        // 3. 成功后，发送确认回执 (ACK)
        channel.basicAck(dellivery.getEnvelope().getDeliveryTag(), false);
    } catch (Exception e) {
        // 4. 失败处理 (进入重试逻辑)
        handleFailMessage(channel, dellivery, queueName);
    }
};
//启动监听
channel.basicConsume(queueName,autoAck,deliverCallback,consumeTag->{});
```
- **手动 ACK**: `autoAck = false` 配合 `channel.basicAck`。这是为了保证数据不丢失。只有代码确认处理成功了，RabbitMQ 才会从队列中删除这条消息。
```java
// 1. 从消息头中获取已重试次数
Map<String, Object> headers = delivery.getProperties().getHeaders();
Integer retryCount = (Integer) headers.get(RETRY_COUNT_KEY);

// 2. 判断是否超过最大重试次数 (3次)
if (retryCount < MAX_RETRYTIME) {
    // 3. 次数 +1 并写回 Header
    headers.put(RETRY_COUNT_KEY, retryCount + 1);
    // 4. 【关键】重新发布消息到队列末尾
    // 注意：这里用的是默认交换机 ("")，直接指定 queueName，实现了“点对点”的重回队列
    channel.basicPublish("", queueName, properties, delivery.getBody());
    // 5. 确认掉旧的那条失败消息
    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
} else {
    // 6. 超过次数，彻底放弃 (拒绝消息且不重回队列)
    channel.basicReject(delivery.getEnvelope().getDeliveryTag(), false);
}
```
这是一个**手写的本地重试机制**。
- **逻辑**：如果处理失败，它不会让消息直接卡在队首（导致死循环），而是把消息取出来，计数器+1，然后重新排队到队尾。
    
- **优点**：简单有效，不会阻塞后续消息的处理。
```java
public void sendMessage(MessageSendDto messageSendDto) {  
    try(Connection connection = factory.newConnection(); Channel channel = connection.createChannel()) {  
	    // 建一个交换机，模式是：无差别广播
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT);  
        String message = JSON.toJSONString(messageSendDto);  
        //RabbitMQ 收到你的 Publish 后，会立刻把消息复制到所有绑定了这个 Exchange 的 Queue 里。
        channel.basicPublish(EXCHANGE_NAME,"",null,message.getBytes());  
    }catch (Exception e){  
        log.error("rabbitmq发送消息失败");  
    }  
}
```
发送消息

### 发送preConnection信息
这段代码是一个 **Netty WebSocket 消息处理器**(HandlerWebSocket.java) 的核心部分，专门用于处理 **WebRTC 信令转发** 或 **端对端（Peer-to-Peer）数据转发**。
主要作用是：**“收信 -> 验身 -> 改包 -> 转发”**。也就是接收客户端发来的信令数据，验证身份后，重新包装成内部消息格式，再转发给目标用户。
```java
@Override
protected void channelRead0(ChannelHandlerContext channelHandlerContext, TextWebSocketFrame textWebSocketFrame) throws Exception {
    String text = textWebSocketFrame.text(); // 获取 WebSocket 帧里的文本内容
```
- **`TextWebSocketFrame`**: Netty 专门处理 WebSocket 文本帧的对象。
    
- **`channelRead0`**: `SimpleChannelInboundHandler` 的标准回调方法，每当有数据进来时触发。
```java
// 反序列化：将 JSON 字符串转为 Java 对象 
PeerConnectionDataDto peerConnectionDataDto = JSON.parseObject(text, PeerConnectionDataDto.class);
```
**`PeerConnectionDataDto`**: 这是**前端传过来的数据格式**。看名字推测包含：

- `token`: 身份凭证。
    
- `signalData` / `signalType`: WebRTC 的 SDP 或 ICE Candidate 数据。
    
- `targetUserId`: 想发给谁？。
```java
MessageSendDto messageSendDto = new MessageSendDto();
messageSendDto.setMessageType(MessageTypeEnum.PEER.getType()); // 标记类型：这是给 Peer 的

// 提取核心业务数据（WebRTC 信令）
PeerMessageDto peerMessageDto = new PeerMessageDto();
peerMessageDto.setSignalData(peerConnectionDataDto.getSignalData());
peerMessageDto.setSignalType(peerConnectionDataDto.getSignalType());

// 填充元数据
messageSendDto.setMessageContent(peerMessageDto); // 内容载荷
messageSendDto.setMeetingId(tokenUserInfoDto.getCurrentMeetingId()); // 绑定会议室
messageSendDto.setSendUserId(tokenUserInfoDto.getUserId()); // 标记是谁发的
```
这里做了一个**格式转换**：从前端的 `PeerConnectionDataDto` 转成了后端的通用传输格式 `MessageSendDto`。
```java
messageHandler.sendMessage(messageSendDto);
```
最后调用之前写的 `MessageHandler`，将消息投递出去。
### 退出会议
#### MeetingInfoController.java
```java
@RequestMapping("/existMeeting")  
@GlobalInterceptor  
public ResponseVO exitMeeting(){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingInfoService.exitMeetingRoom(tokenUserInfoDto, MeetingMemberStatusEnum.EXIT_MEETING);  
    return getSuccessResponseVO(null);  
}
```
#### MeetingInfoServiceImpl.java
**整体业务流程**:
1. **验票：** 检查当前用户的 Token，看他是否记录了正在参加某个会议 (`meetingId`)。如果没在开会，直接忽略。
2. **退场：** 尝试在 Redis 的“当前参会人员名单”中把这个人划掉。如果 Redis 里没划掉，那就只把用户自己的状态重置为空闲，然后直接结束流程（不发通知）。
3. **重置身份**：把用户 Token 信息里的 `currentMeetingId` 设为 `null`，并保存回 Redis。这意味着该用户恢复了“自由身”，可以去参加别的会议了。
4. **广播通知：**  获取最新的、已经剔除了该用户的参会名单。打包一个 `EXIT_MEETING_ROOM` 类型的消息。**内容包含**：谁走了 (`ExitUserId`)、最新的名单 (`MeetingMemberDTOList`)、走的理由 (`ExitStatus`，是自己退的还是被踢的)。
5. **空房检查：** 统计一下现在名单里还有多少正常状态 (`NORMAL`) 的人。如果没人了，触发 `TODO` 逻辑（通常是关闭房间、回收资源）。
6. **惩罚记录：** 如果用户退出的原因是 **“被踢出 (KICK_OUT)”** 或 **“被拉黑 (BLACKLIST)”**，这属于一种**惩罚状态**。需要将这个状态写入 MySQL 数据库，确保下次他想再进来时，数据库有案底能拦住他。
```java
@Override  
public void exitMeetingRoom(TokenUserInfoDto tokenUserInfoDto, MeetingMemberStatusEnum statusEnum) {  
    String meetingId = tokenUserInfoDto.getCurrentMeetingId();  
    if(StringTools.isEmpty(meetingId)){  
        return;  
    }  
    String userId = tokenUserInfoDto.getUserId();  
    Boolean exit = redisComponent.exitMeeting(meetingId,userId,statusEnum);  
    if(!exit){  
        // Redis里没成功退出（可能本来就不在），但仍需清理用户的本地状态  
        tokenUserInfoDto.setCurrentMeetingId(null);  
        redisComponent.saveTokenUserInfoDto(tokenUserInfoDto);  
        return;  
    }  
    //清空当前正在进行的会议  
    tokenUserInfoDto.setCurrentMeetingId(null);  
    redisComponent.saveTokenUserInfoDto(tokenUserInfoDto);  
  
    MessageSendDto messageSendDto = new MessageSendDto();  
    messageSendDto.setMessageType(MessageTypeEnum.EXIT_MEETING_ROOM.getType());  
    // 获取最新名单  
    List<MeetingMemberDTO> meetingMemberDTOList = redisComponent.getMeetingMemberList(meetingId);  
    // 封装业务包  
    MeetingExitDto meetingExitDto = new MeetingExitDto();  
    meetingExitDto.setExitUserId(userId)  
            .setMeetingMemberDTOList(meetingMemberDTOList)  
            .setExitStatus(statusEnum.getStatus());  
    messageSendDto.setMessageContent(JSON.toJSON(meetingExitDto));  
    messageSendDto.setMeetingId(meetingId);  
    messageSendDto.setMessageSend2Type(MessageSend2TypeEnum.GROUP.getType());  
    // 发送  
    messageHandler.sendMessage(messageSendDto);  
  
    List<MeetingMemberDTO> onLineMember = meetingMemberDTOList.stream().filter(item-> MeetingMemberStatusEnum.NORMAL.getStatus().equals(item.getStatus())).collect(Collectors.toList());  
    if(onLineMember.isEmpty()){  
        //TODO 结束会议  
        return;  
    }  
    if(ArrayUtils.contains(new Integer[]{MeetingMemberStatusEnum.KICK_OUT.getStatus(),MeetingMemberStatusEnum.BLACKLIST.getStatus()},statusEnum.getStatus())){  
        MeetingMember meetingMember = new MeetingMember();  
        meetingMember.setStatus(statusEnum.getStatus());  
        meetingMemberService.updateByMeetingIdAndUserId(meetingMember,meetingId,userId);  
    }  
}
```
#### ChannelContextUtils.java
> 在sendMsg2Group()添加内容和辅助函数。

在 WebSocket 服务中，维护了两个重要的映射表（Map）：

1. **`USER_CONTEXT_MAP`**: 用户 ID -> 用户物理连接 (Channel)
    
2. **`MEETING_ROOM_CONTEXT_MAP`**: 会议 ID -> 会议室广播组 (ChannelGroup)
这段代码的作用是：当**有人退出会议**或者**会议彻底结束**时，及时更新这两个 Map，把人从广播组里踢出去，或者直接销毁整个广播组，**防止内存泄漏和消息误发**。

##### 场景 1：单人退出会议 (`EXIT_MEETING_ROOM`)
```java
if(MessageTypeEnum.EXIT_MEETING_ROOM.getType().equals(messageSendDto.getMessageType())){
    // 1. 解析数据，知道是谁要走
    MeetingExitDto exitDto = JSON.parseObject((String) messageSendDto.getMessageContent(),MeetingExitDto.class);
    
    // 2.【核心动作】把这个人的连接，从会议室的广播组(ChannelGroup)里移除
    removeContextFromGroup(exitDto.getExitUserId(), messageSendDto.getMeetingId());
    
    // 3. 检查房间是否空了
    // 去 Redis 查一下现在的名单，过滤出还在正常在线(NORMAL)的人
    List<MeetingMemberDTO> meetingMemberDTOList = redisComponent.getMeetingMemberList(messageSendDto.getMeetingId());
    List<MeetingMemberDTO> onLineMember = meetingMemberDTOList.stream()
            .filter(item-> MeetingMemberStatusEnum.NORMAL.getStatus().equals(item.getStatus()))
            .collect(Collectors.toList());
            
    // 4. 如果房间里没人了，直接销毁这个房间的 Context
    if(onLineMember.isEmpty()){
        removeContextGroup(messageSendDto.getMeetingId());
    }
    return;
}
```
##### 场景 2：会议结束 (`FINISH_MEETING`)
```java
if(MessageTypeEnum.FINISH_MEETING.getType().equals(messageSendDto.getMessageType())){
    // 1. 拿到所有参会人员名单
    List<MeetingMemberDTO> meetingMemberDTOList = redisComponent.getMeetingMemberList(messageSendDto.getMeetingId());
    
    // 2. 遍历所有人，挨个从广播组里移除
    for(MeetingMemberDTO meetingMemberDto : meetingMemberDTOList){
        removeContextFromGroup(meetingMemberDto.getUserId(), messageSendDto.getMeetingId());
    }
    
    // 3. 销毁会议室 Context
    removeContextGroup(messageSendDto.getMeetingId());
}
```
##### 辅助函数解析
```java
private void removeContextGroup(String meetingId){ MEETING_ROOM_CONTEXT_MAP.remove(meetingId); }
```
**作用**：从全局 Map 中直接删除这个 key。这样以后再有消息发往这个 `meetingId`，系统就会发现找不到 Group，从而避免错误投递。
```java
private void removeContextFromGroup(String userId, String meetingId){
    // 1. 先找到这个人的物理连接
    Channel context = USER_CONTEXT_MAP.get(userId);
    if(null == context){
        return; // 如果人已经断线了，Map里没他，就不用处理了
    }
    
    // 2. 找到会议室的广播组
    ChannelGroup group = MEETING_ROOM_CONTEXT_MAP.get(meetingId);
    if(group != null){
        // 3. 【关键】从组里删除这个连接
        group.remove(context);
    }
}
```
**注意**：`group.remove(context)` 只是把连接从组里拿出来，**并不会关闭用户的 WebSocket 连接**。用户依然保持着与服务器的连接（可以接收系统通知或其他消息），只是不再接收这个会议室的消息了。
### 拉黑+踢出会议+结束会议
#### MeetingInfoController.java
```java
// 踢出会议
@RequestMapping("KickOutMeeting")  
@GlobalInterceptor  
public ResponseVO kickOutMeeting(@NotEmpty String userId){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingInfoService.forceExitMeeting(tokenUserInfoDto,userId,MeetingMemberStatusEnum.KICK_OUT);  
    return getSuccessResponseVO(null);  
}  

// 黑名单
@RequestMapping("blackMeeting")  
@GlobalInterceptor  
public ResponseVO blackMeeting(@NotEmpty String userId){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingInfoService.forceExitMeeting(tokenUserInfoDto,userId,MeetingMemberStatusEnum.KICK_OUT);  
    return getSuccessResponseVO(null);  
}  
  
//获取当前正在进行的会议  
@RequestMapping("/getCurrentMeeting")  
@GlobalInterceptor  
public ResponseVO getCurrentMeeting(){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    if(StringTools.isEmpty(tokenUserInfoDto.getCurrentMeetingId())){  
        return getSuccessResponseVO(null);  
    }  
    MeetingInfo meetingInfo = this.meetingInfoService.getMeetingInfoListByMeetingId(tokenUserInfoDto.getCurrentMeetingId());  
    if(MeetingStatusEnum.FINISHED.getStatus().equals(meetingInfo.getStatus())){  
        return  getSuccessResponseVO(null);  
    }  
    return   getSuccessResponseVO(meetingInfo);  
}  
  
//结束会议  
@RequestMapping("/finishMeeting")  
@GlobalInterceptor  
public ResponseVO fishMeeting(){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingInfoService.finishMeeting(tokenUserInfoDto.getCurrentMeetingId(),tokenUserInfoDto.getUserId());  
    return getSuccessResponseVO(null);  
}
```
#### MeetingInfoServiceImpl.java
##### `forceExitMeeting` (强制踢人/拉黑)
这个方法用于主持人将捣乱的用户踢出会议（`KICK_OUT`）或者直接拉黑（`BLACKLIST`）。
**业务流程：**
- **查会议**：根据操作人（主持人）当前所在的 `meetingId` 去数据库查询会议信息。
    
- **验权限**：**关键点**。检查当前操作人的 ID (`tokenUserInfoDto.getUserId()`) 是否等于会议的创建者 ID (`CreateUserId`)。只有房主才能踢人，否则抛出异常。
    
- **找目标**：通过传入的 `userId`（倒霉蛋的ID），去 Redis 里查出这个倒霉蛋的 Token 信息。
    
- **执行踢出**：**巧妙复用**。直接调用上一段代码中解释过的 `exitMeetingRoom` 方法。
    
    - 因为 `exitMeetingRoom` 里已经写好了“Redis 清理”、“广播通知”、“由状态触发的写库惩罚（记录被踢/拉黑状态）”等逻辑，所以这里不需要重写，直接传入对应的状态枚举（`KICK_OUT` 或 `BLACKLIST`）即可。
```java
@Override  
public void forceExitMeeting(TokenUserInfoDto tokenUserInfoDto,String userId, MeetingMemberStatusEnum statusEnum) {  
  
    MeetingInfo meetingInfo = this.getOne(new LambdaQueryWrapper<MeetingInfo>().eq(MeetingInfo::getMeetingId, tokenUserInfoDto.getCurrentMeetingId()));  
    // 验权：如果你不是房主，报错
    if(!meetingInfo.getCreateUserId().equals(tokenUserInfoDto.getUserId())){  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    // 获取被踢人的信息
    TokenUserInfoDto userInfoDto = this.redisComponent.getTokenUserInfoDtoByUserId(userId);  
    // 复用退出逻辑，statusEnum 传入 KICK_OUT 或 BLACKLIST
    exitMeetingRoom(userInfoDto,statusEnum);  
}
```
##### `finishMeeting` (结束会议)
这个方法用于彻底关闭一个会议，解散所有人。
**业务流程：**
- **验权限**：同样先查库，判断操作人是否是房主。
    
    - _注意_：这里加了 `if(userId!=null)` 判断，这暗示该方法可能支持**系统自动结束**（例如定时任务扫描超时会议），此时 `userId` 为 null，跳过权限校验。
        
- **标记会议结束 (DB)**：修改 `MeetingInfo` 表，将状态改为 `FINISHED`，并记录结束时间。
    
- **广播通知**：发送类型为 `FINISH_MEETING` 的消息。
    
    - 这会触发前端强制关闭页面。
        
    - 也会触发后端 Netty 组件（你之前贴的代码）去销毁整个 `ChannelGroup`。
        
- **更新历史记录 (DB)**：将 `MeetingMember` 表里该会议所有成员的状态更新为 `FINISHED`（存档用）。
    
- **释放所有成员 (Redis)**：
    
    - 遍历当前 Redis 里的参会名单。
        
    - 把每一个人的 `currentMeetingId` 置空。
        
    - 目的：如果不做这一步，会议虽然结束了，但用户系统里还标记着“他在开会”，导致他无法加入下一个会议。
```java
@Override  
public void finishMeeting(String currentMeetingId, String userId) {  
    MeetingInfo meetingInfo = this.getOne(new LambdaQueryWrapper<MeetingInfo>().eq(MeetingInfo::getMeetingId, currentMeetingId));  
    if(userId!=null&&!meetingInfo.getCreateUserId().equals(userId)){  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    // 1. 更新主表状态
    MeetingInfo updateInfo = new MeetingInfo();  
    updateInfo.setStatus(MeetingStatusEnum.FINISHED.getStatus());  
    updateInfo.setEndTime(LocalDateTime.now());  
    this.update(updateInfo,new LambdaQueryWrapper<MeetingInfo>().eq(MeetingInfo::getMeetingId, currentMeetingId));  
	  
	// 2. 发送广播 (最重要的动作)
    MessageSendDto messageSendDto = new MessageSendDto<>();  
    messageSendDto.setMessageSend2Type(MessageSend2TypeEnum.GROUP.getType());  
    messageSendDto.setMessageType(MessageTypeEnum.FINISH_MEETING.getType());  
    messageHandler.sendMessage(messageSendDto);  
  
    MeetingMember meetingMember = new MeetingMember();  
    meetingMember.setMeetingStatus(MeetingStatusEnum.FINISHED.getStatus());  
    meetingMemberService.updateByMeeingId(meetingMember,currentMeetingId);  
  
    //TODO 更新预约会议状态  
  
    List<MeetingMemberDTO> meetingMemberDTOList = redisComponent.getMeetingMemberList(currentMeetingId);  
    // 3. 批量释放用户状态
    for (MeetingMemberDTO meetingMemberDTO:meetingMemberDTOList){  
    // 查出该用户
        TokenUserInfoDto userInfoDto = this.redisComponent.getTokenUserInfoDtoByUserId(meetingMemberDTO.getUserId());  
        userInfoDto.setCurrentMeetingId(null);  
        redisComponent.saveTokenUserInfoDto(userInfoDto);  
    }  
}
```

### 预约会议
#### MeetingReserveController.java
```java
@RequestMapping("/loadMeetingReserve")  
public ResponseVO loadMeetingReserve(){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    List<MeetingReserve> list = meetingReserveService.getReserveInfo(tokenUserInfoDto.getUserId(), MeetingReserveStatusEnum.NO_START.getStatus());  
    return getSuccessResponseVO(list);  
}  
  
@RequestMapping("/CreateMeetingReserve")  
public ResponseVO CreateMeetingReserve(MeetingReserve meetingReserve){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingReserve.setCreateUserId(tokenUserInfoDto.getUserId());  
    meetingReserveService.createMeetingReserve(meetingReserve);  
    return getSuccessResponseVO(null);  
}  
  
@RequestMapping("/delMeetingReserve")  
public ResponseVO delMeetingReserve(@NotEmpty String meetingId){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingReserveMemberService.deleteMeetingReserve(meetingId,tokenUserInfoDto.getUserId());  
    return getSuccessResponseVO(null);  
}  
  
@RequestMapping("/loadTodayMeeting")  
public ResponseVO loadTodayMeeting(){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    List<MeetingReserve> list = meetingReserveService.getTodayMeeting(tokenUserInfoDto.getUserId(),MeetingReserveStatusEnum.NO_START.getStatus());  
    return getSuccessResponseVO(list);  
}
```
#### 获取预约会议记录
传入userId和会议状态
```java
@Override  
public List<MeetingReserve> getReserveInfo(String userId, Integer status) {  
    MPJLambdaWrapper<MeetingReserve> wrapper = JoinWrappers.lambda(MeetingReserve.class)  
            .selectAll(MeetingInfo.class)  
            .select(UserInfo::getNickName)  
            .leftJoin(UserInfo.class, UserInfo::getUserId, MeetingReserve::getCreateUserId)  
            .inSql(MeetingReserve::getMeetingId, "SELECT meeting_id FROM meeting_reserve_member WHERE invite_user_id = '" + userId + "'");  
    return meetingReserveMapper.selectJoinList(MeetingReserve.class,wrapper);  
}
```
#### 创建预约会议
**流程：**初始化会议信息** -> 2. **保存主表** -> 3. **解析邀请名单** -> 4. **添加发起人** -> 5. **批量保存成员**
```java
@Override  
public void createMeetingReserve(MeetingReserve meetingReserve) {  
	// 1. 生成唯一的会议ID
    meetingReserve.setMeetingId(StringTools.getMeetingNoOrMeetingId());  
    // 2. 记录创建时间
    meetingReserve.setCreateTime(LocalDateTime.now());  
    // 3. 设置初始状态为“未开始”
    meetingReserve.setStatus(MeetingReserveStatusEnum.NO_START.getStatus());  
    // 4. 保存到 meeting_reserve 表
    this.save(meetingReserve);  
  
    List<MeetingReserveMember> reserveMemberList = new ArrayList<>();  
    // 端传过来的是 "user1,user2,user3" 这样的逗号分隔字符串
    if(!StringTools.isEmpty(meetingReserve.getInviteUserIds())) {  
        String[] inviteUserIdArray = meetingReserve.getInviteUserIds().split(",");  
        for(String userId:inviteUserIdArray){  
            MeetingReserveMember member = new MeetingReserveMember();  
            // 关联刚才生成的会议ID
            member.setMeetingId(meetingReserve.getMeetingId());  
            // 设置参会人ID
            member.setInviteUserId(userId);  
            // 加入列表
            reserveMemberList.add(member);  
        }  
    }  
    MeetingReserveMember member = new MeetingReserveMember();  
    member.setMeetingId(meetingReserve.getMeetingId());  
    member.setInviteUserId(meetingReserve.getCreateUserId());  
    reserveMemberList.add(member);  
    //批量保存成员
    meetingReserveMemberMapper.insertOrUpdate(reserveMemberList);  
 }
```
#### 查询某用户今日相关的会议预约列表
```java
@Override  
public List<MeetingReserve> getTodayMeeting(String userId, Integer status) {  
    LocalDateTime startOfDay = LocalDate.now().atStartOfDay();  
    LocalDateTime endOfDay = LocalDate.now().plusDays(1).atStartOfDay();  
    MPJLambdaWrapper<MeetingReserve> wrapper = new MPJLambdaWrapper<MeetingReserve>()  
            .selectAll(MeetingReserve.class)  
            .distinct()  
            .leftJoin(MeetingReserveMember.class, MeetingReserveMember::getMeetingId, MeetingReserve::getMeetingId)  
            //WHERE 条件 : 筛选参会人是“我”
            .eq(MeetingReserveMember::getInviteUserId, userId)  
            // WHERE 条件 : 筛选会议状态
            .eq(MeetingReserve::getStatus, status)  
            // WHERE 条件 : 筛选时间 (create_time >= 今天0点 AND create_time < 明天0点)
            .ge(MeetingReserve::getCreateTime, startOfDay)  
            .lt(MeetingReserve::getCreateTime, endOfDay);  
    List<MeetingReserve> list = this.list(wrapper);  
    return  list;  
}
```
等价的 SQL 语句
```sql
SELECT DISTINCT t.* FROM meeting_reserve t 
LEFT JOIN meeting_reserve_member t1 ON t1.meeting_id = t.meeting_id 
WHERE t1.invite_user_id = '传入的userId' 
	AND t.status = 传入的status 
	AND t.create_time >= '2026-01-06 00:00:00' AND t.create_time < '2026-01-07 00:00:00';
```
#### 删除预约会议
```java
@Override  
public void deleteMeetingReserve(String meetingId, String userId) {  
    MPJLambdaWrapper<MeetingReserveMember> wrapper = new MPJLambdaWrapper<MeetingReserveMember>()  
            .selectAll(MeetingReserveMember.class)  
            // SQL: LEFT JOIN meeting_reserve t1 ON t1.id = t.meeting_id  
            .leftJoin(MeetingReserve.class, MeetingReserve::getMeetingId, MeetingReserveMember::getMeetingId)  
            .eq(MeetingReserveMember::getMeetingId, meetingId)  
            .eq(MeetingReserve::getCreateUserId, userId);  
    long count = this.count(wrapper);  
    //只有当“你是发起人”且“会议成员数量大于 0” 时，才允许删除
    if(count>0){  
        MeetingReserveMember meetingReserveMember = new MeetingReserveMember();  
        meetingReserveMember.setMeetingId(meetingId);  
        this.remove(new MPJLambdaWrapper<MeetingReserveMember>().eq(MeetingReserveMember::getMeetingId, meetingId));  
    }  
}
```
#### 加入预约会议
把预约的会议数据加入到正式会议数据。
```java
@Override  
public void reserveJoinMeeting(String meetingId, TokenUserInfoDto tokenUserInfoDto, String joinPassword) {  
    String userId = tokenUserInfoDto.getUserId();  
    if(!StringTools.isEmpty(tokenUserInfoDto.getCurrentMeetingId()) && !meetingId.equals(tokenUserInfoDto.getCurrentMeetingId())){  
        throw new BusinessException("你有未结束的会议");  
    }  
    checkMeetingJoin(meetingId,userId);  
    MeetingReserve meetingReserve = meetingReserveService.getMeetingReserve(meetingId);  
    if(meetingReserve==null){  
        throw  new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    MeetingReserveMember member = meetingReserveMemberService.selectByMeetingIdAndUserId(meetingId,userId);  
    if (member == null) {  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    if(MeetingJoinTypeEnum.PASSWORD.getType().equals(meetingReserve.getJoinType())&&!meetingReserve.getJoinPassword().equals(joinPassword)){  
        throw  new BusinessException("入会密码不正确");  
    }  
    MeetingInfo meetingInfo = this.getOne(new LambdaQueryWrapper<MeetingInfo>().eq(MeetingInfo::getMeetingId, meetingId));  
    if(meetingInfo==null){  
        meetingInfo = new MeetingInfo();  
        meetingInfo.setMeetingName(meetingReserve.getMeetingName());  
        meetingInfo.setMeetingNo(StringTools.getMeetingNoOrMeetingId());  
        meetingInfo.setJoinType(meetingReserve.getJoinType());  
        meetingInfo.setJoinPassword(meetingReserve.getJoinPassword());  
        meetingInfo.setCreateTime(LocalDateTime.now());  
        meetingInfo.setMeetingId(meetingId);  
        meetingInfo.setStartTime(LocalDateTime.now());  
        meetingInfo.setStatus(MeetingStatusEnum.RUNING.getStatus());  
        meetingInfo.setCreateUserId(meetingReserve.getCreateUserId());  
        this.save(meetingInfo);  
    }  
    tokenUserInfoDto.setCurrentMeetingId(meetingId);  
    redisComponent.saveTokenUserInfoDto(tokenUserInfoDto);  
}
```
# 联系人
### 数据库
#### 联系人表(user_contact)

| **字段名**              | **数据类型** | **长度** | **非空** | **默认值** | **键**   | **备注** |
| -------------------- | -------- | ------ | ------ | ------- | ------- | ------ |
| **user_id**          | varchar  | 12     | 是      | -       | **PRI** | -      |
| **contact_id**       | varchar  | 12     | 是      | -       | **PRI** | -      |
| **status**           | int      | 1      | 是      | -       |         | -      |
| **last_update_time** | datetime | -      | 是      | -       |         | -      |
#### 好友申请表(user_contact_apply)
| 字段名                 | 数据类型     | 长度  | 非空  | 默认值 |  键  | 备注  |
| :------------------ | :------- | :-- | :-: | :-: | :-: | :-- |
| **apply_id**        | int      | 11  |  是  |  -  | PRI | -   |
| **apply_user_id**   | varchar  | 12  |  是  |  -  |     | -   |
| **receive_user_id** | varchar  | 12  |  是  |  -  | UNI | -   |
| **last_apply_time** | datetime | -   |  是  |  -  | MUL | -   |
| **status**          | int      | 1   |  是  |  -  |     | -   |
### 接口
判断用户的好友关系
```java
@RequestMapping("/searchContact")  
@GlobalInterceptor  
public ResponseVO searchContact(@NotEmpty String userId){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    UserInfoVO4Search userInfoVO4Search = userContactService.searchContact(tokenUserInfoDto.getUserId(),userId);  
    return getSuccessResponseVO(userInfoVO4Search);  
}
```
发送好友邀请
```java
@RequestMapping("/contactApply")  
@GlobalInterceptor  
public ResponseVO contactApply(@NotEmpty String receiverUserId){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    UserContactApply userContactApply = new UserContactApply();  
    userContactApply.setApplyUserId(tokenUserInfoDto.getUserId());  
    userContactApply.setReceiveUserId(receiverUserId);  
    Integer status = userContactApplyService.saveUserContactApply(userContactApply);  
    return getSuccessResponseVO(status);  
}
```
处理好友申请
```java
@RequestMapping("/dealWithApply")  
@GlobalInterceptor  
public ResponseVO dealWithApply(@NotEmpty String applyUserId, @NotNull Integer status){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    userContactApplyService.dealWithApply(applyUserId,tokenUserInfoDto.getUserId(),tokenUserInfoDto.getNickName(),status);  
    return getSuccessResponseVO(null);  
}
```
加载好友列表
```java
@RequestMapping("/loadContactUser")  
@GlobalInterceptor  
public ResponseVO loadContactUser(){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    List<UserContact> userContactList = userContactService.list(new LambdaQueryWrapper<UserContact>()  
            .eq(UserContact::getUserId,tokenUserInfoDto.getUserId())  
            .eq(UserContact::getStatus, UserContactStatusEnum.FRIEND.getStatus()));  
    return getSuccessResponseVO(userContactList);  
}
```
加载申请列表
```java
@RequestMapping("/loadContactApply")  
@GlobalInterceptor  
public ResponseVO loadContactApply(){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    List<UserContactApply> ApplyList = userContactApplyService.list(new LambdaQueryWrapper<UserContactApply>()  
            .eq(UserContactApply::getReceiveUserId,tokenUserInfoDto.getUserId()));  
    return getSuccessResponseVO(ApplyList);  
}
```
### 服务函数
#### 搜索用户
**核心业务流程：**
- **查用户是否存在**：
    
    - 根据 `userId` 查 `UserInfo` 表。如果查不到，直接返回 `null`。
        
- **基本信息封装**：
    
    - 如果人存在，先把 ID 和 昵称 (`NickName`) 填入返回对象 `result`。
        
- **查“是不是自己”**：
    
    - 如果 `myUserId` 等于 `userId`，说明你在搜自己。
        
    - 设置状态为 `PASS` 的负值（一种特殊的标记，前端可能据此显示“你自己”的标签）。
        
- **查询关联表 (数据准备)**：
    
    - **查询申请记录 (`contactApply`)**：查一下我有没有给对方发过好友申请。
        
    - **查询对方的好友表 (`userContact`)**：查一下对方的好友列表里有没有我。
        
    - **查询我的好友表 (`myUserContact`)**：查一下我的好友列表里有没有对方。
        
- **判定关系状态 (优先级逻辑)**：
    
    - **被拉黑判定**：如果申请记录是“拉黑” 或者 对方好友表里把我“拉黑”，直接返回 **BLACKLIST** 状态。
        
    - **已申请判定**：如果申请记录存在且状态是 `INIT`（等待验证），返回 **INIT** 状态（前端显示“等待验证”）。
        
    - **已经是好友判定**：如果 **双方** 的好友表里都有对方，且状态都是 `FRIEND`，返回 **FRIEND** 状态（前端显示“发消息”）。
        
- **陌生人**：
    
    - 如果上面都没命中，返回默认状态（通常是空或 0），前端显示“添加好友”按钮。
```java
@Override  
public UserInfoVO4Search searchContact(String myUserId, String userId) {  
  
    UserInfo userInfo = userInfoService.getOne(new LambdaQueryWrapper<UserInfo>().eq(UserInfo::getUserId,userId));  
    if(userInfo==null){  
        return null;  
    }  
    UserInfoVO4Search result = new UserInfoVO4Search();  
    result.setUserId(userInfo.getUserId());  
    result.setNickName(userInfo.getNickName());  
    if(myUserId.equals(userId)){  
        result.setStatus(-UserContactApplyStatusEnum.PASS.getStatus());  
    }  
    UserContactApply contactApply = userContactApplyService.getOne(new LambdaQueryWrapper<UserContactApply>()  
            .eq(UserContactApply::getApplyUserId,myUserId).eq(UserContactApply::getReceiveUserId,userId));  
  
    UserContact userContact = this.getOne(new LambdaQueryWrapper<UserContact>().eq(UserContact::getUserId,userId)  
            .eq(UserContact::getContactId,myUserId));  
    if(contactApply!=null&&UserContactApplyStatusEnum.BLACKLIST.getStatus().equals(contactApply.getStatus()) ||  
    userContact != null && UserContactApplyStatusEnum.BLACKLIST.getStatus().equals(userContact.getStatus())){  
        result.setStatus(UserContactApplyStatusEnum.BLACKLIST.getStatus());  
        return result;  
    }  
    if(contactApply!=null&&UserContactApplyStatusEnum.INIT.getStatus().equals(contactApply.getStatus())){  
        result.setStatus(UserContactApplyStatusEnum.INIT.getStatus());  
        return result;  
    }  
    UserContact myUserContact = this.getOne(new LambdaQueryWrapper<UserContact>().eq(UserContact::getUserId,myUserId).eq(UserContact::getContactId,userId));  
    if(userContact!=null && UserContactStatusEnum.FRIEND.getStatus().equals(userContact.getStatus())&&  
    myUserContact!=null&& UserContactStatusEnum.FRIEND.getStatus().equals(myUserContact.getStatus())){  
        result.setStatus(UserContactStatusEnum.FRIEND.getStatus());  
        return  result;  
    }  
  
    return result;  
}
```
#### 发送好友申请
**核心业务流程：**
1. **拉黑校验 (Blacklist Check)**：
    - 先去对方的好友列表里查，看我是否被对方拉黑了。如果是，直接抛出异常，阻止申请。
    
2. **自动恢复好友 (Auto-Rejoin)**：
    - 如果对方列表里我是好友状态（说明对方没删我，可能是我单方面删了对方），那么不需要发申请，直接把“我这边的关系”改回“好友”即可。
    
3. **保存/更新申请记录 (Save/Update Apply)**：
    
    - 如果以上情况都不是，说明是正常的加好友。
        
    - 查一下以前发没发过申请。
        
    - **没发过**：插入一条新记录。
        
    - **发过**：更新旧记录的时间和状态（比如以前被拒了，现在重新申请，状态改回“等待验证”）。
        
4. **发送通知 (Notify)**：
    
    - 通过 WebSocket (RabbitMQ) 给对方发一条实时消息，提示“有人加你好友”。
```java
@Override  
public Integer saveUserContactApply(UserContactApply userContactApply) {  
    UserContact userContact = userContactService.getOne(new LambdaQueryWrapper<UserContact>()  
            .eq(UserContact::getContactId,userContactApply.getReceiveUserId())  
            .eq(UserContact::getUserId,userContactApply.getApplyId()));  
    if(userContact!=null&& UserContactStatusEnum.BLACKLIST.getStatus().equals(userContact.getStatus())){  
        throw new BusinessException("对方将你拉黑");  
    }  
    if(userContact!=null&&UserContactStatusEnum.FRIEND.getStatus().equals(userContact.getStatus())){  
        UserContact updateInfo = new UserContact();  
        updateInfo.setStatus(UserContactStatusEnum.FRIEND.getStatus());  
        userContactService.update(updateInfo,new LambdaQueryWrapper<UserContact>()  
                .eq(UserContact::getUserId,userContactApply.getApplyUserId())  
                .eq(UserContact::getContactId,userContactApply.getReceiveUserId()));  
        return UserContactStatusEnum.FRIEND.getStatus();  
    }  
  
    UserContactApply apply = this.getOne(new LambdaQueryWrapper<UserContactApply>()  
            .eq(UserContactApply::getApplyUserId,userContactApply.getApplyUserId())  
            .eq(UserContactApply::getReceiveUserId,userContactApply.getReceiveUserId()));  
    if(apply==null){  
        userContactApply.setStatus(UserContactApplyStatusEnum.INIT.getStatus());  
        userContactApply.setLastApplyTime(LocalDateTime.now());  
        this.save(userContactApply);  
    }else {  
        UserContactApply update = new UserContactApply();  
        userContactApply.setStatus(UserContactApplyStatusEnum.INIT.getStatus());  
        userContactApply.setLastApplyTime(LocalDateTime.now());  
        this.update(update,new LambdaQueryWrapper<UserContactApply>()  
                .eq(UserContactApply::getApplyUserId,userContactApply.getApplyId())  
                .eq(UserContactApply::getReceiveUserId, userContactApply.getReceiveUserId()));  
    }  
  
    MessageSendDto messageSendDto = new MessageSendDto<>();  
    messageSendDto.setMessageSend2Type(MessageSend2TypeEnum.USER.getType());  
    messageSendDto.setMessageType(MessageTypeEnum.USER_CONTACT_APPLY.getType());  
    messageSendDto.setReceiveUserId(userContactApply.getReceiveUserId());  
    messageHandler.sendMessage(messageSendDto);  
    return UserContactApplyStatusEnum.INIT.getStatus();  
}
```
#### 处理好友申请
**核心业务流程：**
- **参数校验**：
    
    - 检查传入的状态 (`status`) 是否合法。你只能把它处理为“通过”、“拒绝”或“拉黑”，不能把它处理回“初始化/待处理” (`INIT`) 状态。
        
- **存在性校验**：
    
    - 确认这条申请记录真的存在（防止有人通过接口乱调，处理不存在的请求）。
        
- **通过好友 (核心逻辑)**：
    
    - 如果操作是 **“同意 (PASS)”**，则需要在好友表 (`UserContact`) 中插入 **两条记录**，实现双向好友关系。
        
- **更新申请状态**：
    
    - 修改申请表 (`UserContactApply`) 中的记录状态（变成“已同意”或“已拒绝”）。
        
- **发送通知**：
    
    - 给申请人发个即时消息，告诉他：“对方通过了你的验证”或者“对方拒绝了你”。
```java
@Override  
public void dealWithApply(String applyUserId, String userId, String nickName, Integer status) {  
    UserContactApplyStatusEnum statusEnum = UserContactApplyStatusEnum.getByStatus(status);  
    if(statusEnum==null||UserContactApplyStatusEnum.INIT == statusEnum){  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    UserContactApply apply = this.getOne(new LambdaQueryWrapper<UserContactApply>()  
            .eq(UserContactApply::getApplyUserId,applyUserId)  
            .eq(UserContactApply::getReceiveUserId,userId));  
    if(apply==null){  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
  
    if(UserContactApplyStatusEnum.PASS == statusEnum){  
        UserContact userContact = new UserContact();  
        userContact.setContactId(userId);  
        userContact.setUserId(applyUserId);  
        userContact.setStatus(UserContactStatusEnum.FRIEND.getStatus());  
        userContact.setLastUpdateTime(LocalDateTime.now());  
        userContactService.saveOrUpdate(userContact);  
  
        userContact.setUserId(userId);  
        userContact.setContactId(applyUserId);  
        userContactService.saveOrUpdate(userContact);  
    }  
  
    UserContactApply updateApply = new UserContactApply();  
    updateApply.setStatus(status);  
    this.update(updateApply,new LambdaQueryWrapper<UserContactApply>()  
            .eq(UserContactApply::getApplyUserId,applyUserId)  
            .eq(UserContactApply::getReceiveUserId, userId));  
  
    MessageSendDto sendDto = new MessageSendDto();  
    sendDto.setMessageSend2Type(MessageSend2TypeEnum.USER.getType());  
    sendDto.setMessageType(MessageTypeEnum.USER_CONTACT_DEAL_WITH.getType());  
    sendDto.setReceiveUserId(applyUserId);  
    sendDto.setSendUserNickName(nickName);  
    sendDto.setMessageContent(status);  
    messageHandler.sendMessage(sendDto);  
}
```
### 联系人入会
#### 接口
```java
//邀请加入会议  
@RequestMapping("/inviteMember")  
@GlobalInterceptor  
public ResponseVO inviteMember(@NotEmpty String selectContactIds){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingInfoService.inviteMember(tokenUserInfoDto,selectContactIds);  
    return   getSuccessResponseVO(null);  
}  
  
//接受加入会议  
@RequestMapping("/acceptInvite")  
@GlobalInterceptor  
public ResponseVO acceptInvite(@NotEmpty String meetingId){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    meetingInfoService.acceptInvite(tokenUserInfoDto,meetingId);  
    return   getSuccessResponseVO(null);  
}
```
#### 服务函数
##### 在会议中邀请好友加入
**业务流程分析：**
- **解析名单**：
    
    - 前端传过来一个逗号分隔的字符串 `selectContactIds`（比如 "userA,userB"），代码将其拆分成数组。
        
- **获取我的好友列表**：
    
    - 查询数据库 `UserContact` 表，找出当前用户 (`tokenUserInfoDto.getUserId()`) 的所有状态为 **FRIEND (好友)** 的联系人。
        
    - 提取出这些好友的 ID 列表 `contactIdList`。
        
- **权限验证 (验证是不是好友)**：
    
    - 判断前端传来的 `contactIds` 是否都在我的好友列表 `contactIdList` 里。如果不包含，说明你在邀请陌生人或非好友，抛出异常。
	
- **获取当前会议信息**：
    
    - 根据用户当前的 `currentMeetingId` 查询会议详情 `MeetingInfo`。
        
- **循环处理邀请**：
    
    - **重复邀请检查**：去 Redis 查一下这个人是不是已经在会议里了。如果已经在里面且状态正常，就 `continue` 跳过，不重复骚扰。
        
    - **写入邀请记录**：调用 `redisComponent.addInviteInfo`，在 Redis 里记一笔“A 邀请了 B”。这通常用于 B 点击链接入会时的合法性校验。
        
    - **构建消息**：封装一个 `INVITE_MEMBER_MEETING` 类型的消息，包含会议名字、邀请人名字、会议 ID。
        
    - **发送消息**：通过 WebSocket/RabbitMQ 推送给被邀请人。'
    ```java
    @Override  
public void inviteMember(TokenUserInfoDto tokenUserInfoDto, String selectContactIds) {  
    String[] contactIds = selectContactIds.split(",");  
    List<UserContact> userContactList = userContactService.list(new LambdaQueryWrapper<UserContact>()  
            .eq(UserContact::getUserId, tokenUserInfoDto.getUserId())  
            .eq(UserContact::getStatus,UserContactStatusEnum.FRIEND.getStatus()));  
    List<String> contactIdList = userContactList.stream().map(UserContact::getContactId).toList();  
    if(!contactIdList.containsAll(Arrays.asList(contactIds))){  
        throw new  BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    MeetingInfo meetingInfo = this.getOne(new LambdaQueryWrapper<MeetingInfo>()  
            .eq(MeetingInfo::getMeetingId,tokenUserInfoDto.getCurrentMeetingId()));  
    for(String contactId : contactIds){  
        MeetingMemberDTO meetingMemberDTO = redisComponent.getMeetingMember(tokenUserInfoDto.getCurrentMeetingId(),contactId);  
        if(meetingMemberDTO!=null&&MeetingMemberStatusEnum.NORMAL.getStatus().equals(meetingInfo.getStatus())){  
            continue;  
        }  
        redisComponent.addInviteInfo(tokenUserInfoDto.getCurrentMeetingId(),contactId);  
        MessageSendDto messageSendDto = new  MessageSendDto();  
        messageSendDto.setMessageType(MessageTypeEnum.INVITE_MEMBER_MEETING.getType());  
        messageSendDto.setMessageSend2Type(MessageSend2TypeEnum.USER.getType());  
        messageSendDto.setReceiveUserId(contactId);  
  
        MeetingInviteDto meetingInviteDto = new MeetingInviteDto();  
        meetingInviteDto.setMeetingName(meetingInfo.getMeetingName());  
        meetingInviteDto.setInviteUserName(tokenUserInfoDto.getNickName());  
        meetingInviteDto.setMeetingId(tokenUserInfoDto.getCurrentMeetingId());  
  
        messageSendDto.setMessageContent(JSON.toJSON(meetingInviteDto));  
        messageHandler.sendMessage(messageSendDto);  
    }  
}
    ```
##### 接受好友邀请的会议
```java
@Override  
public void acceptInvite(TokenUserInfoDto tokenUserInfoDto, String meetingId) {  
    String redisMeetingId = redisComponent.getInviteInfo(tokenUserInfoDto.getUserId(),meetingId);  
    if(redisMeetingId==null){  
        throw new   BusinessException("邀请信息已过期！");  
    }  
    tokenUserInfoDto.setCurrentMeetingId(redisMeetingId);  
    tokenUserInfoDto.setCurrentNickName(tokenUserInfoDto.getNickName());  
    redisComponent.saveTokenUserInfoDto(tokenUserInfoDto);  
}
```
### 聊天
#### ChatController.java
```java
//加载聊天内容
@RequestMapping("/loadMessage")  
@GlobalInterceptor  
public ResponseVO loadMessage(Long maxMessageId,Integer pageNo){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    String meetingId = tokenUserInfoDto.getCurrentMeetingId();  
    String tableName = TableSplitUtils.getMeetingChatMessageTable(meetingId);  
    return getSuccessResponseVO(null);  
}  
//发送聊天内容
@RequestMapping("/sendMessage")  
@GlobalInterceptor  
public ResponseVO sendMessage(String message,Integer messageType, String receiveUserId,String fileName,Long fileSize,Integer fileType){  
    TokenUserInfoDto tokenUserInfoDto = getTokenUserInfoDto();  
    MeetingChatMessage chatMessage = new MeetingChatMessage();  
    chatMessage.setMessageType(messageType);  
    chatMessage.setMessageContent(message);  
    chatMessage.setFileSize(fileSize);  
    chatMessage.setFileType(fileType);  
    chatMessage.setSendUserId(tokenUserInfoDto.getUserId());  
    chatMessage.setSendUserNickName(tokenUserInfoDto.getNickName());  
    chatMessage.setMeetingId(tokenUserInfoDto.getCurrentMeetingId());  
    if(Constants.ZERO_STR.equals(receiveUserId)){  
        chatMessage.setReceiveType(ReceiveTypeEnum.ALL.getType());  
    }else {  
        chatMessage.setReceiveType(ReceiveTypeEnum.USER.getType());  
    }  
    chatMessage.setReceiveUserId(receiveUserId);  
    meetingChatMessageService.saveChatMessage(chatMessage);  
    return getSuccessResponseVO(null);  
}
```
#### 发送聊天服务函数
```java
@Override  
public void saveChatMessage(MeetingChatMessage chatMessage) {  
    if(!ArrayUtils.contains(new Integer[]{MessageTypeEnum.CHAT_TEXT_MESSAGE.getType(),MessageTypeEnum.CHAT_MEDIA_MESSAGE.getType()},chatMessage.getMessageType())){  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    ReceiveTypeEnum receiveTypeEnum = ReceiveTypeEnum.getByStatus(chatMessage.getReceiveType());  
    if(null == receiveTypeEnum){  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    if(receiveTypeEnum==ReceiveTypeEnum.USER&& StringTools.isEmpty(chatMessage.getReceiveUserId())){  
        throw new BusinessException(ResponseCodeEnum.CODE_600);  
    }  
    MessageTypeEnum messageTypeEnum = MessageTypeEnum.getByType(chatMessage.getMessageType());  
    if(messageTypeEnum==MessageTypeEnum.CHAT_TEXT_MESSAGE){  
        if(StringTools.isEmpty(chatMessage.getMessageContent())){  
            throw new BusinessException(ResponseCodeEnum.CODE_600);  
        }  
        chatMessage.setStatus(MessageStatusEnum.SENDED.getStatus());  
    }else if(messageTypeEnum==MessageTypeEnum.CHAT_MEDIA_MESSAGE){  
        if(StringTools.isEmpty(chatMessage.getFileName()) || chatMessage.getFileSize()==null||chatMessage.getFileType()==null){  
            throw  new BusinessException(ResponseCodeEnum.CODE_600);  
        }  
        chatMessage.setFileSuffx(StringTools.getFileSuffix(chatMessage.getFileName()));  
        chatMessage.setStatus(MessageStatusEnum.SENDING.getStatus());  
    }  
  
    chatMessage.setSendTime(LocalDateTime.now().atZone(ZoneId.systemDefault()).toInstant().toEpochMilli());  
    chatMessage.setMessageId(SnowFlakeUtils.nextId());  
    String tableName = TableSplitUtils.getMeetingChatMessageTable(chatMessage.getMeetingId());  
    //TODO sql语句  
    MessageSendDto sendDto = StringTools.copyProperties(chatMessage,MessageSendDto.class);  
    if(ReceiveTypeEnum.USER == receiveTypeEnum){  
        sendDto.setMessageSend2Type(MessageSend2TypeEnum.USER.getType());  
        messageHandler.sendMessage(sendDto);  
        sendDto.setReceiveUserId(chatMessage.getSendUserId());  
        messageHandler.sendMessage(sendDto);  
    }else {  
        sendDto.setMessageSend2Type(MessageSend2TypeEnum.GROUP.getType());  
        messageHandler.sendMessage(sendDto);  
    }  
}
```