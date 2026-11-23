---
kind: configuration_system
name: MeterSphere 3.x 配置系统：后端 Spring Boot + 前端 Vite 双端分层配置
category: configuration_system
scope:
    - '**'
source_files:
    - backend/app/src/main/java/io/metersphere/Application.java
    - backend/app/src/main/resources/commons.properties
    - backend/services/api-test/src/main/java/io/metersphere/api/config/JmeterProperties.java
    - backend/services/system-setting/src/main/java/io/metersphere/system/config/MinioProperties.java
    - frontend/.env.development
    - frontend/.env.production
    - frontend/config/vite.config.base.ts
    - frontend/src/config/settings.json
---

## 1. 总体方案

MeterSphere 3.x 是 Maven 聚合的单体工程，后端基于 Spring Boot，前端基于 Vue3 + Vite。配置体系分为两条线：

- **后端**：Spring Boot 默认 `application.yml`（本分支未提供）+ 自定义 `commons.properties` + 外部覆盖文件 `/opt/metersphere/conf/metersphere.properties`，并通过 `@EnableConfigurationProperties` 将业务属性绑定到强类型 POJO。
- **前端**：Vite 通过 `.env.*` 注入 `VITE_*` 环境变量，运行时 UI 偏好与功能开关集中在 `src/config/settings.json`，构建期常量由 Vite `define` 注入。

## 2. 后端配置加载链

入口类 `backend/app/src/main/java/io/metersphere/Application.java` 通过 `@PropertySource` 显式注册两个属性源（按顺序合并，后者优先）：

```java
@PropertySource(value = {
    "classpath:commons.properties",
    "file:/opt/metersphere/conf/metersphere.properties",
}, encoding = "UTF-8", ignoreResourceNotFound = true)
```

因此配置优先级为：**JVM 参数 / Spring Boot 命令行参数 > `/opt/metersphere/conf/metersphere.properties` > `commons.properties`**。`ignoreResourceNotFound=true` 允许生产环境不挂载该目录时仍启动。

### 2.1 内置默认值：`commons.properties`

`backend/app/src/main/resources/commons.properties` 集中了所有默认值，包括：

- 应用名、端口（`server.port=8081`、管理端口 `management.server.port=7071`）、Gzip 压缩
- Quartz 调度器（`quartz.scheduler-name=msScheduler`，线程数 10）
- HikariCP 连接池（最大 100、最小空闲 10、超时等）
- MyBatis 缓存/懒加载/驼峰映射等全局设置
- Flyway 迁移（`baseline-on-migrate=true`，表 `metersphere_version`）
- Kafka（bootstrap-servers 使用 `${kafka.bootstrap-servers}` 占位符，需外部覆盖）
- Redisson session（`spring.redis.redisson.file=file:/opt/metersphere/conf/redisson.yml`）
- i18n basename（`i18n/commons,i18n/api,...,i18n/system,i18n/dashboard`）
- Swagger 关闭、批下载大小 `metersphere.file.batch-download-max=600MB`、默认 locale `zh_CN`
- JMeter 路径 `jmeter.home=/opt/jmeter`、上传限制 `1024MB`

### 2.2 外部可覆盖配置：`/opt/metersphere/conf/metersphere.properties`

这是部署侧唯一的外部配置文件，用于覆盖数据库、Kafka、MinIO、Redis 等敏感或环境相关参数。Docker 镜像中通常以卷挂载方式提供。

### 2.3 强类型属性绑定

通过 `@EnableConfigurationProperties` 在 Application 上启用两类 POJO：

- `io.metersphere.api.config.JmeterProperties`（前缀 `jmeter`，字段 `home`）
- `io.metersphere.system.config.MinioProperties`（前缀 `minio`，字段 `endpoint/accessKey/secretKey`）

这些类使用 Lombok `@Getter/@Setter` + Spring `@ConfigurationProperties(prefix = ...)`，是后端唯一的类型安全配置入口。其他散落的 `@Value` 用法（如 `embedded.mockserver.host/port`、`metersphere.file.batch-download-max`）直接取键值，无类型校验。

### 2.4 其他配置来源

- **日志**：`logback-spring.xml`（Spring Boot profile 支持）
- **Banner**：`banner.txt`
- **Redisson**：独立 YAML 文件 `file:/opt/metersphere/conf/redisson.yml`
- **Bundle 国际化**：`BundleConfig.java` 通过 `@Value("${spring.messages.default-locale}")` 读取 locale 与 basename

## 3. 前端配置体系

### 3.1 构建期环境变量：`.env.*`

Vite 自动加载 `frontend/.env.development` 与 `frontend/.env.production`，仅暴露 `VITE_*` 前缀变量：

- `.env.development`：`VITE_API_BASE_URL='front'`、`VITE_DEV_DOMAIN='http://172.16.200.18:8081/'`
- `.env.production`：`VITE_API_BASE_URL=''`（空字符串表示相对路径）

### 3.2 构建期常量：`vite.config.base.ts`

通过 `define` 注入编译期常量：

```ts
define: {
  'process.env': {},
  '__VUE_OPTIONS_API__': true,
  '__VUE_PROD_DEVTOOLS__': false,
  '__VUE_PROD_HYDRATION_MISMATCH_DETAILS__': true,
}
```

同时通过 `less.modifyVars` 引入 `src/assets/style/var.less` 作为主题变量入口。

### 3.3 运行时 UI 配置：`src/config/settings.json`

集中存放用户偏好与界面开关（theme、navbar、menu、themeColor、menuWidth、collapsedWidth、pageSize 等），由前端状态层在运行时读写，不属于构建产物。

### 3.4 模块级静态配置

`frontend/src/config/` 下按业务域拆分 TS 文件（`apiTest.ts`、`caseManagement.ts`、`chartTheme.ts`、`common.ts`、`pathMap.ts`、`template.ts`、`testPlan.ts`、`workbench.ts`、`modelConfig.ts`、`fileManagement.ts`），作为各模块的常量/枚举型配置，非运行时可变。

## 4. 约定与约束

- **后端默认值统一放在 `commons.properties`**，不要在各 service 模块重复声明基础 Spring/Hikari/Kafka/Flyway 配置；需要覆盖的项应放入 `/opt/metersphere/conf/metersphere.properties`。
- **外部可覆盖文件路径固定为 `/opt/metersphere/conf/metersphere.properties`**，由 `Application.java` 硬编码，不可通过注解修改。
- **Redisson 配置路径固定为 `/opt/metersphere/conf/redisson.yml`**，由 `commons.properties` 中的 `spring.redis.redisson.file` 指定。
- **对外暴露的配置键必须先在 `commons.properties` 给出默认值**，再在 `metersphere.properties` 中覆盖——否则容器未挂载 conf 目录时会因占位符（如 `${kafka.bootstrap-servers}`）解析失败而启动异常。
- **强类型属性优先使用 `@ConfigurationProperties`**（见 `JmeterProperties`、`MinioProperties`），避免散落 `@Value`；当前代码库中仍有大量 `@Value` 直接取值，属于遗留模式。
- **前端环境变量必须以 `VITE_` 前缀命名**，这是 Vite 的强制约定，非此前缀的变量不会被注入到客户端代码。
- **前端运行期 UI 偏好存放在 `src/config/settings.json`**，而非 `.env` 或代码常量中。
- **Spring Boot 主启动类排除了 `QuartzAutoConfiguration`、`LdapAutoConfiguration`、`Neo4jAutoConfiguration`**，意味着 Quartz/LDAP/Neo4j 的启用由业务模块自行装配，不在主应用层做条件化配置。