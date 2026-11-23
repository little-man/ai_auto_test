---
kind: external_dependency
name: 定时任务：Quartz（独立连接池，非自动装配）
slug: quartz
category: external_dependency
category_hints:
    - migration_status
scope:
    - '**'
---

### 调度框架
- Quartz 在 Application 中排除了自动配置，改为 ScheduleConfig 自行装配。
- 拥有独立的数据库连接池，与业务库分离。
- 表结构由 Flyway 在 migration 中维护。