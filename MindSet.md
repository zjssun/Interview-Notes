你好，我正在开发一个基于 Java Spring Boot 的多模块项目，采用 COLA (DDD) 架构。
请记住我的项目结构和依赖规则，后续的代码生成和问题解答请严格遵循此规范：

### 1. 技术栈
- 语言：Java 17+
- 框架：Spring Boot
- 数据库：MySQL + 原生 MyBatis (非 MyBatis-Plus)
- 缓存：Redis

### 2. 模块结构与职责 (Module Structure)
我的项目名为 `digital_system`，包含以下模块：

- **digital_system-types (公共层)**
  - 职责：存放枚举 (Enums)、通用异常 (AppException)、通用响应 (Response)、全链路共享对象。
  - 特殊规则：`LoginUserDTO` 放在这里，以便 Domain 和 Infra 层都能访问。

- **digital_system-domain (领域层/核心)**
  - 职责：核心业务逻辑。不依赖 API、App 或 Infra。
  - 内容：
    - `model/entity`: 领域实体 (如 `UserEntity`)，包含核心逻辑 (如 `checkPassword`)。
    - `adapter/repository`: 接口定义 (如 `UserGateway`)。
    - `service`: 领域服务 (如 `UserAuthDomainService`)，负责业务编排。

- **digital_system-api (接口定义层)**
  - 职责：定义前端交互契约。
  - 内容：`dto` (如 `UserLoginCmd`), `vo` (如 `LoginDataVO`)。

- **digital_system-infrastructure (基础设施层)**
  - 职责：实现技术细节。
  - 内容：
    - `dao/po`: 数据库对象 (如 `UserPO`)。
    - `dao`: MyBatis Mapper 接口。
    - `adapter/repository`: 实现 Domain 层的接口 (如 `UserGatewayImpl`)。
    - `utils`: Redis 工具类。
  - 特殊规则：MyBatis XML 文件必须非空，且路径配置正确。

- **digital_system-trigger (触发器/Web层)**
  - 职责：接收 HTTP 请求，解析参数。
  - 内容：Controller (如 `AuthController`)。
  - 依赖：调用 `domain` 层的 Service，使用 `api` 层的 DTO。

- **digital_system-app (启动层)**
  - 职责：Spring Boot 启动类 (`Application.java`)，全局配置 (`application.yml`)。
  - 依赖：聚合所有模块，是项目的入口。

### 3. 关键依赖规则 (Dependency Rules)
必须遵守单向依赖，防止循环引用：
App -> Trigger -> Domain <- Infrastructure
(所有层都可以依赖 Types)

### 4. 这里的开发习惯
- **Entity vs PO vs DTO**：严禁混用。Controller 负责将 DTO 转为 Entity 传给 Domain；Infra 负责将 Entity 转为 PO 存库。
- **MyBatis**：使用 XML 编写 SQL，XML 文件不能为空。
- **启动扫描**：启动类使用了 `@ComponentScan("cn.qizheng.digital")` 以确保扫描到所有模块。

如果理解了，请回复“项目结构已加载”，然后等待我的具体问题。