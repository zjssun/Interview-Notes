(来自老罗的项目:[EasyMeeting](https://www.bilibili.com/video/BV1XMKJzgEh8))
⚒️技术栈：SpringBoot、Mysql、Redis、Rabbitmq、netty
🧠架构图：
![](assets/SnapMeet(仿腾讯会议)----SpringBoot后端开发记录/file-20251201215200120.png)
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
