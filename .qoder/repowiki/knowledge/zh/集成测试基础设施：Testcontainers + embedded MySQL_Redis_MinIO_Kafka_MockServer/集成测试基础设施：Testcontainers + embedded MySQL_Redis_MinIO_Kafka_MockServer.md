---
kind: external_dependency
name: 集成测试基础设施：Testcontainers + embedded MySQL/Redis/MinIO/Kafka/MockServer
slug: testcontainers
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### 本地集成测试环境
- 后端统一引入 Testcontainers + playtika embedded-mysql/redis/minio/kafka/mockserver。
- surefire 按字母序执行测试用例。
- 这是本地写集成测试的基础设施约定，新增模块应复用而非自建中间件容器。