---
kind: external_dependency
name: ORM 与连接池：MyBatis + HikariCP + PageHelper
slug: mybatis-hikaricp-pagehelper
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### 数据访问层
- MyBatis（开启 map-underscore-to-camel-case）+ HikariCP 连接池，业务库与 Quartz 调度库各自独立连接池。
- 分页使用 PageHelper。
- 实体类由各 services 模块的 mybatis-generator-maven-plugin 生成（如 apiGeneratorConfig.xml）。