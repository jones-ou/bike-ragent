
# 本地运行 bike-Ragent 完整指南
本指南将带你从零开始在本地机器上运行 bike-Ragent，完成首次 AI 对话。bike-Ragent 依赖 MySQL、Redis、Milvus、RustFS（兼容 S3 的对象存储）和 LLM 提供商五大外部服务，将按「基础设施 → 数据库初始化 → 后端启动 → 前端访问」的顺序逐步操作。

## 前置条件与环境概览
### 核心架构说明
采用「React 前端 + Spring Boot 后端」架构：
- React 前端将 API 调用代理到 Spring Boot 后端
- 后端分别与 Milvus（向量搜索）、MySQL（关系数据）、Redis（缓存/限流）、RustFS（文档存储）通信
- 外部 LLM 提供商负责 embedding 生成和聊天补全

### 软件依赖要求
| 组件         | 版本 / 详情                | 用途                          |
|--------------|----------------------------|-------------------------------|
| JDK          | 17+                        | 编译并运行 Spring Boot 后端   |
| Node.js      | 18+（推荐 20 LTS）         | 构建并服务 React 前端         |
| Maven        | 3.8+（或使用自带的 ./mvnw） | 构建多模块项目                |
| Docker & Compose | Docker 24+, Compose v2+  | 运行 MySQL/Redis/Milvus 等服务 |
| Git          | 任何近期版本               | 克隆代码仓库                  |

### 硬件要求
| 硬件 | 最低要求 | 推荐配置                      |
|------|----------|-------------------------------|
| CPU  | 2 核     | 4+ 核                         |
| RAM  | 8 GB     | 16 GB（仅 Milvus 就可能消耗 1–3 GB） |
| 磁盘 | 10 GB 可用 | 20 GB 可用（用于镜像和文档存储） |

## 步骤 1 — 克隆代码仓库
```bash
git clone https://github.com/jones-ou/bike-ragent.git
cd bike-ragent
```
仓库顶层目录结构：
- 4 个 Maven 模块：bootstrap、framework、infra-ai、mcp-server
- frontend/：React 前端代码
- resources/：数据库脚本、Docker Compose 文件
- docs/：补充文档

## 步骤 2 — 启动基础设施服务
### 启动 Milvus 及依赖服务（Docker Compose）
提供两种 Compose 配置，按需选择：

| 配置类型 | 默认（resources/docker/） | 轻量级（resources/docker/lightweight/） |
|----------|---------------------------|-----------------------------------------|
| 内存限制 | 无（容器按需占用）| Milvus 3 GB、etcd 256 MB、RustFS 256 MB、Attu 256 MB |
| Milvus 版本 | 2.6.6 | 2.6.6（提供 2.5.8 备用文件） |
| 适用场景 | 生产类环境（16 GB+ 内存） | 本地开发（8 GB 内存机器） |

#### 启动命令
```bash
# 选项 A：默认配置
cd resources/docker
docker compose -f milvus-stack-2.6.6.compose.yaml up -d

# 选项 B：轻量级配置（内存受限）
cd resources/docker/lightweight
docker compose -f milvus-stack-2.6.6.compose.yaml up -d
```

#### 启动的容器说明
| 容器            | 镜像                          | 暴露端口       | 角色                                  |
|-----------------|-------------------------------|----------------|---------------------------------------|
| milvus-standalone | milvusdb/milvus:v2.6.6       | 19530, 9091    | 向量数据库（embedding 存储/相似性搜索） |
| etcd            | quay.io/coreos/etcd:v3.5.18   | 仅内部         | Milvus 元数据存储                     |
| rustfs          | rustfs/rustfs:1.0.0-alpha.72  | 9000, 9001     | 兼容 S3 的对象存储（文档上传）|
| attu            | zilliz/attu:v2.6.3            | 8000→3000      | Milvus 集合浏览 Web GUI               |

#### 验证容器状态
```bash
docker compose ps
```
正常状态应为 `healthy` 或 `running`。

#### CentOS 7 特殊处理
Milvus 2.6.6 依赖 CentOS 7 默认内核缺失的功能，若启动失败/日志出现 `seccomp`/`SIGILL` 错误，切换至 2.5.8 版本：
```bash
cd resources/docker/lightweight
docker compose -f milvus-stack-2.5.8.compose.yaml up -d
```

### 启动 MySQL 和 Redis
```bash
# 启动 MySQL（默认密码 root）
docker run -d --name ragent-mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root \
  mysql:8.0

# 启动 Redis（默认密码 123456）
docker run -d --name ragent-redis \
  -p 6379:6379 \
  redis:7 --requirepass 123456
```
以上凭据与 `application.yaml` 中默认配置一致。

## 步骤 3 — 初始化数据库
在 MySQL 运行状态下，执行仓库内的 SQL 脚本：
```bash
# 从项目根目录执行
mysql -u root -proot < resources/database/schema_table.sql
mysql -u root -proot ragent < resources/database/init_data.sql
```
也可通过 Navicat/DBeaver/DataGrip 等 GUI 客户端运行脚本。

### 脚本说明
- `schema_table.sql`：创建 ragent 数据库及 20+ 核心表（t_user、t_conversation、t_knowledge_base 等）
- `init_data.sql`：植入默认管理员账户（用户名：admin，密码：admin，角色：admin）

## 步骤 4 — 配置 LLM 提供商
需配置至少一个有效 LLM 提供商，默认聊天模型为 BaiLian 的 qwen3-max。

### 选项 A：BaiLian（阿里云 DashScope，推荐快速开始）
```bash
export BAILIAN_API_KEY=sk-your-dashscope-api-key-here
```
BaiLian 支持聊天补全、embedding、rerank，默认配置已路由至该提供商。

### 选项 B：SiliconFlow
```bash
export SILICONFLOW_API_KEY=sk-your-siliconflow-key-here
```
SiliconFlow 支持聊天和 embedding，默认 rerank 仍指向 BaiLian，需额外配置 BaiLian Key 或启用 noop 重排序回退。

### 选项 C：Ollama（完全本地，无需 API Key）
```bash
# 拉取所需模型
ollama pull qwen3:8b-fp16
ollama pull qwen3-embedding:8b-fp16

# 启动 Ollama（默认端口 11434）
ollama serve
```
若仅用 Ollama，需修改 `ai.chat.default-model` 和 `ai.embedding.default-model` 指向 Ollama 模型 ID。

### 混合配置（推荐）
同时设置 `BAILIAN_API_KEY` 和 `SILICONFLOW_API_KEY`：
- BaiLian：聊天/重排序（质量更好）
- SiliconFlow：embedding（成本更低）
  默认配置可直接开箱即用。

### 模型配置总结
| 能力            | 默认模型               | 提供商   | 环境变量          |
|-----------------|------------------------|----------|-------------------|
| 聊天（标准）| qwen3-max              | BaiLian  | BAILIAN_API_KEY   |
| 聊天（深度思考） | qwen3-max              | BaiLian  | BAILIAN_API_KEY   |
| Embedding       | qwen-emb-8b            | SiliconFlow | SILICONFLOW_API_KEY |
| Rerank          | qwen3-rerank           | BaiLian  | BAILIAN_API_KEY   |
| 聊天（本地）| qwen3:8b-fp16          | Ollama   | —                 |
| Embedding（本地）| qwen3-embedding:8b-fp16 | Ollama   | —                 |

## 步骤 5 — 构建并启动后端
```bash
# 从项目根目录执行
# 构建所有模块（跳过测试加速）
./mvnw clean package -DskipTests

# 方式 1：通过 jar 包启动
java -jar bootstrap/target/bootstrap-0.0.1-SNAPSHOT.jar

# 方式 2：直接通过 Maven 运行
./mvnw spring-boot:run -pl bootstrap
```

### 关键说明
- 后端启动在 9090 端口，上下文路径为 `/api/ragent`
- 需确认日志中以下关键行（验证服务连接）：
  ```
  Started RagentApplication in X.XX seconds
  Milvus client connected successfully
  ```
- 若连接失败，检查 Docker 容器状态及 `application.yaml` 中的凭据配置。

## 步骤 6 — 启动前端
```bash
# 打开新终端，从项目根目录进入前端目录
cd frontend
npm install
npm run dev
```

### 关键说明
- 前端启动在 5173 端口（Vite 开发服务器）
- `/api/*` 请求自动代理到后端 `http://localhost:9090`，无需配置 CORS
- 核心访问地址：
    - 主聊天界面：http://localhost:5173
    - Milvus 管理 GUI（Attu）：http://localhost:8000

## 步骤 7 — 登录并完成首次 AI 对话
### 1. 登录系统
访问 `http://localhost:5173`，使用默认管理员凭据：
- 用户名：admin
- 密码：admin

### 2. 创建知识库
1. 点击聊天侧边栏左下角「用户头像」→「管理后台」
2. 进入「知识库管理」→「新建知识库」，填写信息：
   | 字段          | 建议值         | 备注                          |
   |---------------|----------------|-------------------------------|
   | 名称（Name）| 公司手册       | 任意描述性名称                |
   | Embedding 模型 | Qwen/Qwen3-Embedding-8B | 需匹配配置的 embedding 候选 |
   | Collection 名称 | company_manual | 唯一标识符，映射到 Milvus 集合 |

### 3. 上传文档
创建知识库后上传文档（支持 PDF/Word/PPT 等），系统自动完成：
1. 解析文档为纯文本
2. 按配置策略分块文本（固定大小/段落/结构感知）
3. 向量化每个文本块（通过配置的 embedding 模型）
4. 将向量索引写入 Milvus 集合

### 4. 首次 RAG 对话
1. 返回聊天界面（头像 → 用户问答）
2. 创建新对话，询问与上传文档相关的问题
3. 系统执行流程：
    - 上下文澄清（如需）→ 重写查询
    - 基于意图树分类意图
    - Milvus 多通道搜索相关文本块
    - 生成带来源引用的流式回答

## 总结
1. Ragent 本地运行需先部署 MySQL/Redis/Milvus/RustFS 基础设施，再完成数据库初始化、LLM 配置、前后端启动；
2. LLM 提供商可选择阿里云 BaiLian（快速）、SiliconFlow（低成本）或 Ollama（本地无 API 依赖）；
3. 首次对话需先创建知识库并上传文档，系统会自动完成文本分块、向量化和索引，最终生成带来源的流式回答。

