---
kind: external_dependency
name: 缓存与会话：Redis + Redisson（Session 外置）
slug: redis-redisson
category: external_dependency
category_hints:
    - client_constraint
scope:
    - '**'
---

### 会话与分布式缓存
- Redis 承载 Shiro Session，超时 12 小时。
- Redisson 提供分布式锁等能力，配置文件 redisson.yml 外置。
- 本地集成测试通过 testcontainers 的 embedded-redis 提供。