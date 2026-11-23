---
kind: external_dependency
name: AI 引擎：Spring AI 多模型适配（OpenAI/DeepSeek/千帆/智谱）
slug: spring-ai
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### AI 助手能力
- 基于 Spring AI 的多模型引擎，封装 ChatToolEngine，内置四种 Client：OpenAI、DeepSeek、千帆、智谱。
- 提供 tools（日期、JVM、文件读取）与 TextRank 摘要等能力。
- 通过 ai-engine 模块对外暴露，供业务域调用。