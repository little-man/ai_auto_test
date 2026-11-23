---
kind: external_dependency
name: 对象存储：MinIO（附件/文件仓库）
slug: minio
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
---

### 文件存储
- MinIO 作为统一附件/文件存储后端，通过 MinioConfig + MinioRepository（SDK 层抽象）接入。
- 与系统设置中的「文件服务」配置联动。