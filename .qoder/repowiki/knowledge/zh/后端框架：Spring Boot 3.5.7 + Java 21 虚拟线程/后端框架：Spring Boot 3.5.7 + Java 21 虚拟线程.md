---
kind: external_dependency
name: 后端框架：Spring Boot 3.5.7 + Java 21 虚拟线程
slug: spring-boot
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### MeterSphere 后端运行框架
- 启动壳 backend/app 排除 Quartz/Ldap/Neo4j 自动配置，自行装配；配置文件加载顺序为 classpath:commons.properties → file:/opt/metersphere/conf/metersphere.properties。
- 端口：HTTP 8081，管理端点 7071。
- 生产形态：通过 maven-antrun-plugin 将前端静态资源打入 jar，由后端 Expr 静态托管，单 jar 部署。