---
kind: dependency_management
name: Maven + pnpm 双栈依赖管理（聚合 POM + 私有仓库）
category: dependency_management
scope:
    - '**'
source_files:
    - pom.xml
    - backend/pom.xml
    - frontend/package.json
    - Jenkinsfile
    - .github/workflows/codecov.yml
    - .github/workflows/sonar.yml
    - .github/workflows/frontend-build.yml
    - build.md
---

## 1. 使用的系统/工具

- **后端 Java**：Maven 3.8.3（Jenkinsfile 中 `PATH` 指定），基于 Spring Boot 3.5.7 的 `spring-boot-starter-parent`，使用 `mvnw`（Maven Wrapper）作为统一入口。
- **前端 JS/TS**：pnpm 8.4.0（由根 POM 属性 `pnpm.version` 锁定），通过 `frontend-maven-plugin` 在 Maven 构建阶段调用；Node 版本由属性 `node.version=v20.8.1` 与 `engines.node>=18.0.0` 共同约束。
- **CI/CD**：Jenkins Pipeline（`Jenkinsfile`）+ GitHub Actions（`.github/workflows/codecov.yml`、`sonar.yml`、`frontend-build.yml`、`sync-to-gitee.yml`）。
- **镜像打包**：Docker Buildx 多架构（`linux/amd64,linux/arm64`），镜像推送到 `registry.fit2cloud.com/metersphere`。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `pom.xml` | 根聚合 POM，声明所有第三方库版本、私有仓库、插件及 `dependencyManagement` |
| `backend/pom.xml` | backend 子聚合 POM，引入测试容器、MockServer 等 test scope 依赖 |
| `frontend/package.json` | 前端依赖清单（dependencies + devDependencies + resolutions） |
| `Jenkinsfile` | CI 流水线，注入 `settings.xml` 并执行 Maven 构建/部署 |
| `.github/workflows/*.yml` | GitHub Actions 中的 Maven 构建与 Sonar/Codecov 任务 |
| `build.md` | 本地构建文档，引用 `./mvnw` |

## 3. 架构与约定

### 3.1 后端依赖管理（Maven）

- **集中式版本治理**：根 `pom.xml` 的 `<properties>` 段集中声明全部第三方库版本（如 `shiro.version=2.0.2`、`guava.version=33.0.0-jre`、`jmeter.version=5.6.3`、`redisson-starter.version=3.49.0` 等），子模块通过 `${xxx.version}` 引用，不直接写版本号。
- **BOM 导入**：通过 `<dependencyManagement><import>pom</import></dependencyManagement>` 引入 `org.springframework.ai:spring-ai-bom:${spring-ai.version}`，统一管理 Spring AI 生态依赖版本。
- **Spring Boot Parent**：继承 `spring-boot-starter-parent:3.5.7`，获得默认依赖版本与插件配置；Java 目标版本固定为 `java.version=21`。
- **版本扁平化**：启用 `flatten-maven-plugin`（`flattenMode=resolveCiFriendliesOnly`），在 `process-resources` 阶段生成不含 `${revision}` 等变量的扁平 POM，便于发布到仓库。
- **版本号变量**：项目自身版本通过 `${revision}`（默认 `3.x`）管理，Jenkins 通过 `-Drevision=${REVISION}` 覆盖。

### 3.2 仓库与私服

- **私有仓库**：根 POM 同时声明 `<repositories>`、`<pluginRepositories>` 与 `<distributionManagement>`，均指向 `https://repository.fit2cloud.com/repository/metersphere/`（id=`metersphere-public`），用于拉取 MeterSphere 自有组件（如 `metersphere-jmeter-functions`、`metersphere-jmeter-assertions`、`monitoring-engine`）以及发布产物。
- **中央仓库**：同时保留 `repo.maven.apache.org/maven2` 作为 `central` 镜像。
- **CI 注入 settings**：Jenkinsfile 通过 `configFileProvider([configFile(fileId: 'metersphere-maven', targetLocation: 'settings.xml')])` 将 Jenkins 凭据中的 `settings.xml` 注入到工作目录，再传给 `./mvnw --settings ./settings.xml`。这意味着认证信息（用户名/密码或 token）不在代码库中，而在 Jenkins 凭据管理中。
- **本地仓库清理**：Build/Test 阶段会删除本地缓存中的 `io/metersphere/metersphere-jmeter-assertions/*`、`metersphere-jmeter-functions/*`、`monitoring-engine/*`，确保从私服拉取最新快照。

### 3.3 前端依赖管理（pnpm）

- **包管理器锁定**：根 POM 通过 `frontend-maven-plugin` 强制使用 `pnpm@8.4.0` 和 `node@v20.8.1`，避免开发者环境差异。
- **依赖声明**：`frontend/package.json` 中 `dependencies` 与 `devDependencies` 分别声明运行时与开发期依赖；`resolutions` 字段强制替换 `bin-wrapper`、`rollup`、`gifsicle` 的版本以规避安全漏洞或兼容性问题。
- **引擎约束**：`engines.node >= 18.0.0` 与 POM 中 `node.version=v20.8.1` 双重约束 Node 版本。
- **无 lockfile 提交**：当前仓库未检出 `pnpm-lock.yaml`（搜索结果为空），依赖解析由 pnpm 在构建时根据 `package.json` 的语义版本范围决定。

### 3.4 构建与发布流程

- **SDK 构建模式**（`BUILD_SDK=true`）：先 `deploy -N` 仅发布父 POM，再 `clean deploy -pl !app` 跳过 app 模块，将 framework/services 各模块发布到私服。
- **社区构建模式**（`BUILD_SDK=false`）：`clean install -DskipTests` 后，将 `general-xpack-impl` 等 jar 拷贝进 `backend/app/target/dependency/BOOT-INF/lib/`，再用 Docker Buildx 打多架构镜像并 push 到 `registry.fit2cloud.com/metersphere`。
- **GitHub Actions**：`codecov.yml` 与 `sonar.yml` 直接运行 `mvn`（非 mvnw），`frontend-build.yml` 通过 `mvn -pl frontend` 触发前端构建。

## 4. 约定与约束

- **版本集中声明**：所有后端第三方库版本集中在根 `pom.xml` 的 `<properties>` 中，子模块不得硬编码版本号——这是 POM 结构本身所体现的约定。
- **私有仓库优先**：MeterSphere 自有组件（groupId `io.metersphere`）必须从 `repository.fit2cloud.com` 拉取，而非 Maven Central。
- **CI 凭据外置**：Maven 私服认证通过 Jenkins `configFileProvider` 注入 `settings.xml`，源码中不包含任何凭据。
- **Node/pnpm 版本锁定**：构建阶段强制使用 POM 中定义的 `node.version` 与 `pnpm.version`，不受宿主环境影响。
- **测试依赖隔离**：testcontainers、mockserver、embedded-* 等仅在 `backend/pom.xml` 中以 `<scope>test</scope>` 声明，不进入生产制品。
- **Sonar/CodeCov 集成**：根 POM 通过 properties 声明 `sonar.organization=metersphere`、`sonar.host.url=https://sonarcloud.io`，配合 `jacoco-maven-plugin` 在 `prepare-package` 阶段生成覆盖率报告。
- **无 vendoring**：未发现 `vendor/`、`lib/` 等第三方代码内嵌目录，所有依赖均通过远程仓库解析。
- **无 npm/yarn 混用**：前端仅使用 pnpm（`pnpm run build:local` 见 README），`package.json` 中未出现 yarn/npm 脚本别名。