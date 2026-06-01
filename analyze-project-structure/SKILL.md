---
name: analyze-project-structure
description: 分析并总结软件项目的工程结构、目录布局、技术栈和模块组织。当用户提出以下需求时使用：(1) 分析、探索或理解项目的结构或架构，(2) 识别项目使用的技术、框架或语言，(3) 获取代码库的组织方式和关键目录概览，(4) 查看项目配置、构建设置或依赖管理文件，(5) 梳理模块、服务或包之间的关系。触发关键词包括：'分析项目结构'、'project architecture'、'技术栈分析'、'项目结构'、'codebase overview'、'what framework does this use'、'了解这个项目'、'目录结构'、'工程结构'。
---

# 工程结构分析

系统化分析项目的工程结构，提供清晰、结构化的概览。

## 分析流程

按以下步骤顺序执行。当用户已获得足够上下文时停止——避免过度分析。

### 1. 目录布局

以递归方式遍历目录结构，生成完整的树形目录结构，了解项目的整体形态和各层级组织。

```bash
# 优先使用 tree 命令生成树形结构（限制深度，避免大型项目输出过多）
tree -L 3 -d -I 'node_modules|target|dist|build|__pycache__|.git|.idea|.vscode' 2>/dev/null || find . -maxdepth 3 -type d -not -path '*/node_modules/*' -not -path '*/target/*' -not -path '*/__pycache__/*' -not -path '*/.git/*' | sort | sed 's|^\./||; s|[^/]*/|  |g' | head -80
```

**递归分析要点：**
- 对每个关键子目录（如 `src/`、`app/`、`pkg/`、`lib/`、`cmd/`）继续深入遍历其内部结构
- 关注目录的层级深度和命名模式，这通常反映架构分层（如 MVC、DDD、按功能/技术分层）
- 标记空目录、单文件目录和过度嵌套的目录

**注意：** 对于 monorepo 或超大型项目，初始展示深度 2 或 3，随后根据用户兴趣深入特定子目录。始终优先展示真实目录树而非手动缩进。

### 2. 技术栈识别

通过特征文件检测主要语言、框架和构建工具：

**常见标识文件：**
- **JavaScript/TypeScript**: `package.json`, `node_modules/`, `tsconfig.json`
- **Python**: `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile`, `poetry.lock`
- **Go**: `go.mod`, `go.sum`
- **Java**: `pom.xml`, `build.gradle`, `build.gradle.kts`
- **Rust**: `Cargo.toml`, `Cargo.lock`
- **Ruby**: `Gemfile`, `Gemfile.lock`
- **Docker**: `Dockerfile`, `docker-compose.yml`
- **CI/CD**: `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`

读取最相关的 1-2 个配置文件，确认版本和关键依赖。

### 3. 项目类型归类

基于发现的信息对项目进行分类：

| 类型 | 判断依据 |
|------|---------|
| Monorepo | 不同层级存在多个 `package.json`、`go.mod` 或 `pom.xml`；存在 workspace 配置（`pnpm-workspace.yaml`、`lerna.json`） |
| 微服务 | 多个服务目录、Docker Compose 包含多个服务、API 网关配置 |
| 库/包 | `setup.py`、带 `[lib]` 的 `Cargo.toml`、无 `scripts.start` 的 `package.json` |
| Web 应用 | `index.html`、前端框架（React、Vue、Angular）、`public/` 或 `dist/` |
| CLI 工具 | `main.go` 或 `cmd/`、`__main__.py`、参数解析库 |
| 后端 API | 路由文件、controller 目录、带 HTTP 服务的 `main.go`、使用 Flask/FastAPI/Django 的 `app.py` |

### 4. 模块与依赖分析

理解代码的组织方式：

- **入口点**: `main()`、`index.js`、`app.py`、`cmd/<name>/main.go`
- **源码目录**: `src/`、`lib/`、`pkg/`、`internal/`、`app/`
- **测试结构**: `test/`、`tests/`、`*_test.go`、`*.spec.ts`、`__tests__/` — 如有覆盖率信息也一并记录
- **配置**: `config/`、`.env*`、`application.yml`、`settings.py`
- **文档**: `README.md`、`docs/`、`DESIGN.md`、`ARCHITECTURE.md`

分析依赖时，如环境支持则使用语言专属工具：

```bash
# Go
go list -m all | head -20

# Python（如 pip 可用）
pip list 2>/dev/null | head -20

# Node.js
npm list --depth=0 2>/dev/null || yarn list --depth=0 2>/dev/null

# Java（如 Maven 可用）
mvn dependency:tree -DoutputType=text 2>/dev/null | head -30
```

### 5. 输出格式

以结构化报告呈现分析结果：

```markdown
## 项目概览

- **名称**: （取自 package.json/pom.xml/go.mod 或目录名）
- **类型**: （Web 应用 / API / 库 / CLI / Monorepo / 微服务）
- **主语言**: （Go / TypeScript / Python / Java 等）
- **框架**: （React / FastAPI / Spring Boot / Gin 等）
- **构建工具**: （npm / Maven / Gradle / Cargo / Poetry 等）

## 目录结构

```
（递归遍历得到的完整树形目录结构，展示各层级子目录关系）
```

## 关键模块

| 模块 | 路径 | 用途 |
|------|------|------|
| ...  | ...  | ...  |

## 技术栈

- **运行环境**: （Node.js 20 / Python 3.11 / Go 1.22 / Java 17）
- **核心依赖**: （列出最重要的 5-10 个）
- **数据库**: （如可从配置或依赖中识别）
- **部署方式**: （Docker / Kubernetes / Serverless — 如可识别）

## 架构特点

- （Monorepo workspace 工具、架构模式、测试策略等）
```

## 执行准则

- **简明扼要**: 聚焦关键信息，跳过生成文件（`node_modules/`、`target/`、`__pycache__/`）。
- **先广度后深度**: 从顶层开始，仅在用户关心的领域深入。
- **不臆测**: 若某项技术不确定，以可能性形式给出并附证据。
- **尊重 `.gitignore`**: 除非用户明确要求，否则不枚举被忽略的目录。
