DTO、vo、Cmd 写在 digital_system-api/src/main/java/cn/qizheng/digital/api/dto

SQL Mapper 写在 digital_system-infrastructure/src/main/java/cn/qizheng/digital/infrastructure/dao/mapper
PO 写在 digital_system-infrastructure/src/main/java/cn/qizheng/digital/infrastructure/dao/po

Entity 写在 digital_system-domain/src/main/java/cn/qizheng/digital/domain/
Gateway接口 写在 digital_system-domain/src/main/java/cn/qizheng/digital/domain/
Gateway实现 写在 digital_system-infrastructure/src/main/java/cn/qizheng/digital/infrastructure/adapter/repository

Service 写在 digital_system-app/src/main/java/cn/qizheng/digital/service/

Controller 写在 digital_system-trigger/src/main/java/cn/qizheng/digital/trigger/http


用户表
```sql
CREATE TABLE `sys_user` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `username` varchar(64) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '用户名(账号登录用)',
  `password` varchar(128) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '加密后的密码',
  `salt` varchar(32) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '加密盐',
  `mobile` varchar(20) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '手机号(手机登录用)',
  `email` varchar(128) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '邮箱(找回密码用)',
  `real_name` varchar(64) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '真实姓名',
  `position` varchar(64) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '岗位',
  `user_type` tinyint NOT NULL DEFAULT '3' COMMENT '用户类型(1:Super, 2:Admin, 3:User)',
  `status` tinyint(1) NOT NULL DEFAULT '1' COMMENT '状态(1:启用 0:禁用)',
  `employee_id` varchar(20) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '关联的员工ID(用于身份绑定)',
  `dept_id` bigint DEFAULT NULL COMMENT '部门ID(冗余字段，方便查询)',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `is_deleted` tinyint(1) DEFAULT '0' COMMENT '逻辑删除(0:未删 1:已删)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`),
  UNIQUE KEY `uk_mobile` (`mobile`),
  UNIQUE KEY `uk_email` (`email`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB AUTO_INCREMENT=1006 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='系统用户表';
```



Prompt:
