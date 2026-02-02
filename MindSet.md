Controller 写在 digital_system-trigger/src/main/java/cn/qizheng/digital/trigger/http
DTO 写在 digital_system-api/src/main/java/cn/qizheng/digital/api/dto
Mapper / DAO / Impl 写在 digital_system-infrastructure
Service 写在 digital_system-app/src/main/java/cn/qizheng/digital/service/


用户表
```sql
DROP TABLE IF EXISTS `sys_user`;
CREATE TABLE `sys_user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  
  -- 核心登录认证字段
  `username` varchar(64) NOT NULL COMMENT '用户名(账号登录用)',
  `password` varchar(128) NOT NULL COMMENT '加密后的密码(BCrypt/MD5)',
  `salt` varchar(32) DEFAULT NULL COMMENT '加密盐(若用BCrypt可不填，若用MD5必填)',
  `mobile` varchar(20) DEFAULT NULL COMMENT '手机号(手机登录用)',
  `email` varchar(128) DEFAULT NULL COMMENT '邮箱(找回密码用)',
  
  -- 业务信息字段
  `real_name` varchar(64) DEFAULT NULL COMMENT '真实姓名',
  `user_type` tinyint(4) NOT NULL DEFAULT '3' COMMENT '用户类型(1:Super, 2:Admin, 3:User)',
  `status` tinyint(1) NOT NULL DEFAULT '1' COMMENT '状态(1:启用 0:禁用)',
  
  -- 组织架构关联 (结合你之前的项目背景)
  `employee_id` bigint(20) DEFAULT NULL COMMENT '关联的员工ID(用于身份绑定)',
  `dept_id` bigint(20) DEFAULT NULL COMMENT '部门ID(冗余字段，方便查询)',
  
  -- 审计字段
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `is_deleted` tinyint(1) DEFAULT '0' COMMENT '逻辑删除(0:未删 1:已删)',
  
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`), -- 用户名必须唯一
  UNIQUE KEY `uk_mobile` (`mobile`),     -- 手机号必须唯一 (防止手机号重复注册)
  UNIQUE KEY `uk_email` (`email`),       -- 邮箱必须唯一 (防止找回密码冲突)
  KEY `idx_status` (`status`)            -- 状态索引，方便统计启用人数
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统用户表';
```
