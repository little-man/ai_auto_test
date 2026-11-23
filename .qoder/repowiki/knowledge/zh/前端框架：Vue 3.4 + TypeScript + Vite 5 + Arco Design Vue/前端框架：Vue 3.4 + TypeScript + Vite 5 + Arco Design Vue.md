---
kind: external_dependency
name: 前端框架：Vue 3.4 + TypeScript + Vite 5 + Arco Design Vue
slug: vue3-vite-arco-design
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### MeterSphere 前端技术栈
- 运行时：Vue 3.4 + TypeScript 5.4 + vue-router（hash 模式）+ Pinia（持久化插件）+ vue-i18n。
- UI 组件库：Arco Design Vue（含主题包），样式走 Tailwind + Less。
- 开发代理：vite.config.dev.ts 把 /front、/ws、/project、/organization、/file 等前缀转发到 VITE_DEV_DOMAIN。
- 特色依赖：ECharts、Monaco/Ace、TipTap 富文本、kityminder 脑图、form-create 动态表单、@tanstack/vue-query、sortablejs。