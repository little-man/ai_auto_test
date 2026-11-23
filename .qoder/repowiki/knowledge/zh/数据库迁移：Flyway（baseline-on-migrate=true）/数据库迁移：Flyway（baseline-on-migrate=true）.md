---
kind: external_dependency
name: 数据库迁移：Flyway（baseline-on-migrate=true）
slug: flyway
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### 表结构演进
- 启用 baseline-on-migrate=true，并通过 metersphere_version 表跟踪版本。
- 新增字段需在对应版本的 migration 目录加 V<version>_x__xxx.sql，再触发 mybatis-generator 生成映射。