---
kind: external_dependency
name: 消息队列：Kafka（异步执行与回调）
slug: kafka
category: external_dependency
category_hints:
    - client_constraint
scope:
    - '**'
---

### 异步执行总线
- Kafka 作为接口/场景执行的异步通道：Controller → Service 组装 TaskRequestDTO → 投递到资源池 → 节点消费执行 → Listener 回调 ApiExecuteCallbackService。
- 单条消息上限调至 1GB，用于大文件/脚本传输。
- 消费端手动提交 + 长轮询（MAX_POLL_INTERVAL_MS=900000）。
- Topic 常量集中在 sdk/KafkaTopicConstants。