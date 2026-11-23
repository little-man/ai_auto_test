---
kind: logging_system
name: 基于 Logback + SLF4J 的分级异步日志系统
category: logging_system
scope:
    - '**'
source_files:
    - backend/app/src/main/resources/logback-spring.xml
    - backend/app/src/main/resources/commons.properties
    - backend/framework/sdk/src/main/java/io/metersphere/sdk/util/LogUtils.java
    - backend/framework/plugin/plugin-sdk/src/main/java/io/metersphere/plugin/sdk/util/PluginLogUtils.java
---

## 1. 使用的框架与工具

后端采用 **SLF4J + Logback** 作为日志门面与实现，通过 Spring Boot 的 `logback-spring.xml` 完成初始化。前端（Vue3）未引入独立日志库，业务日志集中在后端。

- 日志门面：`org.slf4j.Logger` / `LoggerFactory`
- 日志实现：`ch.qos.logback.core.*`、`ch.qos.logback.classic.*`
- 配置入口：`backend/app/src/main/resources/logback-spring.xml`
- 运行时属性：`backend/app/src/main/resources/commons.properties`（定义 `logging.file.path=/opt/metersphere/logs/metersphere`）

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `backend/app/src/main/resources/logback-spring.xml` | Logback 主配置，定义 Appender、RollingPolicy、Logger 级别 |
| `backend/app/src/main/resources/commons.properties` | 提供 `logging.file.path`、`logger.level`、`logger.sql.level` 等可覆盖属性 |
| `backend/framework/sdk/src/main/java/io/metersphere/sdk/util/LogUtils.java` | 框架层统一日志封装，提供 `debug/info/warn/error` 静态方法 |
| `backend/framework/plugin/plugin-sdk/src/main/java/io/metersphere/plugin/sdk/util/PluginLogUtils.java` | 插件 SDK 侧的日志封装，API 与 `LogUtils` 对称 |

## 3. 架构与约定

### 3.1 输出目标（Sinks）

- **ConsoleAppender**：控制台输出，pattern 为 `%d %5p %40.40c:%4L - %m%n`。
- **RollingFileAppender**：按级别拆分到四个文件：
  - `debug.log` — 仅 DEBUG
  - `info.log` — INFO 及以上（ThresholdFilter=INFO）
  - `warn.log` — 仅 WARN
  - `error.log` — 仅 ERROR
- 所有文件均使用 `SizeAndTimeBasedRollingPolicy`：
  - 文件名模板：`${logging.file.path}/history/<level>.%d{yyyyMMdd}-%i.log`
  - 单文件上限：`50MB`
  - 保留天数：`${logger.max.history:-30}`（默认 30 天）
- 每个 FileAppender 都包裹一层 `AsyncAppender`（queueSize=10000），对 error/warn 开启 `includeCallerData=true`，避免同步 I/O 阻塞业务线程。

### 3.2 Logger 级别策略

- `root level="INFO"`，附加 `infoAsyncAppender` 和 `console`。
- `io.metersphere` 包下 logger 默认 `${logger.level:INFO}`，输出到 info/warn/error 三个异步 Appender。
- 各 Mapper 包（`io.metersphere.*.mapper`）单独声明，级别由 `${logger.sql.level}` 控制，仅输出到 console，便于调试 SQL。
- `io.metersphere.Application` 单独声明，additivity=false，只输出到 `infoAsyncAppender`。

### 3.3 调用方接入方式

代码中主要存在两种风格：

1. **直接使用 SLF4J**：在业务类中 `private static final Logger log = LoggerFactory.getLogger(...)`，然后调用 `log.info/debug/warn/error`（如 `MsSaveService`、`LocalRealm`、`RingBuffer`、`DockerUtils`、`NamingThreadFactory` 等）。
2. **通过框架封装**：`LogUtils` 与 `PluginLogUtils` 提供静态方法，内部通过反射 `StackTraceElement` 自动推断调用类名与方法名，并以 `Method[xxx]` 前缀拼接消息体（见 `getMsg` / `getLogMethod`）。

注意：`PluginLogUtils.info(Object)` 在写日志的同时还执行了 `System.out.println(msg)`，这是插件 SDK 特有的行为，与 `LogUtils` 不同。

### 3.4 结构化字段

当前日志格式是纯文本，没有 JSON 结构化字段。Encoder pattern 固定为：

```
%d [%thread] %-5level %logger{36} %line - %msg%n
```

即包含时间、线程名、级别、logger 名、行号、消息体；无 MDC、traceId、userId 等上下文字段。

## 4. 约定与约束

- **日志级别**：生产根级别为 INFO；可通过 JVM 参数或外部属性覆盖 `logger.level` 与 `logger.sql.level`。
- **日志路径**：统一写入 `/opt/metersphere/logs/metersphere`，由 `commons.properties` 中的 `logging.file.path` 驱动。
- **SQL 日志**：每个模块的 mapper 包需显式在 `logback-spring.xml` 中注册（XML 注释写明“包名不支持通配符 * ，这里需要每个模块固定好 mapper”），新增模块需手动追加对应 `<logger name="io.metersphere.xxx.mapper">` 条目。
- **异步化**：所有文件输出走 AsyncAppender，队列大小固定为 10000；ERROR/WARN 启用 caller data 以保留调用位置。
- **历史清理**：按天滚动 + 最大 50MB 单文件 + 最多保留 30 天（可被 `logger.max.history` 覆盖）。
- **编码**：文件 encoder 强制 UTF-8。
- **前端**：未发现前端侧的日志采集或上报逻辑，前端错误监控位于 `frontend/src/utils/monitor.ts`（与后端日志无关）。