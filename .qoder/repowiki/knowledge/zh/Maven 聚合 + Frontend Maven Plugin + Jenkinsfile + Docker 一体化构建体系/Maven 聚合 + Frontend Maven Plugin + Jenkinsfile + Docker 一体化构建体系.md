---
kind: build_system
name: Maven 聚合 + Frontend Maven Plugin + Jenkinsfile + Docker 一体化构建体系
category: build_system
scope:
    - '**'
source_files:
    - pom.xml
    - frontend/pom.xml
    - Dockerfile
    - Jenkinsfile
    - build.md
---

## 1. 使用的系统/工具

- **后端**：基于 Spring Boot 3.5.7（Java 21）的 Maven 聚合工程，根 `pom.xml` 以 `<packaging>pom</packaging>` 编排 `frontend`、`backend` 两个子模块。
- **前端**：Vue3 + TypeScript + Vite，通过 `frontend/pom.xml` 中的 `frontend-maven-plugin`（v1.12.1）在 Maven 生命周期内安装 Node v20.8.1 与 pnpm 8.4.0，并执行 `pnpm install` 和 `pnpm run build`。
- **CI**：Jenkins Pipeline（`Jenkinsfile`），使用 Groovy DSL，依赖 `configFileProvider` 注入 `settings.xml`，并通过 `qyWechatNotification` 推送企业微信通知。
- **镜像**：单文件 `Dockerfile`，基础镜像为 `registry.fit2cloud.com/metersphere/alpine-openjdk21-jre`，将后端 `BOOT-INF` 与前端静态资源合并打包。
- **版本管理**：通过 `maven-flatten-plugin`（`resolveCiFriendliesOnly`）在 `process-resources` 阶段生成扁平化 POM，`${revision}` 属性统一控制版本号；默认值为 `3.x`。

## 2. 关键文件

- `pom.xml`：根聚合 POM，定义 Java 21、Spring Boot parent、所有第三方依赖版本、Jacoco 排除规则、SonarCloud 配置、Maven 仓库（`repo.maven.apache.org` + `repository.fit2cloud.com/repository/metersphere`）。
- `frontend/pom.xml`：集成 `frontend-maven-plugin`，绑定 `generate-resources` 阶段执行 Node/pnpm 安装与构建。
- `backend/app/pom.xml`：Spring Boot 应用打包（由父 POM 继承），被 Jenkins 解压到 `backend/app/target/dependency` 供 Docker 使用。
- `Dockerfile`：复制 `BOOT-INF/lib`、`META-INF`、`BOOT-INF/classes` 以及 `backend/app/src/main/resources/static` 与 `frontend/public` 到 `/app/static`。
- `Jenkinsfile`：多阶段 Pipeline，包含 SDK 发布、Build/Test、Community 镜像构建与推送、Standalone 触发等阶段。
- `build.md`：官方文档说明本地构建命令（`./mvnw install -N`、`./mvnw clean install ... --file backend/pom.xml`、`./mvnw clean package`）。

## 3. 架构与约定

### 3.1 Maven 聚合结构
根 POM 声明 `<modules><module>frontend</module><module>backend</module></modules>`，因此 `mvn clean install` 会依次构建 frontend 与 backend。frontend 子模块本身是 `pom` 类型，仅用于挂载 `frontend-maven-plugin`。

### 3.2 前端构建嵌入 Maven
`frontend/pom.xml` 中 `frontend-maven-plugin` 的三个 execution：
1. `install node and pnpm`（phase=`generate-resources`）——下载固定版本的 Node v20.8.1 与 pnpm 8.4.0 到 `.node` 目录。
2. `pnpm install` —— 安装依赖。
3. `pnpm run build` —— 调用 Vite 生产构建。
同时 `maven-clean-plugin` 被扩展以清理 `dist` 目录。

### 3.3 版本策略
- `${revision}` 在根 POM 的 `<properties>` 中定义为 `3.x`，并被 `flatten-maven-plugin` 在 `process-resources` 阶段写入扁平 POM，使 CI 可通过 `-Drevision=xxx` 覆盖。
- Jenkins 从 `BRANCH_NAME` 推导 REVISION：若分支名以 `v` 开头则去掉前缀作为版本号，否则直接使用分支名。
- 镜像标签：`${TAG_NAME:-$BRANCH_NAME}-${GIT_COMMIT:0:8}`，即支持显式 TAG 或回退到短 SHA。

### 3.4 Jacoco 覆盖率排除
根 POM 对 `io/metersphere/*/mapper/**`、`domain/**`、`dto/**`、`config/**`、`constants/**`、`utils/**`、`sdk/**`、`provider/**`、`plugin/**`、`job/**` 等包统一排除，只统计业务逻辑代码。

### 3.5 Docker 镜像组装
`Dockerfile` 假设 `backend/app/target/dependency` 已存在（由 Jenkins 通过 `jar -xf ../*.jar` 解压生成），并将前后端静态资源合并到 `/app/static`。运行参数通过环境变量注入：`JAVA_CLASSPATH`、`JAVA_MAIN_CLASS=io.metersphere.Application`、`JAVA_OPTIONS`（含 UTF-8、headless、`--add-opens`）。镜像入口为 `/deployments/run-java.sh`。

### 3.6 Jenkins Pipeline 流程
- **Preparation**：解析 `REVISION`、`BUILD_SDK`、可选的 `FRONTEND_LINK`。
- **SDK**（当 `BUILD_SDK=true`）：`./mvnw deploy -N` 安装 parent，再 `clean deploy -T 1C -B -DskipTests -DskipAntRunForJenkins -pl !app` 发布除 app 外的所有后端模块。
- **Build/Test**（默认）：`clean install -DskipTests`，然后解压 JAR 到 `backend/app/target/dependency`。
- **Community build & push**：从本地 Maven 仓库拷贝 `general-xpack-impl` 等 jar 到 `BOOT-INF/lib`，再用 `docker buildx` 构建 `linux/amd64,linux/arm64` 双架构镜像并推送至 `registry.fit2cloud.com/metersphere/metersphere:<tag>`。
- **Build Standalone**：当 REVISION 匹配正则 `^\d+\.\w+$` 且非 SDK 构建时，触发下游 job `../metersphere-standalone/${BRANCH_NAME}`。
- **post Notification**：始终通过企业微信 webhook 发送通知。

## 4. 约定与约束

- **Java 版本**：根 POM 强制 `java.version=21`，编译器插件使用 `<release>${java.version}</release>`；Jenkins 节点要求 `JAVA_HOME=/opt/jdk-21`。
- **Node/pnpm 版本锁定**：通过 `frontend-maven-plugin` 固定 Node v20.8.1、pnpm 8.4.0，避免环境差异。
- **版本号来源**：版本号统一来自 `${revision}`，CI 通过 `-Drevision=${REVISION}` 传入；本地开发默认 `3.x`。
- **仓库源**：所有依赖与插件均从 Maven Central 与 `https://repository.fit2cloud.com/repository/metersphere/` 拉取，私有依赖需部署到该仓库。
- **镜像基础镜像**：必须使用 `registry.fit2cloud.com/metersphere/alpine-openjdk21-jre`，不可替换。
- **构建产物路径**：Dockerfile 硬编码读取 `backend/app/target/dependency` 下的 `BOOT-INF/lib`、`META-INF`、`BOOT-INF/classes`，因此必须先执行 Maven 构建并解压 JAR。
- **Jacoco 排除范围**：见根 POM 中 `<excludes>` 列表，新增包若要纳入覆盖率统计需显式移除排除项。
- **CI 设置**：Jenkins 通过 `configFileProvider` 加载 `metersphere-maven` 配置的 `settings.xml`，构建时必须提供该凭据。
- **Standalone 触发条件**：仅当 `REVISION` 匹配 `^\d+\.\w+$`（如 `3.3.0`）且 `BUILD_SDK=false` 时才触发独立构建下游 job。
- **多架构镜像**：Jenkins 使用 `--platform linux/amd64,linux/arm64` 构建多架构镜像，需要 Docker buildx 支持。
