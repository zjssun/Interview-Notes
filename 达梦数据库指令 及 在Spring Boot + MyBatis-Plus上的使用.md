## 创建表 (Create Table)
最大的区别在于**自增主键**和**标识符引号**。

- **MySQL:** 使用 `` ` `` (反引号) 包裹表名/字段名；自增使用 `AUTO_INCREMENT`。
    
- **达梦:** 默认不加引号，或者使用 `"` (双引号)；自增使用 `IDENTITY` 关键字。

MySQL 习惯在 `CREATE TABLE` 里直接写注释，达梦虽然也支持部分内联写法，但更标准的做法是**分开写**（特别是通过工具生成 DDL 时）。

**MySQL 写法：**
```SQL
CREATE TABLE `sys_user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `username` varchar(50) DEFAULT NULL COMMENT '用户名',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`)
) COMMENT='用户表' ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
**达梦 写法：**
```SQL 
CREATE TABLE "SYS_USER" (
  "ID" BIGINT IDENTITY(1,1) NOT NULL, -- IDENTITY(种子, 增量) 代替 AUTO_INCREMENT
  "USERNAME" VARCHAR(50),
  "CREATE_TIME" TIMESTAMP,
  PRIMARY KEY ("ID")
);
COMMENT ON TABLE SYS_USER IS '用户表'; 
COMMENT ON COLUMN SYS_USER.ID IS '主键';
```

---
## 修改表
修改表结构主要使用 `ALTER TABLE` 语句。在达梦中，这部分语法主要遵循 **Oracle 风格**，与 MySQL 有一些明显的习惯差异，尤其是**改名**和**注释**的处理。
### 修改表名（Rename Table）
修改表名在达梦数据库中使用的是 standard SQL 的 `ALTER TABLE` 语法。
**MySQL写法:**
```sql
RENAME TABLE old_name TO new_name;
```
**达梦写法：**
```sql
ALTER TABLE "模式名"."旧表名" RENAME TO "新表名";
```
### 添加字段 (Add Column)
假设我们要给 `SYS_USER` 表添加一个 `STATUS` 字段，类型为 `INT`，默认值为 `0`。

- **MySQL:** 通常在添加字段的同时，直接写上 `COMMENT`（注释）和 `AFTER`（指定位置）。
    
- **达梦:**
    
    1. 不支持 `AFTER` / `BEFORE` 指定字段位置（新字段永远只能加在最后）。
        
    2. 不支持在 `ADD` 语句中直接加 `COMMENT`（需单独执行）。

**MySQL 写法：**
```sql
ALTER TABLE `sys_user` 
ADD COLUMN `status` INT DEFAULT 0 COMMENT '状态' AFTER `username`;
```
**达梦 写法：**
```sql
ALTER TABLE "SYS_USER" ADD "STATUS" INT DEFAULT 0;
COMMENT ON COLUMN "SYS_USER"."STATUS" IS '状态';
```
### 修改字段类型/长度 (Modify Column)
假设我们需要把 `USERNAME` 字段的长度从 50 扩充到 100，并设置为非空。

- **MySQL:** 使用 `MODIFY` 关键字。
    
- **达梦:** 也使用 `MODIFY` 关键字（语法基本一致）。

**MySQL & 达梦 (通用写法)：**
```sql
ALTER TABLE "SYS_USER" MODIFY "USERNAME" VARCHAR(100) NOT NULL;
```
### 修改字段名称 (Rename Column)
两者区别：
- **MySQL:** 使用 `CHANGE` 关键字，既能改名又能改类型。
    
- **达梦:** 必须使用 `RENAME COLUMN ... TO ...` 语法

**MySQL 写法：**
```sql
-- 格式: CHANGE 旧名 新名 类型
ALTER TABLE `sys_user` CHANGE `phone` `mobile` VARCHAR(20);
```
**达梦 写法：**
```sql
-- 格式: RENAME COLUMN 旧名 TO 新名 
ALTER TABLE "SYS_USER" RENAME COLUMN "PHONE" TO "MOBILE";
```
### 删除字段 (Drop Column)
语法基本一致。

**MySQL & 达梦 (通用写法)：**
```sql
ALTER TABLE "SYS_USER" DROP COLUMN "STATUS";
```
---

## 插入数据 (INSERT)
普通插入两者基本一致，但达梦有一个**事务提交**的习惯差异需要注意。

- **MySQL:** 默认 `autocommit=on`，执行完 `INSERT` 数据立刻入库。
    
- **达梦:** 达梦管理工具默认**不自动提交**。执行完 `INSERT` 后，必须手动点“提交”按钮或执行 `COMMIT;`，否则别人查不到你的数据。

**MySQL & 达梦 (通用写法):**
```sql
INSERT INTO SYS_USER (USERNAME, STATUS) VALUES ('zhangsan', 1), ('lisi', 0), ('wangwu', 1);
```
### **“不存在则插入，存在则更新” (Upsert)**
MySQL 有很好用的 `ON DUPLICATE KEY UPDATE` 或 `REPLACE INTO`，**达梦不支持这些语法**。

场景：如果 ID=1 存在则更新名字，不存在则插入
**MySQL 写法:**
```SQL
INSERT INTO SYS_USER (ID, USERNAME) VALUES (1, 'NewName') ON DUPLICATE KEY UPDATE USERNAME = 'NewName';
```
**达梦 (MERGE INTO) 写法:**
```sql
MERGE INTO SYS_USER T1 
USING (SELECT 1 AS ID, 'NewName' AS USERNAME FROM DUAL) T2
ON (T1.ID = T2.ID)
WHEN MATCHED THEN 
	UPDATE SET T1.USERNAME = T2.USERNAME 
WHEN NOT MATCHED THEN 
	INSERT (ID, USERNAME) VALUES (T2.ID, T2.USERNAME);
```
> **MyBatis-Plus 提示：** 如果你使用的是 MP 的 `saveOrUpdate()` 方法，只要配置了 `DbType.DM`，MP 会自动帮你生成上面的 `MERGE INTO` 语句，不需要手写。
### IN 与 EXISTS 操作
语法完全一致。
**MySQL & 达梦 (通用写法)：**
```sql
SELECT * FROM SYS_USER WHERE ID IN (1, 2, 3);
```
>在达梦（及 Oracle）中，如果 `IN` 后面的列表非常长（超过 1000 个），建议改用 `EXISTS` 或者将 ID 放入临时表中关联查询，否则可能会报错或性能极差。MySQL 也有类似建议，但达梦对此更敏感。
### 连表查询 (JOIN)

`JOIN` 的语法在标准 SQL 中是通用的。
**MySQL & 达梦 (通用写法)：**
```sql
--场景：查询用户及其对应的部门名称
SELECT 
    u.USERNAME, 
    d.DEPT_NAME
FROM SYS_USER u
LEFT JOIN SYS_DEPT d ON u.DEPT_ID = d.ID
WHERE u.STATUS = 1;
```

## 日期处理 (常用函数对比)
| **功能**     | **MySQL**                               | **达梦**                                            |
| ---------- | --------------------------------------- | ------------------------------------------------- |
| **当前时间**   | `NOW()` 或 `SYSDATE()`                   | `SYSDATE` 或 `CURRENT_TIMESTAMP` (DM8 也支持 `NOW()`) |
| **字符串转日期** | `STR_TO_DATE('2026-01-01', '%Y-%m-%d')` | `TO_DATE('2026-01-01', 'YYYY-MM-DD')` (Oracle 风格) |
| **日期转字符串** | `DATE_FORMAT(date, '%Y-%m-%d')`         | `TO_CHAR(date, 'YYYY-MM-DD')`                     |
| **日期加减**   | `DATE_ADD(date, INTERVAL 1 DAY)`        | `date + 1` (直接加数字，单位为天)                           |

## Spring Boot
### 添加依赖  和 配置
可以在这里查看最新版本👉[DmJdbcDriver18](https://mvnrepository.com/artifact/com.dameng/DmJdbcDriver18)
```xml
<!--   DM     -->  
<dependency>  
    <groupId>com.dameng</groupId>  
    <artifactId>DmJdbcDriver18</artifactId>  
    <version>8.1.3.140</version>  
</dependency>  
```
在application.properties中配置达梦和MyBatis-Plus 
```properties
spring.datasource.driver-class-name=dm.jdbc.driver.DmDriver  
spring.datasource.url=jdbc:dm://localhost:5236/SYSDBA?zeroDateTimeBehavior=convertToNull&useUnicode=true&characterEncoding=utf-8  
spring.datasource.hikari.username=SYSDBA  
spring.datasource.hikari.password=Dm123456  
spring.datasource.hikari.pool-name=HikariCPDatasource  
spring.datasource.hikari.minimum-idle=5  
spring.datasource.hikari.idle-timeout=180000  
spring.datasource.hikari.maximum-pool-size=10  
spring.datasource.hikari.auto-commit=true  
spring.datasource.hikari.max-lifetime=1800000  
spring.datasource.hikari.connection-timeout=30000  
spring.datasource.hikari.connection-test-query=SELECT 1

mybatis-plus.configuration.log-impl=org.apache.ibatis.logging.stdout.StdOutImpl  
mybatis-plus.configuration.map-underscore-to-camel-case=true  
mybatis-plus.global-config.db-config.id-type=auto
```
在MyBatis-Plus的分页插件中需要设置**数据库类型**(dbType)为`DbType.DM`
```java
@Configuration  
public class MybatisPlusConfig {  
    @Bean  
    public MybatisPlusInterceptor mybatisPlusInterceptor() {  
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();  
  
        // 添加分页拦截器，并指定数据库类型为 DM (达梦)  
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.DM));  
  
        return interceptor;  
    }  
}
```
### 实体、Mapper、Service、Controller
剩下的CRUD和Mysql的操作一样，就直接发代码了。
**实体（SysUser）：**
```java
@Data  
@EqualsAndHashCode(callSuper = false)  
@Accessors(chain = true)  
public class SysUser {  
    @TableId(value = "id",type = IdType.AUTO)  
    private Long id;  
    @TableField("USERNAME")  
    private String username;  
    @TableField("CREATE_TIME")  
    private Date createTime;  
    @TableField("STATUS")  
    private Integer status;  
}
```
**Mapper 接口 (SysUserMapper):**
```java
public interface SysUserMapper extends BaseMapper<SysUser> {  
}
```
**Service 层**
```java
//接口
public interface ISysUserService extends IService<SysUser> {  
	PageDto<SysUser> getPage(PageQuery pageQuery);
	Boolean updateStatus(Integer id, Integer status);
}
//实现类
@Service  
public class SysUserServiceImpl extends ServiceImpl<SysUserMapper, SysUser> implements ISysUserService {  
	@Override  
	public PageDto<SysUser> getPage(PageQuery pageQuery) {  
	    Page<SysUser> page = new Page<>(pageQuery.getCurrentPage(),pageQuery.getPageSize());  
	    if(pageQuery.getOrderBy() != null && pageQuery.getAsc() != null){  
	        OrderItem orderItem = pageQuery.getAsc() ? OrderItem.asc(pageQuery.getOrderBy()) : OrderItem.desc(pageQuery.getOrderBy());  
	        page.addOrder(orderItem);  
	    }  
	    Page<SysUser> result = this.page(page);  
	    PageDto<SysUser> pageDto = new PageDto<>();  
	    pageDto.setTotalPages(result.getPages());  
	    pageDto.setSize(result.getSize());  
	    pageDto.setRecords(result.getRecords());  
	    return pageDto;  
	}
}
```
**Controller 层 (SysUserController)**
```java
@RestController  
@RequestMapping("/user")  
public class SysUserController extends ABaseController{  
    @Resource  
    private SysUserServiceImpl sysUserServiceImpl;  
  
    @RequestMapping("/add")  
    public ResponseVO add(String username){  
        Date createTime = new Date();  
        boolean success = sysUserServiceImpl.save(new SysUser().setUsername(username).setCreateTime(createTime));  
        return getSuccessResponseVO(success);  
    }  
    
    @RequestMapping("/page")  
	public ResponseVO getPage(Long currentPage, Long pageSize,String orderBy,Boolean asc){  
	    PageQuery pageQuery = new PageQuery();  
	    pageQuery.setCurrentPage(currentPage);  
	    pageQuery.setPageSize(pageSize);  
	    pageQuery.setOrderBy(orderBy);  
	    pageQuery.setAsc(asc);  
	    PageDto<SysUser> page = sysUserServiceImpl.getPage(pageQuery);  
	    return getSuccessResponseVO(page);  
	}
	
	@RequestMapping("/status")
	public ResponseVO updateStatus(Integer id, Integer status){  
	    Boolean success = sysUserServiceImpl.updateStatus(id, status);  
	    return getSuccessResponseVO(success);  
	}
}
```
### 运行测试
#### 添加列
![](assets/达梦数据库指令%20及%20在Spring%20Boot%20+%20MyBatis-Plus上的使用/file-20260127112034994.png)
返回结果：
```json
{
    "status": "success",
    "code": 200,
    "info": "请求成功",
    "data": true,
    "time": "2026-1-27 11:08:56"
}
```
在数据库中为
![](assets/达梦数据库指令%20及%20在Spring%20Boot%20+%20MyBatis-Plus上的使用/file-20260127112340206.png)
#### 分页查询
查询第1页，每页显示5条数据，根据`CREATE_TIME` 排序，降序排列。
![](assets/达梦数据库指令%20及%20在Spring%20Boot%20+%20MyBatis-Plus上的使用/file-20260127135110997.png)
返回结果：
```json
{
    "status": "success",
    "code": 200,
    "info": "请求成功",
    "data": {
        "totalPages": 3,
        "size": 5,
        "records": [
            {
                "id": 12,
                "username": "诗雨",
                "createTime": "2026-01-27T13:12:51.868",
                "status": 1
            },
            {
                "id": 11,
                "username": "荣轩",
                "createTime": "2026-01-27T13:12:51.258",
                "status": 1
            },
            省略3条...
        ]
    }
}
```
#### 修改用户状态(Update)
