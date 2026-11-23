---
kind: external_dependency
name: 执行引擎：JMeter 5.6.3 + metersphere-jmeter-assertions/functions
slug: jmeter
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### 接口/场景执行内核
- 插件体系：MsTestElement 经 JmeterElementConverter / MsElementConverter 转换为 JMeter 元素后下发到资源池节点执行。
- 执行结果通过 Listener 回调回主进程，再由 WebSocket 实时推送。