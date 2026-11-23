---
kind: external_dependency
name: 认证与权限：Apache Shiro 2.0 + 自定义注解拦截
slug: apache-shiro
category: external_dependency
category_hints:
    - auth_protocol
scope:
    - '**'
---

### 认证鉴权集成
- 数据级权限通过 AOP 切面实现：@CheckOwner / @CheckProjectOwner / @CheckOrgOwner 配合对应 Aspect。
- Session 存储在 Redis，超时 12 小时（spring.session.timeout=43200s）。
- 前端路由守卫 + permission 双重控制，未开通功能模块走 featureRouteMap 跳转无权限页。