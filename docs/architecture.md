---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - 'docs/prd.md'
  - 'docs/analysis/research/technical-happy-server-go-rewrite-research-2025-12-09.md'
workflowType: 'architecture'
lastStep: 8
status: 'complete'
completedAt: '2025-12-10'
project_name: 'happy-server-go'
user_name: 'XDwanj'
date: '2025-12-09'
techPreferences:
  orm: 'GORM'
  websocket: 'coder/websocket'
  projectInit: 'manual'
  devTools: 'minimal'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## 项目上下文分析

### 需求概览

**功能性需求：**

Happy Server Go 是 Happy Server (Node.js) 的完整功能重写，必须保持 100% API 兼容性。核心功能包括：

1. **零知识加密存储**
   - 服务器存储加密数据但无法解密（客户端加密/解密）
   - 支持端到端加密的多设备同步
   - 架构含义：所有敏感数据以 blob 形式存储，服务器仅负责路由和持久化

2. **完整的 RESTful API**（50+ 端点）
   - 认证（公钥签名、终端认证请求）
   - 会话管理（CRUD、消息、活跃度跟踪）
   - 设备管理（注册、状态、元数据）
   - Artifacts 存储（header/body 分离、版本控制）
   - 账户管理（资料、设置、头像上传）
   - 键值存储（支持 CAS 操作、批量变更）
   - 推送通知、社交功能（好友关系）、Feed 流
   - 架构含义：需要清晰的路由分组、中间件链、统一错误处理

3. **WebSocket 实时同步**
   - 跨设备实时数据同步（10+ 事件类型）
   - 支持三种连接类型：user-scoped、session-scoped、machine-scoped
   - 事件路由和过滤（避免发送者回显）
   - 心跳检测和自动重连
   - 架构含义：需要高效的连接管理、事件总线、状态同步机制

4. **多设备会话管理**
   - 设备注册和认证
   - 会话元数据和 Agent 状态独立版本控制
   - 会话活跃度跟踪和批量更新优化
   - 架构含义：需要缓存层、后台任务处理、乐观锁机制

5. **加密推送通知**
   - 任务完成、权限请求等通知
   - 通知内容加密
   - 多设备 token 管理
   - 架构含义：需要与 FCM/APNs 集成，异步通知派发

6. **加密签名认证**
   - Ed25519 公钥签名验证
   - Bearer token 会话管理
   - Token 缓存和失效机制
   - 架构含义：需要加密库兼容层、高性能 token 验证

**非功能性需求（NFRs）：**

1. **性能要求**
   - 内存占用峰值 < 500MB（10 个并发设备场景）
   - API 响应时间 P95 < 200ms（正常负载）
   - WebSocket 消息延迟 < 100ms
   - 支持至少 50 个并发 WebSocket 连接
   - 服务启动到就绪 < 10 秒
   - **架构影响**：需要内存高效的数据结构（BigCache、连接池）、Goroutine 优化、SQLite 性能调优

2. **兼容性要求**
   - API 端点 100% 兼容（路径、方法、参数）
   - 请求/响应 JSON schema 完全一致
   - HTTP 状态码和错误格式保持一致
   - WebSocket 事件类型和 payload 格式兼容
   - **架构影响**：需要详细的集成测试套件、契约测试、端到端验证

3. **可靠性要求**
   - 7×24 小时持续运行无内存泄漏
   - 数据库事务 ACID 保证
   - WebSocket 自动重连机制
   - 优雅关闭（等待请求完成）
   - **架构影响**：需要资源清理机制、错误恢复、健康检查

4. **安全要求**
   - 零知识架构验证（服务器无法解密用户数据）
   - 所有通信使用 TLS 1.3+
   - 防止 SQL 注入（参数化查询）
   - 敏感数据加密存储
   - **架构影响**：需要加密服务抽象、安全的配置管理

5. **部署要求**
   - 零外部依赖（无需 PostgreSQL/Redis/MinIO）
   - 单二进制文件部署
   - Docker 镜像 < 100MB
   - 支持环境变量和配置文件
   - **架构影响**：嵌入式数据库、内存缓存、本地文件存储

### 规模与复杂度

- **主要领域**：API Backend + Real-time Communication Service
- **复杂度等级**：中高复杂度
  - **技术复杂度**：高（加密兼容性、实时通信、兼容性约束）
  - **业务逻辑复杂度**：中等（会话管理、多设备同步、社交功能）
  - **集成复杂度**：中等（GitHub OAuth、推送通知、文件存储）
- **预估架构组件数量**：15-20 个主要模块
  - API 层（路由、中间件、处理器）
  - 业务逻辑层（认证、会话、社交、Feed）
  - 存储层（数据库、缓存、文件）
  - 实时通信层（WebSocket 管理、事件路由）
  - 工具层（加密、日志、监控）

**规模指标：**
- 原 Node.js 项目：~8,700 行 TypeScript
- 预估 Go 项目：10,000-12,000 行（考虑显式错误处理和类型定义）
- 数据模型：18 个 Prisma 模型 → 18 个 GORM/sqlc 模型
- API 端点：50+ REST endpoints + 10+ WebSocket 事件
- 迁移工作量：预估 10-15 周全职开发（分三阶段）

### 技术约束与依赖

**硬性兼容性约束：**

1. **API 契约不可变**
   - 所有 REST 端点的路径、HTTP 方法、请求/响应格式必须保持完全一致
   - WebSocket 事件名称、payload 结构、连接握手协议必须兼容
   - 错误响应格式（`{error: string}`）必须一致
   - **架构决策影响**：需要使用原项目的 Zod schemas 作为 Go 类型定义的参考

2. **数据库 Schema 一致性**
   - Prisma schema 定义的所有模型、字段、关系、索引必须在 Go 中精确映射
   - 保留所有唯一约束、复合索引
   - 保留 JSON 字段结构和版本控制机制
   - **架构决策影响**：推荐使用 sqlc 而非 GORM，以确保 SQL 级别的精确控制

3. **加密和认证协议兼容**
   - Ed25519 签名验证流程必须保持不变（使用 tweetnacl 等效库）
   - Token 格式必须兼容或提供迁移路径（privacy-kit → Go crypto）
   - 服务端对称加密（KeyTree 派生）需要在 Go 中重新实现
   - **架构决策影响**：这是最高风险点，需要在 MVP 阶段优先验证

4. **WebSocket 协议层**
   - Socket.IO 不仅是 WebSocket，还包括轮询回退、自定义握手
   - 推荐使用 `googollee/go-socket.io` 保持完全兼容
   - 长期可考虑迁移到纯 WebSocket（需客户端配合）
   - **架构决策影响**：短期兼容优先，长期演进规划

**资源约束：**

1. **内存限制 < 500MB**
   - 必须在 1GB RAM VPS 上稳定运行（系统预留 > 200MB）
   - 影响缓存策略（BigCache 配置）、连接池大小、数据结构选择
   - **架构决策影响**：需要内存分析和优化工具（pprof）

2. **并发能力基线**
   - 最少支持 50 个并发 WebSocket 连接
   - 需要高效的 Goroutine 管理（worker pool 模式）
   - **架构决策影响**：Goroutines + Channels + Context 并发模型

**已知技术依赖（原项目 → Go 映射）：**

| 原项目依赖 | Go 替代方案 | 兼容性风险 |
|-----------|------------|-----------|
| Fastify | Gin | 低（中间件生态相似） |
| Prisma | GORM / sqlc | 中（需精确映射 JSON 字段） |
| Socket.IO | go-socket.io | 中高（协议复杂） |
| Redis | BigCache + 事件总线 | 低（进程内通信） |
| MinIO | 本地文件存储 | 低（接口抽象） |
| privacy-kit | Go crypto（自实现） | **高**（需详细验证） |
| sharp | bimg (libvips) | 中（需 CGo） |
| tweetnacl | Go nacl/ed25519 | 中（签名验证兼容） |

### 跨领域关注点识别

1. **认证和授权**
   - 贯穿所有 API 端点和 WebSocket 连接
   - 需要统一的中间件链（Bearer auth、token 验证、用户上下文注入）
   - **架构模式**：中间件 + 请求上下文（Gin Context）

2. **版本控制和冲突解决**
   - 所有可变加密字段（metadata, agentState, settings, KV values）
   - 乐观锁机制（`UPDATE ... WHERE version = ?`）
   - 冲突时返回当前值，客户端合并后重试
   - **架构模式**：数据访问层统一封装版本检查

3. **事件路由和分发**
   - 影响所有实时功能（会话更新、消息、artifact 变更）
   - 需要高效的连接管理和事件过滤
   - 支持跳过发送者连接（避免回显）
   - **架构模式**：事件总线 + 连接注册表（Map[userId][]Connection）

4. **序列号分配**
   - 用户级、会话级、Artifact 级序列号
   - 用于事件排序和增量同步
   - 需要原子递增（数据库级别）
   - **架构模式**：存储层提供 `AllocateSeq(type, id)` 接口

5. **性能和内存优化**
   - 严格的 500MB 内存约束
   - 需要缓存策略（活跃度缓存、token 验证缓存）
   - 批量数据库操作（会话活跃度更新）
   - **架构模式**：后台 Goroutines + Ticker、内存监控

6. **数据加密分层**
   - **客户端加密**：端到端，服务器不可解密（metadata, messages, artifacts）
   - **服务端加密**：对称加密，服务器可解密（GitHub tokens, service tokens）
   - 需要清晰的抽象层区分两种加密
   - **架构模式**：加密服务接口（ClientEncrypted vs ServerEncrypted）

7. **可观测性**
   - 结构化日志（zap/zerolog）
   - Prometheus 指标（HTTP 延迟、WebSocket 连接数、内存使用）
   - 性能分析（pprof）
   - 健康检查端点（liveness/readiness）
   - **架构模式**：中间件注入 + 全局指标采集器

8. **错误处理和恢复**
   - 统一的错误类型和错误码
   - 分层错误处理（API 层 → 业务逻辑层 → 存储层）
   - Panic 恢复中间件
   - **架构模式**：自定义 Error 类型 + 中间件错误转换

## 项目结构设计

### 主要技术域

**API Backend + Real-time Communication Service** - 基于项目需求分析和技术选型

### 技术偏好确认

基于用户输入的技术偏好：

- **ORM 策略**：GORM - 功能丰富、开发速度快、类似 Prisma 体验
- **WebSocket 库**：coder/websocket - 现代化、高性能、长期演进路线
- **项目初始化**：手动搭建 - 完全控制架构和项目结构
- **开发工具链**：最小化配置 - 专注核心功能，暂不引入热重载/Swagger 等工具

### 项目结构方案

**选择的架构模式：标准布局 + 模块化单体 + 分层架构**

**选型理由：**

1. **标准 Go 项目布局（cmd/internal/pkg）**
   - 遵循 [golang-standards/project-layout](https://github.com/golang-standards/project-layout) 社区标准
   - 编译器强制 internal/ 私有性，清晰的公共/私有边界
   - 映射原 Node.js 项目：cmd/server/main.go → index.ts，internal/ → sources/

2. **分层架构（Layered Architecture）**
   - 清晰的关注点分离：handlers (API) → service (业务逻辑) → repository (数据访问) → models (领域)
   - 参考：[Organizing Large-Scale Gin-GORM Service](https://fenixara.com/organizing-a-large-scale-gin-gorm-web-service-a-guide-to-effective-folder-structure/)
   - 便于测试、维护和团队协作

3. **模块化单体（Modular Monolith）**
   - 参考：[GO Modular Monolith](https://medium.com/@arkjuniork.yudistira/go-modular-monolith-part-i-f963da742e81)
   - 按业务域划分模块（auth, session, social, feed），保持与原项目一致性
   - 清晰的模块边界，易于演进到微服务（如需要）

4. **六边形架构原则**
   - 参考：[Clean Architecture & Hexagonal Architecture in Go](https://medium.com/@kemaltf_/clean-architecture-hexagonal-architecture-in-go-a-practical-guide-aca2593b7223)
   - 依赖始终指向内部，基础设施层隔离外部依赖
   - 业务逻辑与技术细节分离，易于测试和替换

### 目录结构

```
happy-server-go/
├── cmd/
│   └── server/                 # 应用程序入口
│       └── main.go            # 初始化和启动
│
├── internal/                   # 私有应用代码
│   ├── api/                   # API 层（Gin）
│   │   ├── handlers/          # HTTP 请求处理器
│   │   │   ├── auth/          # 认证端点
│   │   │   ├── session/       # 会话管理端点
│   │   │   ├── machine/       # 设备管理端点
│   │   │   ├── artifact/      # Artifact 端点
│   │   │   ├── account/       # 账户管理端点
│   │   │   ├── kv/            # 键值存储端点
│   │   │   ├── push/          # 推送通知端点
│   │   │   ├── social/        # 社交功能端点
│   │   │   └── feed/          # Feed 流端点
│   │   ├── middleware/        # Gin 中间件
│   │   │   ├── auth.go        # 认证中间件
│   │   │   ├── cors.go        # CORS 配置
│   │   │   ├── recovery.go    # Panic 恢复
│   │   │   ├── logger.go      # 请求日志
│   │   │   └── metrics.go     # Prometheus 指标
│   │   ├── routes/            # 路由定义
│   │   │   ├── router.go      # 主路由器
│   │   │   └── v1.go          # v1 API 路由
│   │   └── dto/               # 数据传输对象
│   │       ├── request/       # 请求 DTO
│   │       └── response/      # 响应 DTO
│   │
│   ├── websocket/             # WebSocket 层
│   │   ├── server.go          # WebSocket 服务器
│   │   ├── connection.go      # 连接管理
│   │   ├── handlers/          # WebSocket 事件处理器
│   │   │   ├── ping.go
│   │   │   ├── session.go
│   │   │   ├── machine.go
│   │   │   ├── artifact.go
│   │   │   ├── accesskey.go
│   │   │   ├── rpc.go
│   │   │   └── usage.go
│   │   └── types.go           # WebSocket 类型定义
│   │
│   ├── service/               # 业务逻辑层
│   │   ├── auth/              # 认证服务
│   │   │   ├── service.go
│   │   │   ├── token.go       # Token 管理
│   │   │   └── signature.go   # 签名验证
│   │   ├── session/           # 会话管理服务
│   │   │   ├── service.go
│   │   │   ├── cache.go       # 活跃度缓存
│   │   │   └── message.go     # 消息管理
│   │   ├── machine/           # 设备管理服务
│   │   ├── artifact/          # Artifact 服务
│   │   ├── account/           # 账户服务
│   │   ├── kv/                # 键值存储服务
│   │   ├── social/            # 社交服务
│   │   │   ├── service.go
│   │   │   ├── friendship.go  # 好友关系
│   │   │   └── notification.go
│   │   ├── feed/              # Feed 流服务
│   │   ├── push/              # 推送通知服务
│   │   └── upload/            # 文件上传服务
│   │       ├── service.go
│   │       └── image.go       # 图片处理
│   │
│   ├── repository/            # 数据访问层（GORM）
│   │   ├── account.go
│   │   ├── session.go
│   │   ├── machine.go
│   │   ├── artifact.go
│   │   ├── kv.go
│   │   ├── social.go
│   │   ├── feed.go
│   │   └── push.go
│   │
│   ├── domain/                # 领域模型层
│   │   ├── models/            # GORM 模型
│   │   │   ├── account.go
│   │   │   ├── session.go
│   │   │   ├── machine.go
│   │   │   ├── artifact.go
│   │   │   ├── kv.go
│   │   │   ├── social.go
│   │   │   ├── feed.go
│   │   │   └── push.go
│   │   └── types/             # 共享类型
│   │       ├── update_event.go  # WebSocket 事件类型
│   │       └── constants.go
│   │
│   ├── infrastructure/        # 基础设施层
│   │   ├── database/          # 数据库客户端
│   │   │   ├── db.go          # GORM 连接
│   │   │   └── migrations.go  # 迁移管理
│   │   ├── cache/             # 缓存层
│   │   │   ├── bigcache.go    # BigCache 配置
│   │   │   └── types.go
│   │   ├── eventbus/          # 事件总线
│   │   │   ├── bus.go         # 进程内事件总线
│   │   │   └── router.go      # 事件路由
│   │   ├── storage/           # 文件存储
│   │   │   ├── local.go       # 本地文件系统
│   │   │   └── types.go
│   │   └── crypto/            # 加密服务
│   │       ├── keytree.go     # KeyTree 实现
│   │       ├── token.go       # Token 生成/验证
│   │       └── signature.go   # Ed25519 签名
│   │
│   ├── pkg/                   # 内部共享库
│   │   ├── config/            # 配置管理
│   │   │   ├── config.go
│   │   │   └── types.go
│   │   ├── logger/            # 日志封装
│   │   │   └── zap.go
│   │   ├── errors/            # 错误类型
│   │   │   ├── errors.go
│   │   │   └── codes.go
│   │   ├── validator/         # 验证器
│   │   │   └── validator.go
│   │   └── utils/             # 工具函数
│   │       ├── cuid.go        # CUID 生成
│   │       ├── seq.go         # 序列号分配
│   │       └── helpers.go
│   │
│   └── server/                # 服务器组装
│       ├── server.go          # HTTP + WebSocket 服务器
│       ├── dependencies.go    # 依赖注入
│       └── graceful.go        # 优雅关闭
│
├── migrations/                # 数据库迁移（golang-migrate）
│   ├── 000001_init_schema.up.sql
│   ├── 000001_init_schema.down.sql
│   └── ...
│
├── pkg/                       # 公共可重用库（如果需要）
│   └── ...                    # 目前为空，按需添加
│
├── configs/                   # 配置文件
│   ├── config.yaml
│   └── config.dev.yaml
│
├── scripts/                   # 构建和部署脚本
│   ├── build.sh
│   ├── docker-build.sh
│   └── migrate.sh
│
├── docs/                      # 文档
│   ├── architecture.md        # 架构决策文档（本文档）
│   ├── api.md                 # API 文档
│   └── deployment.md          # 部署指南
│
├── .github/                   # GitHub 配置
│   └── workflows/             # CI/CD
│       └── ci.yml
│
├── Dockerfile                 # Docker 构建文件
├── docker-compose.yml         # 本地开发环境
├── .env.example               # 环境变量示例
├── .gitignore
├── go.mod
├── go.sum
├── Makefile                   # 常用命令
└── README.md
```

### 架构层次与依赖关系

**依赖方向（遵循六边形架构）：**

```
handlers (API 层)
    ↓
service (业务逻辑层)
    ↓
repository (数据访问层) ← infrastructure (基础设施层)
    ↓
models (领域模型)
```

**关键原则：**
- 依赖始终指向内部（handlers → service → repository → models）
- Infrastructure 实现接口，被 service 依赖但不依赖 service
- 业务逻辑（service）与技术细节（infrastructure）分离

### 模块映射（Node.js → Go）

| Node.js (sources/app/) | Go (internal/) | 说明 |
|----------------------|---------------|------|
| `api/routes/` | `api/handlers/` + `api/routes/` | REST API 端点 |
| `api/socket/` | `websocket/handlers/` | WebSocket 事件处理 |
| `auth/` | `service/auth/` | 认证服务 |
| `session/` | `service/session/` | 会话管理 |
| `social/` | `service/social/` | 社交功能 |
| `feed/` | `service/feed/` | Feed 流 |
| `kv/` | `service/kv/` | 键值存储 |
| `monitoring/` | `api/middleware/metrics.go` | Prometheus 指标 |
| `modules/encrypt.ts` | `infrastructure/crypto/` | 加密服务 |
| `storage/db.ts` | `infrastructure/database/` | 数据库客户端 |
| `storage/redis.ts` | `infrastructure/cache/` + `infrastructure/eventbus/` | 缓存 + 事件总线 |
| `storage/files.ts` | `infrastructure/storage/` | 文件存储 |

### 架构决策提供的能力

**语言与运行时：**
- **Go 1.21+**：利用最新特性（泛型、错误处理改进、性能优化）
- **模块管理**：go.mod/go.sum 精确的依赖版本控制
- **编译优化**：`CGO_ENABLED=0` 静态编译，单二进制部署

**项目组织：**
- **按功能模块划分**：每个业务域独立的 handler/service/repository 三层
- **清晰的分层**：API → 业务逻辑 → 数据访问 → 基础设施，职责单一
- **依赖注入**：`server/dependencies.go` 手动组装，清晰的依赖关系

**开发体验：**
- **结构清晰**：按业务域和技术层双维度组织，易于导航和理解
- **关注点分离**：每层职责明确，便于单元测试和集成测试
- **可扩展性**：新增功能只需添加对应模块，最小化对现有代码的影响

**迁移友好性：**
- **模块映射清晰**：原 Node.js 项目的每个模块都有明确的 Go 对应
- **渐进式实现**：可按模块逐个迁移（auth → session → artifact → ...）
- **保持兼容性**：API handlers 结构直接映射原项目路由，确保端点一致

**技术栈整合：**
- **Gin 框架**：高性能 HTTP 路由，中间件生态丰富
- **GORM**：功能丰富的 ORM，类似 Prisma 开发体验
- **coder/websocket**：现代化 WebSocket 库，性能优秀
- **BigCache**：GC 优化的内存缓存，低延迟
- **golang-migrate**：数据库迁移管理，版本控制
- **zap**：高性能结构化日志

**注意事项：**

1. **项目初始化**：手动创建目录结构，不使用 CLI 工具生成
2. **迁移文件**：使用 golang-migrate，SQL 文件放在 `migrations/` 目录
3. **配置管理**：支持 YAML 配置文件 + 环境变量覆盖
4. **依赖注入**：手动在 `server/dependencies.go` 中组装，避免过度抽象
5. **测试组织**：测试文件与源文件同目录，使用 `_test.go` 后缀

### 参考资源

**架构模式：**
- [Clean Architecture & Hexagonal Architecture in Go](https://medium.com/@kemaltf_/clean-architecture-hexagonal-architecture-in-go-a-practical-guide-aca2593b7223)
- [How to implement Clean Architecture in Go](https://threedots.tech/post/introducing-clean-architecture/)
- [Prabogo Hexagonal Architecture Guide](https://prabogo.com/docs/architecture.html)

**项目布局：**
- [Standard Go Project Layout](https://github.com/golang-standards/project-layout)
- [GO Modular Monolith](https://medium.com/@arkjuniork.yudistira/go-modular-monolith-part-i-f963da742e81)
- [go-monolith-example](https://github.com/powerman/go-monolith-example)

**Gin-GORM 最佳实践：**
- [Organizing Large-Scale Gin-GORM Service](https://fenixara.com/organizing-a-large-scale-gin-gorm-web-service-a-guide-to-effective-folder-structure/)
- [Gin Framework Project Structure](https://academy.withcodeexample.com/gin-for-beginners-build-apis-with-golang-easily/gin-framework-project-structure)
- [Real-world Projects with Gin](https://withcodeexample.com/chapter-10-real-world-projects-and-best-practices-with-gin/)

## 核心架构决策

### 决策优先级分析

**关键决策（阻塞实现）：**
- privacy-kit 加密兼容实现（最高优先级）
- GORM 配置和 SQLite 优化
- WebSocket Hub 模式实现
- 配置管理和环境变量加载

**重要决策（塑造架构）：**
- Token 缓存策略
- 连接限制和保护机制
- 健康检查端点设计
- 优雅关闭流程

**延后决策（Post-MVP）：**
- 高级监控和追踪
- 性能优化细节
- 集群部署支持

### 数据架构决策

#### 数据库：SQLite + GORM

**决策理由：**
- 符合"零外部依赖"目标
- 对于中小规模部署性能充足（< 100K hits/day）
- GORM 提供类似 Prisma 的开发体验

**GORM 配置：**

```go
// 生产环境配置
config := &gorm.Config{
    Logger: logger.Default.LogMode(logger.Warn),  // 只记录警告和错误
    PrepareStmt: true,                            // 预编译语句缓存
    DisableForeignKeyConstraintWhenMigrating: true, // SQLite 限制
}

// 连接池配置
sqlDB, _ := db.DB()
sqlDB.SetMaxOpenConns(25)            // SQLite 单文件，不需要太多连接
sqlDB.SetMaxIdleConns(5)             // 保持少量空闲连接
sqlDB.SetConnMaxLifetime(5 * time.Minute)

// 开发环境配置
config := &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),  // 显示 SQL 语句
    PrepareStmt: true,
}
```

**配置理由：**
- **MaxOpenConns: 25** - SQLite 单文件数据库，过多连接无益
- **MaxIdleConns: 5** - 保持少量空闲连接，减少重连开销
- **PrepareStmt: true** - 预编译语句提高重复查询性能
- **LogLevel: Warn** - 生产环境避免大量日志，仅记录问题

**SQLite 性能优化（PRAGMA 设置）：**

```sql
-- 在连接建立后立即执行
PRAGMA journal_mode = WAL;           -- Write-Ahead Logging，允许并发读写
PRAGMA synchronous = NORMAL;         -- 平衡性能和安全，在系统崩溃时可能丢失最后的事务
PRAGMA cache_size = -64000;          -- 64MB 页面缓存
PRAGMA temp_store = MEMORY;          -- 临时表使用内存
PRAGMA mmap_size = 268435456;        -- 256MB 内存映射 I/O
PRAGMA foreign_keys = ON;            -- 启用外键约束
```

**优化理由：**
- **WAL 模式** - 更好的并发性能，允许读写同时进行
- **NORMAL 同步** - 比 FULL 快很多，对于非关键数据可接受的风险
- **大缓存和 mmap** - 在 500MB 内存约束内，分配足够的缓存提高性能
- **内存临时表** - 避免磁盘 I/O

**参考资料：**
- [SQLite Performance Optimization 2025](https://forwardemail.net/en/blog/docs/sqlite-performance-optimization-pragma-chacha20-production-guide)
- [High Performance SQLite](https://highperformancesqlite.com/)

#### 数据迁移策略

**工具选择：** golang-migrate

**迁移文件组织：**

```
migrations/
├── 000001_init_schema.up.sql      # 完整的初始 schema（所有表）
├── 000001_init_schema.down.sql    # 回滚（DROP ALL）
├── 000002_add_indexes.up.sql      # 性能优化索引
├── 000002_add_indexes.down.sql
└── 000003_xxx.up.sql               # 后续增量迁移
```

**决策理由：**
- **单一初始 schema** - 因为这是重写项目，完整 schema 已知，便于新部署
- **SQL 文件** - 精确控制 DDL，避免 ORM 自动迁移的不确定性
- **明确的 up/down** - 清晰的前进和回滚路径

**迁移管理实现：**

```go
// internal/infrastructure/database/migrations.go
import "github.com/golang-migrate/migrate/v4"

func RunMigrations(databaseURL string) error {
    m, err := migrate.New(
        "file://migrations",
        databaseURL,
    )
    if err != nil {
        return err
    }

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }

    return nil
}
```

#### 数据验证策略

**工具选择：** go-playground/validator (Gin 内置)

**验证层次：**
1. **DTO 层验证**（请求参数）：使用 struct tags
2. **Service 层验证**（业务规则）：自定义验证逻辑
3. **Database 层验证**（约束）：UNIQUE、NOT NULL、CHECK

**示例：**

```go
// internal/api/dto/request/session.go
type CreateSessionRequest struct {
    Tag               string  `json:"tag" binding:"required,max=100"`
    Metadata          string  `json:"metadata" binding:"required"`
    AgentState        *string `json:"agentState,omitempty"`
    DataEncryptionKey *string `json:"dataEncryptionKey,omitempty"`
}

// 在 handler 中使用
func (h *SessionHandler) Create(c *gin.Context) {
    var req dto.CreateSessionRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // 业务逻辑...
}
```

#### 缓存策略

**工具选择：** BigCache（替代 Redis）

**使用场景：**
1. **Token 验证缓存** - 简单内存 Map（见安全决策）
2. **会话活跃度缓存** - 批量更新优化
3. **频繁访问数据** - 用户资料、配置等

**BigCache 配置：**

```go
// internal/infrastructure/cache/bigcache.go
config := bigcache.Config{
    Shards: 1024,                       // 分片数，提高并发性能
    LifeWindow: 24 * time.Hour,         // 默认过期时间
    MaxEntriesInWindow: 1000 * 10 * 60, // 估算的条目数
    MaxEntrySize: 500,                   // 单个条目最大字节数
    HardMaxCacheSize: 128,               // 最大缓存大小 128MB
    OnRemove: nil,                       // 淘汰回调（可选）
    CleanWindow: 5 * time.Minute,       // 清理周期
}

cache, err := bigcache.NewBigCache(config)
```

**内存预算分配：**
- BigCache: 128MB（通用缓存）
- Token Cache: < 10MB（简单 Map）
- GORM 连接池: < 20MB
- WebSocket 连接: 约 200MB（1000 连接 × 200KB）
- 应用代码: < 100MB
- **总计: < 500MB** ✅

---

### 认证与安全决策

#### privacy-kit 兼容实现（最高优先级）

**决策：完全重新实现**

**实现策略：**

**Phase 1: 研究和原型（优先级：P0）**
1. 分析 `privacy-kit` 源码，理解加密算法和参数
2. 创建独立的兼容性测试项目：
   ```
   test-compat/
   ├── node/          # Node.js 生成测试数据
   │   └── generate.js
   └── go/            # Go 验证测试数据
       └── verify.go
   ```
3. 生成测试向量：
   - 持久化 token（不同用户 ID、extras）
   - 临时 token（不同 TTL）
   - KeyTree 对称加密（不同路径）
4. Go 实现验证/解密，迭代直到 100% 匹配

**Phase 2: 集成到主项目**

```go
// internal/infrastructure/crypto/token.go
package crypto

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "time"
)

type TokenService struct {
    masterSecret []byte
    keyTree      *KeyTree
}

// 创建持久化 token
func (s *TokenService) CreatePersistentToken(
    userID string,
    extras map[string]interface{},
) (string, error) {
    // 实现与 privacy-kit 兼容的 token 生成
    // 关键：确保签名算法、编码格式完全一致
}

// 验证 token
func (s *TokenService) VerifyToken(token string) (*TokenClaims, error) {
    // 解析和验证签名
}

// 创建临时 token（如 GitHub OAuth）
func (s *TokenService) CreateEphemeralToken(
    userID string,
    ttl time.Duration,
) (string, error) {
    // 带过期时间的 token
}
```

**Phase 3: KeyTree 对称加密**

```go
// internal/infrastructure/crypto/keytree.go
type KeyTree struct {
    masterKey []byte
}

// 派生子密钥
func (kt *KeyTree) DeriveKey(path []string) []byte {
    // 实现 privacy-kit 的密钥派生逻辑
}

// 加密字符串
func (kt *KeyTree) EncryptString(path []string, plaintext string) (string, error) {
    key := kt.DeriveKey(path)
    // AES-GCM 加密，确保与 privacy-kit 格式兼容
}

// 解密字符串
func (kt *KeyTree) DecryptString(path []string, ciphertext string) (string, error) {
    key := kt.DeriveKey(path)
    // 解密
}
```

**验收标准：**
- ✅ Go 能验证 Node.js 生成的所有类型 token
- ✅ Node.js 能验证 Go 生成的所有类型 token
- ✅ 加密/解密互操作性 100% 成功率
- ✅ 性能测试：token 验证 < 1ms，加密/解密 < 5ms

**风险缓解：**
- **高风险项** - 如果兼容性无法达成，Plan B 是协调客户端升级
- **时间预估** - 至少 1 周专注研究和实现
- **测试覆盖** - 必须有完整的兼容性测试套件

**参考资料：**
- [Go crypto 标准库](https://pkg.go.dev/crypto)
- [golang.org/x/crypto](https://pkg.go.dev/golang.org/x/crypto)

#### Token 缓存策略

**决策：简单内存 Map + Mutex**

**实现：**

```go
// internal/service/auth/cache.go
package auth

import "sync"

type TokenCache struct {
    mu     sync.RWMutex
    tokens map[string]*TokenClaims  // token -> claims
}

var globalTokenCache = &TokenCache{
    tokens: make(map[string]*TokenClaims),
}

// 缓存 token 验证结果（永久，除非手动失效）
func (tc *TokenCache) Set(token string, claims *TokenClaims) {
    tc.mu.Lock()
    defer tc.mu.Unlock()
    tc.tokens[token] = claims
}

// 获取缓存的验证结果
func (tc *TokenCache) Get(token string) (*TokenClaims, bool) {
    tc.mu.RLock()
    defer tc.mu.RUnlock()
    claims, exists := tc.tokens[token]
    return claims, exists
}

// 失效用户的所有 tokens（如密码更改、登出）
func (tc *TokenCache) InvalidateUser(userID string) {
    tc.mu.Lock()
    defer tc.mu.Unlock()
    for token, claims := range tc.tokens {
        if claims.UserID == userID {
            delete(tc.tokens, token)
        }
    }
}
```

**决策理由：**
- **永久缓存** - Token 验证结果不会改变（除非密钥轮换）
- **内存占用低** - 每个 token 约 200 字节，1000 个活跃用户 < 1MB
- **简单可靠** - 无需处理 TTL、淘汰策略
- **提供失效接口** - 支持主动清除（安全需求）

#### 密钥管理方案

**决策：环境变量**

**实现：**

```go
// internal/pkg/config/config.go
type Config struct {
    Port         int    `envconfig:"PORT" default:"3005"`
    DatabaseURL  string `envconfig:"DATABASE_URL" required:"true"`
    MasterSecret string `envconfig:"HANDY_MASTER_SECRET" required:"true"`
    // ... 其他配置
}

func LoadConfig() (*Config, error) {
    var cfg Config

    // 从环境变量加载
    if err := envconfig.Process("", &cfg); err != nil {
        return nil, err
    }

    // 验证主密钥长度
    if len(cfg.MasterSecret) < 32 {
        return nil, errors.New("HANDY_MASTER_SECRET must be at least 32 characters")
    }

    return &cfg, nil
}
```

**部署安全实践：**

1. **`.env.example` 文件**（不包含真实密钥）：
```bash
PORT=3005
DATABASE_URL=./data/happy.db
HANDY_MASTER_SECRET=generate-a-secure-random-key-at-least-32-chars
```

2. **生成主密钥的推荐方法**（文档说明）：
```bash
# Linux/macOS
openssl rand -base64 32

# 或使用 Go
go run -c 'import "crypto/rand"; import "encoding/base64";
b := make([]byte, 32); rand.Read(b);
print(base64.StdEncoding.EncodeToString(b))'
```

3. **Docker 部署**：
```bash
docker run -e HANDY_MASTER_SECRET="xxx" happy-server-go
```

4. **Kubernetes 部署**：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: happy-server-secrets
type: Opaque
data:
  master-secret: <base64-encoded-secret>
```

**决策理由：**
- ✅ 符合 12-factor app 原则
- ✅ 容器化友好
- ✅ 避免误提交到版本控制
- ✅ 易于密钥轮换

---

### WebSocket 实现策略决策

#### 连接管理：Hub/Manager 模式

**决策：采用 Go 标准的 Hub 模式**

**实现：**

```go
// internal/websocket/hub.go
package websocket

import "sync"

type ConnectionType string

const (
    UserScoped    ConnectionType = "user-scoped"
    SessionScoped ConnectionType = "session-scoped"
    MachineScoped ConnectionType = "machine-scoped"
)

type Connection struct {
    UserID      string
    SessionID   string         // 可选，仅 session-scoped
    MachineID   string         // 可选，仅 machine-scoped
    Type        ConnectionType
    Conn        *websocket.Conn
    Send        chan []byte    // 发送队列
}

type Hub struct {
    mu           sync.RWMutex
    connections  map[string][]*Connection  // userID -> connections
    register     chan *Connection
    unregister   chan *Connection
    broadcast    chan *BroadcastMessage
}

type BroadcastMessage struct {
    UserID          string
    Event           *UpdateEvent
    Filter          RecipientFilter
    SkipConnection  *Connection  // 跳过发送者（避免回显）
}

// 独立 goroutine 运行
func (h *Hub) Run() {
    for {
        select {
        case conn := <-h.register:
            h.addConnection(conn)

        case conn := <-h.unregister:
            h.removeConnection(conn)
            close(conn.Send)

        case msg := <-h.broadcast:
            h.routeEvent(msg)
        }
    }
}

// 添加连接
func (h *Hub) addConnection(conn *Connection) {
    h.mu.Lock()
    defer h.mu.Unlock()

    // 检查连接数限制
    if len(h.connections[conn.UserID]) >= MaxConnectionsPerUser {
        // 拒绝或断开最旧的连接
        return
    }

    h.connections[conn.UserID] = append(
        h.connections[conn.UserID],
        conn,
    )
}

// 路由事件到目标连接
func (h *Hub) routeEvent(msg *BroadcastMessage) {
    h.mu.RLock()
    defer h.mu.RUnlock()

    conns := h.connections[msg.UserID]
    for _, conn := range conns {
        // 跳过发送者
        if conn == msg.SkipConnection {
            continue
        }

        // 根据过滤器和连接类型决定是否发送
        if h.shouldSendTo(conn, msg.Event, msg.Filter) {
            select {
            case conn.Send <- msg.Event.ToJSON():
            default:
                // 发送队列满，断开连接
                h.unregister <- conn
            }
        }
    }
}

// 判断是否应该发送给此连接
func (h *Hub) shouldSendTo(
    conn *Connection,
    event *UpdateEvent,
    filter RecipientFilter,
) bool {
    switch filter {
    case AllInterestedInSession:
        // 发送给所有关注此会话的连接
        return conn.Type == UserScoped ||
            (conn.Type == SessionScoped && conn.SessionID == event.SessionID)

    case UserScopedOnly:
        return conn.Type == UserScoped

    case MachineScopedOnly:
        return conn.Type == MachineScoped ||
            (conn.Type == MachineScoped && conn.MachineID == event.MachineID) ||
            conn.Type == UserScoped

    case AllUserAuthenticated:
        return true
    }
    return false
}
```

**每个连接的读写 goroutines：**

```go
// internal/websocket/connection.go

// 读取客户端消息
func (c *Connection) ReadPump(hub *Hub) {
    defer func() {
        hub.unregister <- c
        c.Conn.Close()
    }()

    c.Conn.SetReadDeadline(time.Now().Add(PingTimeout))
    c.Conn.SetPongHandler(func(string) error {
        c.Conn.SetReadDeadline(time.Now().Add(PingTimeout))
        return nil
    })

    for {
        var event IncomingEvent
        err := c.Conn.ReadJSON(&event)
        if err != nil {
            break
        }

        // 速率限制检查
        if !c.rateLimiter.Allow() {
            // 超过速率限制
            continue
        }

        // 处理事件（分发到对应 handler）
        HandleEvent(c, &event, hub)
    }
}

// 写入消息到客户端
func (c *Connection) WritePump() {
    ticker := time.NewTicker(PingInterval)
    defer func() {
        ticker.Stop()
        c.Conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.Send:
            c.Conn.SetWriteDeadline(time.Now().Add(WriteWait))
            if !ok {
                // Hub 关闭了 channel
                c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.Conn.WriteMessage(websocket.TextMessage, message); err != nil {
                return
            }

        case <-ticker.C:
            c.Conn.SetWriteDeadline(time.Now().Add(WriteWait))
            if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

**决策理由：**
- ✅ Go 社区标准模式（gorilla/websocket 文档推荐）
- ✅ 高并发性能（channel 通信，无锁读取）
- ✅ 清晰的关注点分离（Hub 管理连接，Connection 处理 I/O）
- ✅ 易于扩展（添加连接池、监控等）

#### 事件序列化：JSON

**决策：使用 JSON（与原项目一致）**

**事件类型定义：**

```go
// internal/domain/types/update_event.go
package types

import "encoding/json"

type UpdateEvent struct {
    Type      string          `json:"type"`
    Payload   json.RawMessage `json:"payload"`
    Timestamp int64           `json:"timestamp"`
    SessionID string          `json:"sessionId,omitempty"`
    MachineID string          `json:"machineId,omitempty"`
}

// 具体事件类型
const (
    EventNewMessage      = "new-message"
    EventNewSession      = "new-session"
    EventUpdateSession   = "update-session"
    EventNewMachine      = "new-machine"
    EventUpdateMachine   = "update-machine"
    EventNewArtifact     = "new-artifact"
    EventUpdateArtifact  = "update-artifact"
    EventDeleteArtifact  = "delete-artifact"
    EventSessionActivity = "session-activity"  // 临时事件
    EventMachineActivity = "machine-activity"  // 临时事件
    EventKVBatchUpdate   = "kv-batch-update"
    // ... 更多事件类型
)

// 序列化为 JSON
func (e *UpdateEvent) ToJSON() []byte {
    b, _ := json.Marshal(e)
    return b
}

// 从 JSON 反序列化
func ParseUpdateEvent(data []byte) (*UpdateEvent, error) {
    var event UpdateEvent
    err := json.Unmarshal(data, &event)
    return &event, err
}
```

**决策理由：**
- ✅ 与 Node.js 版本 100% 格式兼容
- ✅ 人类可读，易于调试
- ✅ 客户端零修改
- ✅ 性能对此项目足够（非高频交易）

#### 连接限制和保护

**配置常量：**

```go
// internal/websocket/limits.go
package websocket

import (
    "time"
    "golang.org/x/time/rate"
)

const (
    // 连接限制
    MaxConnectionsPerUser = 20               // 每用户最大连接数

    // 消息限制
    MaxMessagesPerSecond = 100               // 每连接每秒最大消息数
    MaxMessageSize       = 1 * 1024 * 1024   // 1MB 最大消息大小

    // 超时配置（与原项目一致）
    PingInterval   = 15 * time.Second        // 心跳间隔
    PingTimeout    = 45 * time.Second        // 心跳超时
    WriteWait      = 10 * time.Second        // 写入超时
    ConnectTimeout = 20 * time.Second        // 连接建立超时
)

// 每连接速率限制器
type ConnectionRateLimiter struct {
    limiter *rate.Limiter
}

func NewConnectionRateLimiter() *ConnectionRateLimiter {
    return &ConnectionRateLimiter{
        limiter: rate.NewLimiter(
            rate.Limit(MaxMessagesPerSecond),
            MaxMessagesPerSecond, // burst
        ),
    }
}

func (crl *ConnectionRateLimiter) Allow() bool {
    return crl.limiter.Allow()
}
```

**实施策略：**
1. **连接数限制** - 在 Hub.addConnection 中检查
2. **消息频率限制** - 在 Connection.ReadPump 中使用 rate.Limiter
3. **消息大小限制** - 在 websocket.Conn 配置 `SetReadLimit(MaxMessageSize)`
4. **心跳超时** - 在 Connection.ReadPump 中设置 deadline

**决策理由：**
- ✅ 防止单用户耗尽服务器资源
- ✅ 防止恶意客户端攻击
- ✅ 保持与原项目一致的超时配置
- ✅ 使用标准库 `golang.org/x/time/rate` 实现令牌桶算法

---

### 基础设施与部署决策

#### 配置管理：envconfig + 手动 YAML

**实现：**

```go
// internal/pkg/config/config.go
package config

import (
    "os"
    "github.com/kelseyhightower/envconfig"
    "gopkg.in/yaml.v3"
)

type Config struct {
    // 服务器配置
    Port        int    `yaml:"port" envconfig:"PORT" default:"3005"`
    Host        string `yaml:"host" envconfig:"HOST" default:"0.0.0.0"`

    // 数据库配置
    DatabaseURL string `yaml:"database_url" envconfig:"DATABASE_URL" default:"./data/happy.db"`

    // 加密配置
    MasterSecret string `yaml:"-" envconfig:"HANDY_MASTER_SECRET" required:"true"`

    // Metrics 配置
    MetricsEnabled bool `yaml:"metrics_enabled" envconfig:"METRICS_ENABLED" default:"true"`
    MetricsPort    int  `yaml:"metrics_port" envconfig:"METRICS_PORT" default:"9090"`

    // WebSocket 配置
    MaxConnectionsPerUser int `yaml:"max_connections_per_user" envconfig:"MAX_CONNECTIONS_PER_USER" default:"20"`

    // 日志配置
    LogLevel string `yaml:"log_level" envconfig:"LOG_LEVEL" default:"info"`

    // 环境
    Environment string `yaml:"environment" envconfig:"ENVIRONMENT" default:"development"`
}

func Load() (*Config, error) {
    var cfg Config

    // 1. 尝试加载 YAML 配置文件（可选）
    configFile := os.Getenv("CONFIG_FILE")
    if configFile == "" {
        configFile = "./configs/config.yaml"
    }

    if data, err := os.ReadFile(configFile); err == nil {
        yaml.Unmarshal(data, &cfg)
    }

    // 2. 环境变量覆盖 YAML 配置
    if err := envconfig.Process("", &cfg); err != nil {
        return nil, err
    }

    // 3. 验证
    if err := cfg.Validate(); err != nil {
        return nil, err
    }

    return &cfg, nil
}

func (c *Config) Validate() error {
    if len(c.MasterSecret) < 32 {
        return errors.New("HANDY_MASTER_SECRET must be at least 32 characters")
    }
    return nil
}
```

**配置文件示例：**

```yaml
# configs/config.yaml
port: 3005
host: "0.0.0.0"
database_url: "./data/happy.db"

metrics_enabled: true
metrics_port: 9090

max_connections_per_user: 20

log_level: "info"
environment: "production"

# 敏感配置通过环境变量提供：
# HANDY_MASTER_SECRET
```

**决策理由：**
- ✅ 轻量级（envconfig 是小型库）
- ✅ 环境变量优先（12-factor app）
- ✅ 可选的 YAML 用于复杂配置
- ✅ 类型安全且易于测试

#### 健康检查端点（与原项目一致）

**实现：**

```go
// internal/api/handlers/health.go
package handlers

import (
    "time"
    "github.com/gin-gonic/gin"
)

type HealthHandler struct {
    db *gorm.DB
}

// 主服务健康检查（端口 3005）
func (h *HealthHandler) Check(c *gin.Context) {
    // 测试数据库连接
    var result int
    err := h.db.Raw("SELECT 1").Scan(&result).Error

    if err != nil {
        c.JSON(503, gin.H{
            "status":    "error",
            "timestamp": time.Now().Format(time.RFC3339),
            "service":   "happy-server",
            "error":     "Database connectivity failed",
        })
        return
    }

    c.JSON(200, gin.H{
        "status":    "ok",
        "timestamp": time.Now().Format(time.RFC3339),
        "service":   "happy-server",
    })
}

// Metrics 服务器健康检查（端口 9090，简单检查）
func SimpleHealthCheck(c *gin.Context) {
    c.JSON(200, gin.H{
        "status":    "ok",
        "timestamp": time.Now().Format(time.RFC3339),
    })
}
```

**路由注册：**

```go
// 主服务
router.GET("/health", healthHandler.Check)

// Metrics 服务器
metricsRouter.GET("/health", handlers.SimpleHealthCheck)
metricsRouter.GET("/metrics", metricsHandler.Metrics)
```

**决策理由：**
- ✅ 与原 Node.js 项目 100% 一致
- ✅ 主服务检查数据库（真实的就绪性检查）
- ✅ Metrics 服务简单检查（避免循环依赖）
- ✅ 返回格式完全相同

#### 优雅关闭

**实现：**

```go
// internal/server/graceful.go
package server

import (
    "context"
    "os"
    "os/signal"
    "syscall"
    "time"
)

const GracefulShutdownTimeout = 30 * time.Second

func (s *Server) GracefulShutdown() {
    // 监听系统信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    <-quit
    log.Info("Server is shutting down...")

    // 创建超时 context
    ctx, cancel := context.WithTimeout(
        context.Background(),
        GracefulShutdownTimeout,
    )
    defer cancel()

    // 1. 停止接受新连接
    if err := s.httpServer.Shutdown(ctx); err != nil {
        log.Error("HTTP server forced to shutdown:", err)
    }

    // 2. 通知所有 WebSocket 客户端
    s.websocketHub.NotifyShutdown()

    // 3. 等待 WebSocket 连接关闭（最多 10 秒）
    s.websocketHub.WaitForClose(10 * time.Second)

    // 4. 关闭数据库连接
    sqlDB, _ := s.db.DB()
    sqlDB.Close()

    // 5. 清理其他资源
    s.cache.Close()

    log.Info("Server exited gracefully")
}
```

**决策理由：**
- ✅ 30 秒足够完成大多数请求
- ✅ WebSocket 客户端收到通知可以重连
- ✅ 数据库连接安全关闭，避免数据损坏
- ✅ 标准的 Go 优雅关闭模式

---

### 决策影响分析

#### 实现优先级序列

**P0 - 阻塞所有开发（必须首先完成）：**
1. ✅ **privacy-kit 兼容性验证** - 创建独立测试项目，验证加密兼容性
2. ✅ **项目结构搭建** - 创建目录结构，初始化 go.mod
3. ✅ **配置管理** - 实现 Config 加载，支持环境变量

**P1 - 基础设施（支撑其他功能）：**
4. ✅ **数据库连接** - GORM + SQLite 配置，PRAGMA 优化
5. ✅ **日志系统** - zap 配置，结构化日志
6. ✅ **错误处理** - 自定义 Error 类型，统一错误响应
7. ✅ **Token 服务** - 实现 privacy-kit 兼容的 token 生成/验证
8. ✅ **密钥管理** - KeyTree 实现

**P2 - 核心功能（MVP 必需）：**
9. ✅ **认证 API** - 公钥签名认证、终端认证请求
10. ✅ **WebSocket Hub** - 连接管理、事件路由
11. ✅ **会话管理** - CRUD、消息、活跃度缓存
12. ✅ **设备管理** - 注册、状态管理
13. ✅ **Artifact 存储** - header/body 分离、版本控制

**P3 - 扩展功能：**
14. ✅ 账户管理、KV 存储、推送通知、社交功能

#### 跨组件依赖关系

**依赖链：**

```
1. 配置管理 (Config)
   ↓
2. 日志系统 (Logger) + 数据库连接 (DB)
   ↓
3. Token 服务 (Crypto) + Repository 层
   ↓
4. Service 层（业务逻辑）
   ↓
5. API Handlers + WebSocket Handlers
   ↓
6. 路由注册 + 服务器启动
```

**关键接口定义顺序：**

1. **Domain Models** (internal/domain/models/) - 定义所有 GORM 模型
2. **Repository Interfaces** (internal/repository/) - 数据访问接口
3. **Service Interfaces** (internal/service/) - 业务逻辑接口
4. **DTO Definitions** (internal/api/dto/) - 请求/响应类型
5. **Handler Implementation** (internal/api/handlers/) - API 实现

**测试策略：**
- **单元测试** - 每个 service、repository 独立测试（使用 mock）
- **集成测试** - API 端点测试（使用测试数据库）
- **兼容性测试** - 与 Node.js 版本对比（契约测试）
- **端到端测试** - 使用真实客户端（Claude Code）

---

### 技术栈版本总结

**核心依赖（待初始化项目时确认最新稳定版本）：**

| 依赖 | 用途 | 预期版本 |
|-----|------|---------|
| github.com/gin-gonic/gin | HTTP 框架 | v1.10+ |
| gorm.io/gorm | ORM | v1.25+ |
| gorm.io/driver/sqlite | SQLite 驱动 | v1.5+ |
| github.com/golang-migrate/migrate/v4 | 数据库迁移 | v4.17+ |
| github.com/allegro/bigcache/v3 | 内存缓存 | v3.1+ |
| github.com/coder/websocket | WebSocket | v1.8+ |
| go.uber.org/zap | 结构化日志 | v1.27+ |
| github.com/kelseyhightower/envconfig | 环境变量配置 | v1.5+ |
| golang.org/x/time/rate | 速率限制 | latest |
| golang.org/x/crypto | 加密工具 | latest |
| github.com/h2non/bimg | 图片处理（CGo） | v1.1+ |
| github.com/prometheus/client_golang | Prometheus 指标 | v1.19+ |

**Go 版本：** 1.21+ （利用泛型和最新性能优化）

**注意：** 实际版本在项目初始化时通过 `go get` 获取最新稳定版。

---

## 实现模式与一致性规则

### 模式类别概述

**识别的潜在冲突点**: 52个领域,不同AI代理可能做出不同选择导致代码不兼容

**模式类别**:
1. 命名模式 (15个规则)
2. 结构模式 (12个规则)
3. 格式模式 (10个规则)
4. 通信模式 (8个规则)
5. 流程模式 (7个规则)

---

### 1. 命名模式 (Naming Patterns)

#### 数据库命名规范

**表名**: `snake_case` + 复数形式
```go
func (User) TableName() string { return "users" }
func (Note) TableName() string { return "notes" }
func (Attachment) TableName() string { return "attachments" }
```

**列名**: `snake_case`
```go
type User struct {
    ID        uint      `gorm:"column:id;primaryKey"`
    Username  string    `gorm:"column:username;uniqueIndex"`
    CreatedAt time.Time `gorm:"column:created_at"`
    UpdatedAt time.Time `gorm:"column:updated_at"`
}
```

**外键命名**: `{referenced_table}_id`
```go
type Note struct {
    UserID uint `gorm:"column:user_id;index"`
}
```

**索引命名**: `idx_{table}_{column(s)}`
```sql
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_notes_user_id_created_at ON notes(user_id, created_at);
```

#### API命名规范

**端点命名**: RESTful风格,复数资源名
```go
// ✅ 正确
GET    /api/users
GET    /api/users/:id
POST   /api/users
PUT    /api/users/:id
DELETE /api/users/:id

GET    /api/notes/:id/attachments
POST   /api/notes/:id/attachments
```

**路由参数**: `:paramName` (Gin风格)
```go
router.GET("/api/users/:id", handler.GetUser)
router.GET("/api/notes/:noteId/attachments/:attachmentId", handler.GetAttachment)
```

**查询参数**: `camelCase`
```
GET /api/notes?userId=123&createdAfter=2025-01-01T00:00:00Z
```

#### Go代码命名规范

**文件命名**: `snake_case`
```
user_handler.go
user_service.go
user_repository.go
encryption_service.go
websocket_hub.go
```

**包命名**: 简洁单数小写
```go
package handlers
package service
package repository
package models
package middleware
```

**结构体与接口**: `PascalCase`
```go
type UserService struct {}
type NoteRepository interface {}
type WebSocketHub struct {}
```

**方法与函数**: `PascalCase` (导出) / `camelCase` (私有)
```go
// 导出
func (s *UserService) CreateUser(ctx context.Context, req *CreateUserRequest) error
func NewUserService(repo UserRepository) *UserService

// 私有
func (s *UserService) validateUsername(username string) error
func parseUserID(idStr string) (uint, error)
```

**变量与字段**: `PascalCase` (导出) / `camelCase` (私有)
```go
type User struct {
    UserID   uint   // 导出
    Username string // 导出
    password string // 私有
}

var GlobalConfig *Config // 导出
var defaultTimeout = 5 * time.Second // 私有
```

**常量**: `PascalCase` / `SCREAMING_SNAKE_CASE` (组常量)
```go
const DefaultPort = 3005
const MaxConnections = 1000

const (
    ENV_PRODUCTION  = "production"
    ENV_DEVELOPMENT = "development"
)
```

---

### 2. 结构模式 (Structure Patterns)

#### 项目组织规则

**测试位置**: 同目录co-located
```
internal/handlers/
├── user_handler.go
├── user_handler_test.go
├── note_handler.go
└── note_handler_test.go
```

**层次文件组织**: 单文件per资源/域
```
internal/
├── handlers/          # 一个资源一个文件
│   ├── user_handler.go
│   ├── note_handler.go
│   └── attachment_handler.go
├── service/           # 一个业务域一个文件
│   ├── user_service.go
│   ├── note_service.go
│   └── encryption_service.go
├── repository/        # 接口集中,实现分文件
│   ├── interfaces.go
│   ├── user_repo.go
│   └── note_repo.go
└── models/            # 每个实体独立文件
    ├── user.go
    ├── note.go
    └── common.go
```

**领域模块组织**: 按业务功能分模块
```
internal/
├── user/              # 用户域
│   ├── handler.go
│   ├── service.go
│   └── repository.go
├── note/              # 笔记域
│   ├── handler.go
│   ├── service.go
│   └── repository.go
└── encryption/        # 加密域
    ├── token_service.go
    └── keytree_service.go
```

**配置文件位置**
```
configs/
├── config.yaml           # 默认配置
└── config.example.yaml   # 示例配置(提交到git)

internal/config/
└── config.go             # 配置结构与加载逻辑
```

**中间件与工具**
```
internal/middleware/      # 业务相关中间件
├── auth.go
├── logging.go
└── rate_limit.go

pkg/utils/               # 可复用工具
├── crypto.go
├── validator.go
└── time.go
```

**静态资源与文档**
```
docs/                    # 项目文档
├── architecture.md
├── prd.md
└── api.md

scripts/                 # 脚本工具
├── migrate.sh
└── build.sh
```

---

### 3. 格式模式 (Format Patterns)

#### API响应格式

**成功响应**: 直接返回数据对象/数组
```json
// 单个资源
{
    "userId": 123,
    "username": "testuser",
    "createdAt": "2025-12-09T10:30:00Z"
}

// 列表资源
[
    {"userId": 1, "username": "user1"},
    {"userId": 2, "username": "user2"}
]

// 空列表
[]
```

**错误响应**: 统一错误对象结构
```json
{
    "error": {
        "type": "ValidationError",
        "message": "Username is required",
        "details": {
            "field": "username",
            "constraint": "required"
        }
    }
}
```

**错误类型标准**:
```go
const (
    ErrorTypeValidation   = "ValidationError"   // 400
    ErrorTypeUnauthorized = "UnauthorizedError" // 401
    ErrorTypeForbidden    = "ForbiddenError"    // 403
    ErrorTypeNotFound     = "NotFoundError"     // 404
    ErrorTypeConflict     = "ConflictError"     // 409
    ErrorTypeInternal     = "InternalError"     // 500
)
```

#### JSON序列化规范

**字段命名**: `camelCase`
```go
type UserResponse struct {
    UserID    uint      `json:"userId"`
    Username  string    `json:"username"`
    CreatedAt time.Time `json:"createdAt"`
    UpdatedAt time.Time `json:"updatedAt"`
}
```

**时间格式**: RFC3339 (ISO 8601)
```go
type Response struct {
    CreatedAt time.Time `json:"createdAt"` // "2025-12-09T10:30:00Z"
    UpdatedAt time.Time `json:"updatedAt"` // "2025-12-09T12:45:30Z"
}
```

**布尔值与空值处理**
```go
type UserResponse struct {
    IsActive  bool     `json:"isActive"`           // false也序列化
    Bio       string   `json:"bio,omitempty"`      // 空字符串不序列化
    Avatar    *string  `json:"avatar,omitempty"`   // nil不序列化
    Tags      []string `json:"tags,omitempty"`     // 空数组不序列化
}
```

**数值类型**
```go
type NoteResponse struct {
    NoteID uint   `json:"noteId"`      // 整数作为number
    Size   int64  `json:"size"`        // 文件大小number
    Rating *float64 `json:"rating,omitempty"` // 可空浮点
}
```

#### HTTP状态码使用标准

```go
200 OK              - 成功GET/PUT/PATCH,返回资源
201 Created         - 成功POST创建资源,返回新资源
204 No Content      - 成功DELETE,无返回体

400 Bad Request     - 请求参数格式错误,验证失败
401 Unauthorized    - 未提供认证凭据或凭据无效
403 Forbidden       - 已认证但无权限访问资源
404 Not Found       - 资源不存在
409 Conflict        - 资源冲突(如username重复)

500 Internal Server Error - 未预期的服务器错误
503 Service Unavailable   - 数据库/外部服务不可用
```

---

### 4. 通信模式 (Communication Patterns)

#### WebSocket事件规范

**事件命名**: `resource.action` 点分格式
```go
const (
    // Note事件
    EventNoteCreated      = "note.created"
    EventNoteUpdated      = "note.updated"
    EventNoteDeleted      = "note.deleted"

    // User事件
    EventUserConnected    = "user.connected"
    EventUserDisconnected = "user.disconnected"

    // Sync事件
    EventSyncRequested    = "sync.requested"
    EventSyncCompleted    = "sync.completed"
)
```

**事件消息结构**: 统一包装
```go
type WebSocketMessage struct {
    Event     string          `json:"event"`      // "note.created"
    Data      json.RawMessage `json:"data"`       // 事件特定数据
    Timestamp time.Time       `json:"timestamp"`  // 服务器时间戳
    RequestID string          `json:"requestId,omitempty"` // 请求关联ID
}

// 具体事件数据
type NoteCreatedData struct {
    NoteID    uint      `json:"noteId"`
    UserID    uint      `json:"userId"`
    Title     string    `json:"title"`
    CreatedAt time.Time `json:"createdAt"`
}

// 序列化发送
msg := WebSocketMessage{
    Event:     EventNoteCreated,
    Data:      json.Marshal(noteData),
    Timestamp: time.Now(),
}
```

**事件路由规则**
```go
// 按UserID路由到特定用户的所有连接
hub.BroadcastToUser(userID, msg)

// 广播到所有连接
hub.BroadcastAll(msg)

// 发送到特定连接
conn.Send(msg)
```

#### 日志格式与级别

**日志格式**: 结构化JSON (zap)
```go
logger.Info("user authenticated",
    zap.Uint("userId", user.ID),
    zap.String("username", user.Username),
    zap.String("ip", c.ClientIP()),
    zap.Duration("latency", elapsed),
)

// 输出:
// {"level":"info","ts":1702123456.789,"msg":"user authenticated","userId":123,"username":"test","ip":"127.0.0.1","latency":0.123}
```

**日志级别使用标准**
```go
// Debug - 开发调试信息
logger.Debug("cache hit",
    zap.String("key", key),
    zap.Int("size", len(value)),
)

// Info - 正常业务流程
logger.Info("user login",
    zap.Uint("userId", userID),
    zap.String("method", "password"),
)

// Warn - 异常但可恢复
logger.Warn("cache miss, fetching from db",
    zap.String("key", key),
)

// Error - 错误需要关注
logger.Error("database query failed",
    zap.Error(err),
    zap.String("query", sql),
)

// Fatal - 致命错误,程序退出
logger.Fatal("failed to connect database",
    zap.Error(err),
)
```

**敏感信息处理**: 永不记录
```go
// ❌ 禁止记录
// - password, masterSecret, encryptionKey
// - 完整token (最多记录前8字符)
// - 笔记内容, 附件内容
// - 个人隐私数据

// ✅ 安全日志
logger.Info("token validated",
    zap.String("tokenPrefix", token[:8]),  // 只记录前缀
    zap.Uint("userId", userID),
    zap.Bool("valid", true),
)
```

---

### 5. 流程模式 (Process Patterns)

#### 错误处理模式

**分层错误处理**: Repository → Service → Handler
```go
// Repository层: 返回原始错误,包装为领域错误
func (r *UserRepo) FindByID(ctx context.Context, id uint) (*models.User, error) {
    var user models.User
    if err := r.db.WithContext(ctx).First(&user, id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound // 领域错误
        }
        return nil, fmt.Errorf("query user failed: %w", err)
    }
    return &user, nil
}

// Service层: 业务逻辑错误处理
func (s *UserService) GetUser(ctx context.Context, id uint) (*UserDTO, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err // 传递错误
    }
    return s.toDTO(user), nil
}

// Handler层: 转换为HTTP响应
func (h *UserHandler) GetUser(c *gin.Context) {
    id := parseID(c.Param("id"))
    user, err := h.service.GetUser(c.Request.Context(), id)

    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            c.JSON(404, gin.H{"error": gin.H{
                "type": "NotFoundError",
                "message": "User not found",
            }})
            return
        }

        h.logger.Error("get user failed", zap.Error(err))
        c.JSON(500, gin.H{"error": gin.H{
            "type": "InternalError",
            "message": "Internal server error",
        }})
        return
    }

    c.JSON(200, user)
}
```

**领域错误定义**: 使用sentinel errors
```go
// internal/domain/errors.go
var (
    ErrUserNotFound      = errors.New("user not found")
    ErrUsernameTaken     = errors.New("username already taken")
    ErrInvalidCredentials = errors.New("invalid credentials")
    ErrUnauthorized      = errors.New("unauthorized")
)
```

**禁止panic**: 仅在初始化时使用
```go
// ✅ 允许: 启动时配置错误
func main() {
    config, err := config.Load()
    if err != nil {
        log.Fatal("failed to load config", zap.Error(err))
    }
}

// ❌ 禁止: 运行时panic
func (h *Handler) GetUser(c *gin.Context) {
    user, err := h.service.GetUser(...)
    if err != nil {
        panic(err) // 禁止!应该返回错误响应
    }
}
```

#### Context传递标准

**每个请求创建context**: 设置超时
```go
func (h *NoteHandler) CreateNote(c *gin.Context) {
    // 5秒超时
    ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
    defer cancel()

    note, err := h.service.CreateNote(ctx, req)
    // ...
}
```

**所有service/repository方法第一参数必须是context**
```go
// Service接口
type UserService interface {
    CreateUser(ctx context.Context, req *CreateUserRequest) (*User, error)
    GetUser(ctx context.Context, id uint) (*User, error)
    UpdateUser(ctx context.Context, id uint, req *UpdateUserRequest) error
}

// Repository接口
type UserRepository interface {
    Insert(ctx context.Context, user *models.User) error
    FindByID(ctx context.Context, id uint) (*models.User, error)
    Update(ctx context.Context, user *models.User) error
}
```

**Context值传递**: 仅用于请求范围数据
```go
// 在中间件中设置
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := validateToken(c.GetHeader("Authorization"))
        ctx := context.WithValue(c.Request.Context(), "userID", userID)
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}

// 在handler中获取
func GetUserIDFromContext(ctx context.Context) (uint, bool) {
    userID, ok := ctx.Value("userID").(uint)
    return userID, ok
}
```

#### 数据库事务模式

**使用GORM Transaction辅助函数**
```go
func (s *UserService) CreateUserWithProfile(ctx context.Context, req *CreateUserRequest) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        // 所有操作在同一事务中
        user := &models.User{Username: req.Username}
        if err := tx.Create(user).Error; err != nil {
            return err // 自动回滚
        }

        profile := &models.Profile{UserID: user.ID}
        if err := tx.Create(profile).Error; err != nil {
            return err // 自动回滚
        }

        return nil // 自动提交
    })
}
```

**嵌套事务**: 使用SavePoint
```go
return s.db.Transaction(func(tx *gorm.DB) error {
    // 外层事务

    err := tx.Transaction(func(tx2 *gorm.DB) error {
        // 内层事务 (SavePoint)
        return nil
    })

    return err
})
```

#### 输入验证模式

**Handler层: 格式验证** (使用gin binding)
```go
type CreateNoteRequest struct {
    Title   string `json:"title" binding:"required,min=1,max=200"`
    Content string `json:"content" binding:"required"`
    Tags    []string `json:"tags" binding:"max=10,dive,min=1,max=50"`
}

func (h *NoteHandler) CreateNote(c *gin.Context) {
    var req CreateNoteRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": gin.H{
            "type": "ValidationError",
            "message": err.Error(),
        }})
        return
    }

    // 传递给service
    note, err := h.service.CreateNote(c.Request.Context(), &req)
    // ...
}
```

**Service层: 业务规则验证**
```go
func (s *NoteService) CreateNote(ctx context.Context, req *CreateNoteRequest) (*Note, error) {
    // 业务规则: 检查用户笔记数量限制
    userID := GetUserIDFromContext(ctx)
    count, err := s.repo.CountByUserID(ctx, userID)
    if err != nil {
        return nil, err
    }

    if count >= MaxNotesPerUser {
        return nil, ErrNoteQuotaExceeded
    }

    // 创建笔记
    note := &models.Note{
        UserID:  userID,
        Title:   req.Title,
        Content: req.Content,
    }

    if err := s.repo.Insert(ctx, note); err != nil {
        return nil, err
    }

    return s.toDTO(note), nil
}
```

#### 资源清理模式

**统一使用defer清理资源**
```go
// 文件操作
func (s *AttachmentService) ProcessFile(ctx context.Context, path string) error {
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close() // 确保关闭

    // 处理文件...
    return nil
}

// 数据库连接
func setupDatabase(config *Config) (*gorm.DB, error) {
    db, err := gorm.Open(sqlite.Open(config.DatabaseURL), &gorm.Config{})
    if err != nil {
        return nil, err
    }

    sqlDB, _ := db.DB()
    sqlDB.SetMaxOpenConns(25)

    // 返回db,调用者负责关闭
    return db, nil
}

// 在main.go中
func main() {
    db, err := setupDatabase(config)
    if err != nil {
        log.Fatal(err)
    }

    sqlDB, _ := db.DB()
    defer sqlDB.Close() // 程序退出时关闭

    // ...
}

// Context取消
func (s *Service) LongRunningTask(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel() // 确保释放资源

    // 执行任务...
    return nil
}
```

---

### 强制执行规则 (Enforcement Guidelines)

#### 所有AI代理必须遵守 (MANDATORY)

1. **命名一致性**:
   - 数据库表/列必须使用snake_case
   - Go代码必须遵循标准命名规范(PascalCase导出,camelCase私有)
   - API端点必须使用RESTful复数资源名

2. **错误处理**:
   - 禁止在运行时使用panic
   - 所有错误必须返回并妥善处理
   - Handler层必须将领域错误转换为HTTP响应

3. **Context传递**:
   - 所有service/repository方法第一参数必须是context.Context
   - 请求处理必须设置合理超时(5-30秒)

4. **JSON格式**:
   - API字段名必须使用camelCase
   - 时间必须使用RFC3339格式
   - 错误响应必须使用统一结构

5. **测试覆盖**:
   - 每个public方法必须有对应测试
   - 测试文件必须与源文件同目录

6. **敏感信息**:
   - 禁止记录password, secret, key, token完整内容
   - 禁止在错误消息中暴露内部实现细节

#### 模式验证方法

**自动化检查**:
```bash
# 代码格式检查
gofmt -l .
golangci-lint run

# 命名检查
go vet ./...

# 测试覆盖率
go test -cover ./...
```

**Code Review检查清单**:
- [ ] 命名是否符合规范?
- [ ] 错误处理是否正确?(无panic,有错误包装)
- [ ] Context是否正确传递?
- [ ] JSON tag是否使用camelCase?
- [ ] 是否有敏感信息泄露?
- [ ] 测试覆盖是否充分?

#### 模式更新流程

1. 发现新的冲突点或改进建议
2. 在architecture.md中提出修改
3. 团队讨论确认
4. 更新本文档
5. 通知所有开发者(AI代理与人类)

---

### 模式示例

#### ✅ 良好示例

**完整的Handler实现**:
```go
package handlers

import (
    "context"
    "errors"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"
)

type CreateNoteRequest struct {
    Title   string `json:"title" binding:"required,min=1,max=200"`
    Content string `json:"content" binding:"required"`
}

type NoteResponse struct {
    NoteID    uint      `json:"noteId"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"createdAt"`
}

type NoteHandler struct {
    service NoteService
    logger  *zap.Logger
}

func (h *NoteHandler) CreateNote(c *gin.Context) {
    // 1. 验证请求
    var req CreateNoteRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": gin.H{
            "type":    "ValidationError",
            "message": err.Error(),
        }})
        return
    }

    // 2. 创建context with timeout
    ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
    defer cancel()

    // 3. 调用service
    note, err := h.service.CreateNote(ctx, &req)
    if err != nil {
        h.handleError(c, err)
        return
    }

    // 4. 返回成功响应
    c.JSON(http.StatusCreated, &NoteResponse{
        NoteID:    note.ID,
        Title:     note.Title,
        Content:   note.Content,
        CreatedAt: note.CreatedAt,
    })
}

func (h *NoteHandler) handleError(c *gin.Context, err error) {
    if errors.Is(err, ErrNoteQuotaExceeded) {
        c.JSON(http.StatusForbidden, gin.H{"error": gin.H{
            "type":    "QuotaError",
            "message": "Note quota exceeded",
        }})
        return
    }

    h.logger.Error("create note failed", zap.Error(err))
    c.JSON(http.StatusInternalServerError, gin.H{"error": gin.H{
        "type":    "InternalError",
        "message": "Internal server error",
    }})
}
```

#### ❌ 反模式 (Anti-Patterns)

**错误示例1: 不正确的命名**
```go
// ❌ 错误: 表名单数,列名驼峰
type User struct {
    ID       uint   `gorm:"column:ID;primaryKey"`    // 应该用小写id
    UserName string `gorm:"column:UserName"`         // 应该用user_name
}
func (User) TableName() string { return "user" }     // 应该用users

// ❌ 错误: 文件名驼峰,包名复数
// 文件: internal/handlers/UserHandler.go              // 应该用user_handler.go
package handler                                       // 应该用handlers
```

**错误示例2: 不正确的错误处理**
```go
// ❌ 错误: 使用panic
func (h *Handler) GetUser(c *gin.Context) {
    user, err := h.service.GetUser(...)
    if err != nil {
        panic(err) // 应该返回HTTP错误响应
    }
    c.JSON(200, user)
}

// ❌ 错误: 吞掉错误
func (s *Service) UpdateUser(ctx context.Context, id uint, req *UpdateRequest) error {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil // 应该返回错误!
    }
    // ...
}

// ❌ 错误: 暴露内部错误
func (h *Handler) CreateUser(c *gin.Context) {
    _, err := h.service.CreateUser(...)
    if err != nil {
        // 直接暴露数据库错误
        c.JSON(500, gin.H{"error": err.Error()}) // 可能包含敏感信息!
    }
}
```

**错误示例3: 缺少Context**
```go
// ❌ 错误: 方法缺少context参数
type UserService interface {
    GetUser(id uint) (*User, error)          // 应该有ctx参数
    UpdateUser(id uint, req *UpdateRequest) error
}

// ❌ 错误: 没有设置超时
func (h *Handler) CreateNote(c *gin.Context) {
    // 直接使用c.Request.Context(),没有超时控制
    note, err := h.service.CreateNote(c.Request.Context(), req)
    // ...
}
```

**错误示例4: JSON格式不一致**
```go
// ❌ 错误: 使用snake_case
type UserResponse struct {
    UserID   uint   `json:"user_id"`    // 应该用userId
    UserName string `json:"user_name"`  // 应该用username
}

// ❌ 错误: 使用自定义时间格式
type Response struct {
    CreatedAt string `json:"createdAt"` // 应该用time.Time自动序列化为RFC3339
}

// ❌ 错误: 不统一的错误响应
func (h *Handler) GetUser(c *gin.Context) {
    _, err := h.service.GetUser(...)
    if err != nil {
        c.JSON(404, gin.H{"msg": "not found"}) // 应该用统一的error结构
    }
}
```

**错误示例5: 敏感信息泄露**
```go
// ❌ 错误: 记录完整token
logger.Info("user login",
    zap.String("token", token), // 应该只记录前8字符!
    zap.Uint("userId", userID),
)

// ❌ 错误: 记录密码
logger.Debug("validating user",
    zap.String("username", username),
    zap.String("password", password), // 绝对不能记录密码!
)

// ❌ 错误: 在响应中返回密码
type UserResponse struct {
    UserID   uint   `json:"userId"`
    Username string `json:"username"`
    Password string `json:"password"` // 绝对不能返回密码!
}
```

---

### 实施建议

**对AI代理的指令**:
1. 在开始编写任何代码前,完整阅读本实现模式文档
2. 每个命名决策都必须参考命名模式章节
3. 所有错误处理必须遵循分层错误处理模式
4. 代码提交前自查反模式清单
5. 当遇到文档未覆盖的场景,查询现有代码或询问架构师

**文档维护**:
- 当发现新的冲突模式,立即更新本文档
- 每次重大技术栈升级后,审查并更新模式
- 收集实际开发中的良好/不良案例,补充到示例章节


---

## 项目结构与边界

### 需求到架构映射

基于PRD和原Node.js项目分析,功能需求映射到Go架构组件:

**功能域映射表:**

| 功能域 | 原Node.js路径 | Go模块 | 说明 |
|--------|--------------|--------|------|
| 认证与会话 | `routes/auth.ts`, `sessions.ts` | `internal/auth/`, `internal/session/` | 公钥签名认证、会话CRUD |
| 设备管理 | `routes/devices.ts` | `internal/device/` | 设备注册、状态、元数据 |
| Artifacts存储 | `routes/artifacts.ts` | `internal/artifact/` | header/body分离、版本控制 |
| 用户与账户 | `routes/accounts.ts`, `users.ts` | `internal/user/`, `internal/account/` | 资料、头像、设置 |
| KV存储 | `routes/kv.ts` | `internal/kv/` | CRUD、CAS、批量操作 |
| 推送通知 | `routes/push.ts` | `internal/push/` | 订阅管理、通知发送 |
| 社交功能 | `routes/friends.ts`, `feeds.ts` | `internal/social/` | 好友关系、Feed流 |
| WebSocket同步 | `api/socket/` | `internal/websocket/` | Hub、事件路由、心跳 |
| 加密服务 | `privacy-kit` | `internal/encryption/` | TokenService、KeyTree |
| 对象存储 | MinIO | `internal/storage/` | 文件上传下载、图片处理 |

---

### 完整项目目录结构

```
happy-server-go/
├── README.md
├── LICENSE
├── .gitignore
├── .env.example
├── go.mod
├── go.sum
│
├── cmd/
│   └── server/
│       └── main.go                 # 应用程序入口
│
├── configs/
│   ├── config.yaml                 # 默认配置
│   └── config.example.yaml         # 配置示例
│
├── internal/
│   ├── config/
│   │   └── config.go               # 配置加载逻辑
│   │
│   ├── server/
│   │   ├── server.go               # HTTP服务器设置
│   │   ├── router.go               # 路由注册
│   │   └── middleware.go           # 全局中间件
│   │
│   ├── domain/
│   │   └── errors.go               # 领域错误定义
│   │
│   ├── models/
│   │   ├── user.go                 # User模型
│   │   ├── session.go              # Session模型
│   │   ├── device.go               # Device模型
│   │   ├── artifact.go             # Artifact模型
│   │   ├── kv.go                   # KV模型
│   │   ├── friend.go               # Friend模型
│   │   ├── feed.go                 # Feed模型
│   │   ├── push_subscription.go    # PushSubscription模型
│   │   └── common.go               # 共享类型和时间戳mixin
│   │
│   ├── repository/
│   │   ├── interfaces.go           # 所有repository接口定义
│   │   ├── user_repo.go
│   │   ├── session_repo.go
│   │   ├── device_repo.go
│   │   ├── artifact_repo.go
│   │   ├── kv_repo.go
│   │   ├── friend_repo.go
│   │   ├── feed_repo.go
│   │   └── push_repo.go
│   │
│   ├── service/
│   │   ├── user_service.go
│   │   ├── session_service.go
│   │   ├── device_service.go
│   │   ├── artifact_service.go
│   │   ├── kv_service.go
│   │   ├── friend_service.go
│   │   ├── feed_service.go
│   │   └── push_service.go
│   │
│   ├── handlers/
│   │   ├── user_handler.go
│   │   ├── session_handler.go
│   │   ├── device_handler.go
│   │   ├── artifact_handler.go
│   │   ├── kv_handler.go
│   │   ├── friend_handler.go
│   │   ├── feed_handler.go
│   │   ├── push_handler.go
│   │   ├── health_handler.go       # 健康检查
│   │   └── auth_handler.go         # 认证端点
│   │
│   ├── middleware/
│   │   ├── auth.go                 # Token验证中间件
│   │   ├── logging.go              # 请求日志中间件
│   │   ├── rate_limit.go           # 速率限制中间件
│   │   ├── cors.go                 # CORS中间件
│   │   └── recovery.go             # Panic恢复中间件
│   │
│   ├── websocket/
│   │   ├── hub.go                  # WebSocket连接管理Hub
│   │   ├── connection.go           # Connection封装
│   │   ├── events.go               # 事件类型定义
│   │   └── router.go               # 事件路由逻辑
│   │
│   ├── encryption/
│   │   ├── token_service.go        # JWT Token服务
│   │   ├── keytree.go              # KeyTree对称加密
│   │   ├── signature.go            # 公钥签名验证
│   │   └── utils.go                # 加密工具函数
│   │
│   ├── storage/
│   │   ├── storage.go              # 存储接口
│   │   ├── local.go                # 本地文件系统存储
│   │   ├── minio.go                # MinIO对象存储(可选)
│   │   └── image.go                # 图片处理(bimg)
│   │
│   └── cache/
│       ├── cache.go                # 缓存接口
│       ├── bigcache.go             # BigCache实现
│       └── token_cache.go          # Token专用缓存
│
├── pkg/
│   └── utils/
│       ├── crypto.go               # 加密工具
│       ├── validator.go            # 验证器
│       ├── time.go                 # 时间工具
│       └── strings.go              # 字符串工具
│
├── migrations/
│   ├── 000001_init_schema.up.sql
│   ├── 000001_init_schema.down.sql
│   └── README.md
│
├── scripts/
│   ├── migrate.sh                  # 数据库迁移脚本
│   ├── build.sh                    # 构建脚本
│   └── dev.sh                      # 开发环境启动脚本
│
├── data/
│   └── .gitkeep                    # SQLite数据库文件目录
│
├── docs/
│   ├── architecture.md             # 架构文档
│   ├── prd.md                      # PRD
│   └── api/
│       └── openapi.yaml            # OpenAPI规范(可选)
│
├── tests/
│   ├── integration/
│   │   ├── user_test.go
│   │   ├── session_test.go
│   │   └── websocket_test.go
│   ├── fixtures/
│   │   └── test_data.go
│   └── mocks/
│       └── repositories.go         # Repository mock
│
└── .github/
    └── workflows/
        ├── ci.yml                  # CI流程
        └── release.yml             # 发布流程
```

---

### 架构边界定义

#### API边界

**RESTful API端点结构 (端口3005)**

```go
// API根路径: /api
router := gin.Default()
apiV1 := router.Group("/api")
{
    // 认证端点 (公开)
    apiV1.POST("/auth/challenge", authHandler.Challenge)
    apiV1.POST("/auth/verify", authHandler.Verify)
    
    // 需要认证的端点
    authenticated := apiV1.Group("")
    authenticated.Use(middleware.Auth())
    {
        // 用户资源
        authenticated.GET("/users/me", userHandler.GetCurrentUser)
        authenticated.PUT("/users/me", userHandler.UpdateProfile)
        authenticated.POST("/users/me/avatar", userHandler.UploadAvatar)
        
        // 会话资源
        authenticated.GET("/sessions", sessionHandler.List)
        authenticated.POST("/sessions", sessionHandler.Create)
        authenticated.DELETE("/sessions/:id", sessionHandler.Delete)
        
        // 设备资源
        authenticated.GET("/devices", deviceHandler.List)
        authenticated.POST("/devices", deviceHandler.Register)
        authenticated.PUT("/devices/:id", deviceHandler.Update)
        
        // Artifacts资源
        authenticated.GET("/artifacts", artifactHandler.List)
        authenticated.POST("/artifacts", artifactHandler.Create)
        authenticated.GET("/artifacts/:id/header", artifactHandler.GetHeader)
        authenticated.GET("/artifacts/:id/body", artifactHandler.GetBody)
        
        // KV存储
        authenticated.GET("/kv/:key", kvHandler.Get)
        authenticated.PUT("/kv/:key", kvHandler.Set)
        authenticated.POST("/kv/batch", kvHandler.BatchUpdate)
        authenticated.POST("/kv/:key/cas", kvHandler.CompareAndSwap)
        
        // 社交功能
        authenticated.GET("/friends", friendHandler.List)
        authenticated.POST("/friends", friendHandler.Add)
        authenticated.DELETE("/friends/:id", friendHandler.Remove)
        authenticated.GET("/feeds", feedHandler.List)
        
        // 推送通知
        authenticated.POST("/push/subscribe", pushHandler.Subscribe)
        authenticated.DELETE("/push/subscribe", pushHandler.Unsubscribe)
    }
}

// 健康检查端点 (公开,主服务端口3005)
router.GET("/health", healthHandler.Check)

// Metrics端点 (独立端口9090)
metricsRouter := gin.Default()
metricsRouter.GET("/health", healthHandler.SimpleCheck)
metricsRouter.GET("/metrics", promhttp.Handler())
```

**WebSocket端点 (端口3005)**

```go
// WebSocket连接端点: /ws
router.GET("/ws", websocketHandler.HandleConnection)

// 连接类型通过查询参数区分:
// - /ws?type=user&token=xxx     (user-scoped连接)
// - /ws?type=session&token=xxx  (session-scoped连接)
// - /ws?type=machine&token=xxx  (machine-scoped连接)
```

#### 组件边界

**层次依赖规则 (单向依赖)**

```
┌─────────────────────────────────────────┐
│  Handler层 (HTTP/WebSocket入口)         │
│  - 参数验证                              │
│  - HTTP状态码映射                        │
└──────────────┬──────────────────────────┘
               ↓ 只能调用Service
┌──────────────┴──────────────────────────┐
│  Service层 (业务逻辑)                    │
│  - 业务规则验证                          │
│  - 跨service协调                         │
└──────────────┬──────────────────────────┘
               ↓ 只能调用Repository
┌──────────────┴──────────────────────────┐
│  Repository层 (数据访问)                 │
│  - CRUD操作                             │
│  - 查询优化                              │
└──────────────┬──────────────────────────┘
               ↓ 只能操作Models
┌──────────────┴──────────────────────────┐
│  Models层 (数据模型)                     │
│  - GORM结构体定义                        │
│  - 表映射                                │
└──────────────┬──────────────────────────┘
               ↓
         ┌─────┴─────┐
         │  Database  │
         │  (SQLite)  │
         └────────────┘

✅ 允许: Handler → Service → Repository → Models → DB
❌ 禁止: Repository → Service, Service → Handler
❌ 禁止: Handler → Repository (跳过Service层)
```

**横切关注点边界**

```go
// Middleware: 在Handler之前执行
// - 访问gin.Context
// - 不直接调用Service层
// - 通过context.Context传递认证信息

type AuthMiddleware struct {
    tokenService encryption.TokenService
    tokenCache   cache.TokenCache
}

// Encryption: 被Service层调用
// - 不依赖业务逻辑
// - 提供纯函数式API

type TokenService interface {
    Generate(claims TokenClaims) (string, error)
    Verify(token string) (*TokenClaims, error)
}

// Cache: 被Service层调用
// - 不知道业务规则
// - 统一的Get/Set/Delete接口

type Cache interface {
    Get(key string) ([]byte, error)
    Set(key string, value []byte, ttl time.Duration) error
    Delete(key string) error
}

// Storage: 被Service层调用
// - 不知道业务逻辑
// - 统一的存储接口

type Storage interface {
    Upload(ctx context.Context, key string, data io.Reader) error
    Download(ctx context.Context, key string) (io.ReadCloser, error)
    Delete(ctx context.Context, key string) error
}
```

#### 服务边界

**服务间通信规则**

```go
// 所有服务间通信通过接口依赖
type UserService interface {
    GetUser(ctx context.Context, id uint) (*UserDTO, error)
    UpdateUser(ctx context.Context, id uint, req *UpdateUserRequest) error
}

// 跨服务调用示例
type SessionService struct {
    sessionRepo   repository.SessionRepository
    deviceService DeviceService  // 接口依赖
    userService   UserService    // 接口依赖
}

func (s *SessionService) CreateSession(ctx context.Context, req *CreateSessionRequest) error {
    // 1. 验证设备存在
    device, err := s.deviceService.GetDevice(ctx, req.DeviceID)
    if err != nil {
        return err
    }
    
    // 2. 验证用户存在
    user, err := s.userService.GetUser(ctx, req.UserID)
    if err != nil {
        return err
    }
    
    // 3. 创建会话
    session := &models.Session{
        UserID:   req.UserID,
        DeviceID: req.DeviceID,
    }
    return s.sessionRepo.Insert(ctx, session)
}

// ✅ 允许: Service A → Service B (通过接口)
// ❌ 禁止: Service A → Repository B (跨越边界)
// ❌ 禁止: 循环依赖 (Service A → Service B → Service A)
```

**外部服务集成边界**

```go
// MinIO对象存储 (可选)
type Storage interface {
    Upload(ctx context.Context, key string, data io.Reader) error
    Download(ctx context.Context, key string) (io.ReadCloser, error)
}

// 实现可以切换:
// 1. LocalStorage: 本地文件系统
// 2. MinIOStorage: MinIO对象存储
// 配置决定使用哪个实现

// 依赖注入示例
var storage storage.Storage
if config.MinIOEndpoint != "" {
    storage = storage.NewMinIOStorage(config)
} else {
    storage = storage.NewLocalStorage(config.DataDir)
}
```

#### 数据边界

**数据库访问边界**

```go
// 规则: 只有Repository层可以访问*gorm.DB
// Service层通过Repository接口访问数据

type UserRepository interface {
    Insert(ctx context.Context, user *models.User) error
    FindByID(ctx context.Context, id uint) (*models.User, error)
    FindByUsername(ctx context.Context, username string) (*models.User, error)
    Update(ctx context.Context, user *models.User) error
    Delete(ctx context.Context, id uint) error
}

// Repository实现持有*gorm.DB
type userRepoImpl struct {
    db *gorm.DB
}

func (r *userRepoImpl) FindByID(ctx context.Context, id uint) (*models.User, error) {
    var user models.User
    err := r.db.WithContext(ctx).First(&user, id).Error
    return &user, err
}
```

**缓存访问边界**

```go
// BigCache: 用于业务数据缓存
// - 大小限制: 128MB
// - 键格式标准:
//   - "artifacts:{id}:header"
//   - "artifacts:{id}:body"
//   - "kv:{key}"
//   - "feed:{userId}:page:{pageNum}"

// TokenCache: 专用Token验证缓存
// - 实现: sync.Map
// - 内存占用: <10MB
// - 键格式: token字符串本身
// - TTL: 与token过期时间一致

// 使用示例
type ArtifactService struct {
    repo  repository.ArtifactRepository
    cache cache.Cache
}

func (s *ArtifactService) GetHeader(ctx context.Context, id uint) ([]byte, error) {
    // 1. 尝试从缓存获取
    cacheKey := fmt.Sprintf("artifacts:%d:header", id)
    if data, err := s.cache.Get(cacheKey); err == nil {
        return data, nil
    }
    
    // 2. 缓存未命中,从数据库获取
    artifact, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // 3. 写入缓存
    s.cache.Set(cacheKey, artifact.Header, 10*time.Minute)
    
    return artifact.Header, nil
}
```

**数据流边界**

```
┌─────────────────┐
│  客户端请求      │
└────────┬────────┘
         ↓
┌────────┴────────────────────────────┐
│  Gin Router (server/router.go)      │
└────────┬────────────────────────────┘
         ↓
┌────────┴────────────────────────────┐
│  中间件链                            │
│  - Recovery (panic恢复)              │
│  - Logging (请求日志)                │
│  - CORS (跨域处理)                   │
│  - RateLimit (速率限制)              │
│  - Auth (Token验证) ← 设置userID    │
└────────┬────────────────────────────┘
         ↓
┌────────┴────────────────────────────┐
│  Handler (handlers/*.go)             │
│  - 参数验证 (gin binding)            │
│  - 创建context with timeout          │
└────────┬────────────────────────────┘
         ↓
┌────────┴────────────────────────────┐
│  Service (service/*.go)              │
│  - 业务规则验证                       │
│  - 调用Repository                    │
│  - 可能调用其他Service                │
└────────┬────────────────────────────┘
         ↓
┌────────┴────────────────────────────┐
│  Repository (repository/*.go)        │
│  - 数据库查询/写入                    │
│  - 可选缓存检查                       │
└────────┬────────────────────────────┘
         ↓
    ┌───┴───┐
    │  DB   │  Cache
    │ SQLite│  BigCache
    └───────┘
         ↓
┌────────┴────────────────────────────┐
│  响应返回                            │
│  - Service → DTO转换                 │
│  - Handler → JSON序列化              │
│  - 中间件 → 日志记录                 │
└─────────────────────────────────────┘
```

---

### 功能模块映射表

| 功能需求 | Handler | Service | Repository | Models | 说明 |
|---------|---------|---------|------------|--------|------|
| **用户认证** | `auth_handler.go` | `encryption/token_service.go` | - | - | 无状态JWT认证 |
| **会话管理** | `session_handler.go` | `session_service.go` | `session_repo.go` | `session.go` | CRUD + 活跃度跟踪 |
| **设备管理** | `device_handler.go` | `device_service.go` | `device_repo.go` | `device.go` | 注册、状态、元数据 |
| **用户资料** | `user_handler.go` | `user_service.go` | `user_repo.go` | `user.go` | 资料、设置、头像 |
| **Artifacts** | `artifact_handler.go` | `artifact_service.go` | `artifact_repo.go` | `artifact.go` | header/body分离存储 |
| **KV存储** | `kv_handler.go` | `kv_service.go` | `kv_repo.go` | `kv.go` | CRUD + CAS操作 |
| **好友关系** | `friend_handler.go` | `friend_service.go` | `friend_repo.go` | `friend.go` | 添加、删除、列表 |
| **Feed流** | `feed_handler.go` | `feed_service.go` | `feed_repo.go` | `feed.go` | 时间线查询 |
| **推送通知** | `push_handler.go` | `push_service.go` | `push_repo.go` | `push_subscription.go` | 订阅管理 |
| **WebSocket** | `websocket/hub.go` | - | - | - | 实时事件分发 |

**跨模块共享组件映射:**

| 组件类型 | 位置 | 使用者 | 说明 |
|---------|------|--------|------|
| **Token验证** | `middleware/auth.go` | 所有需认证的handler | 中间件 |
| **Token服务** | `encryption/token_service.go` | auth_handler, middleware | 生成/验证JWT |
| **Token缓存** | `cache/token_cache.go` | encryption/token_service | 加速验证 |
| **日志记录** | 全局`*zap.Logger` | 所有层 | 结构化日志 |
| **错误定义** | `domain/errors.go` | 所有service | 领域错误 |
| **数据库连接** | `cmd/server/main.go` | 所有repository | *gorm.DB |

---

### 集成点定义

#### 内部通信模式

**Handler ↔ Service通信**

```go
// Handler通过依赖注入获取Service
type UserHandler struct {
    userService service.UserService  // 接口依赖
    logger      *zap.Logger
}

// 初始化 (cmd/server/main.go)
userService := service.NewUserService(userRepo)
userHandler := handlers.NewUserHandler(userService, logger)

// 调用示例
func (h *UserHandler) GetUser(c *gin.Context) {
    // 1. 创建超时context
    ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
    defer cancel()
    
    // 2. 从context获取userID (由中间件设置)
    userID := middleware.GetUserIDFromContext(ctx)
    
    // 3. 调用service
    user, err := h.userService.GetUser(ctx, userID)
    if err != nil {
        h.handleError(c, err)
        return
    }
    
    // 4. 返回响应
    c.JSON(200, user)
}
```

**Service ↔ Repository通信**

```go
// Service通过依赖注入获取Repository
type UserService struct {
    userRepo repository.UserRepository  // 接口依赖
}

// 调用示例
func (s *UserService) GetUser(ctx context.Context, id uint) (*UserDTO, error) {
    // 1. 调用repository
    user, err := s.userRepo.FindByID(ctx, id)
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrUserNotFound
        }
        return nil, fmt.Errorf("find user failed: %w", err)
    }
    
    // 2. 转换为DTO
    return &UserDTO{
        UserID:   user.ID,
        Username: user.Username,
        // ...
    }, nil
}
```

**Service ↔ Service通信**

```go
// 跨服务依赖注入
type SessionService struct {
    sessionRepo   repository.SessionRepository
    deviceService service.DeviceService  // 跨服务依赖
    userService   service.UserService    // 跨服务依赖
}

// 跨服务调用示例
func (s *SessionService) CreateSession(ctx context.Context, req *CreateSessionRequest) error {
    // 1. 验证设备存在 (跨服务调用)
    device, err := s.deviceService.GetDevice(ctx, req.DeviceID)
    if err != nil {
        return fmt.Errorf("device validation failed: %w", err)
    }
    
    // 2. 验证用户存在 (跨服务调用)
    user, err := s.userService.GetUser(ctx, req.UserID)
    if err != nil {
        return fmt.Errorf("user validation failed: %w", err)
    }
    
    // 3. 创建会话
    session := &models.Session{
        UserID:   user.ID,
        DeviceID: device.ID,
        Token:    generateSessionToken(),
    }
    
    return s.sessionRepo.Insert(ctx, session)
}
```

**WebSocket事件分发**

```go
// Service层发布事件
type SessionService struct {
    hub *websocket.Hub  // WebSocket Hub依赖
}

func (s *SessionService) CreateSession(ctx context.Context, req *CreateSessionRequest) error {
    // ... 创建会话逻辑 ...
    
    // 发布WebSocket事件到用户所有连接
    s.hub.BroadcastToUser(session.UserID, websocket.Message{
        Event: "session.created",
        Data: map[string]interface{}{
            "sessionId": session.ID,
            "deviceId":  session.DeviceID,
            "createdAt": session.CreatedAt,
        },
        Timestamp: time.Now(),
    })
    
    return nil
}

// Hub路由事件到连接
func (h *Hub) BroadcastToUser(userID uint, msg Message) {
    h.mu.RLock()
    connections := h.userConnections[userID]
    h.mu.RUnlock()
    
    for _, conn := range connections {
        select {
        case conn.Send <- msg:
        default:
            // 连接阻塞,跳过
        }
    }
}
```

#### 外部集成点

**数据库集成 (SQLite)**

```go
// 初始化: cmd/server/main.go
func setupDatabase(config *config.Config) (*gorm.DB, error) {
    db, err := gorm.Open(sqlite.Open(config.DatabaseURL), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Warn),
        PrepareStmt: true,
        DisableForeignKeyConstraintWhenMigrating: true,
    })
    if err != nil {
        return nil, err
    }
    
    // SQLite优化
    sqlDB, _ := db.DB()
    sqlDB.Exec("PRAGMA journal_mode = WAL")
    sqlDB.Exec("PRAGMA synchronous = NORMAL")
    sqlDB.Exec("PRAGMA cache_size = -64000")
    sqlDB.Exec("PRAGMA temp_store = MEMORY")
    sqlDB.Exec("PRAGMA mmap_size = 268435456")
    
    // 连接池配置
    sqlDB.SetMaxOpenConns(25)
    sqlDB.SetMaxIdleConns(5)
    sqlDB.SetConnMaxLifetime(5 * time.Minute)
    
    return db, nil
}
```

**缓存集成 (BigCache)**

```go
// 初始化: cmd/server/main.go
func setupCache() (cache.Cache, error) {
    bigCache, err := bigcache.NewBigCache(bigcache.Config{
        Shards:             1024,
        LifeWindow:         10 * time.Minute,
        MaxEntriesInWindow: 1000 * 10 * 60,
        MaxEntrySize:       500,
        HardMaxCacheSize:   128, // MB
        Verbose:            false,
    })
    if err != nil {
        return nil, err
    }
    
    return cache.NewBigCacheAdapter(bigCache), nil
}

// 使用: internal/service/artifact_service.go
func (s *ArtifactService) GetHeader(ctx context.Context, id uint) ([]byte, error) {
    cacheKey := fmt.Sprintf("artifacts:%d:header", id)
    
    // 尝试缓存
    if data, err := s.cache.Get(cacheKey); err == nil {
        return data, nil
    }
    
    // 查询数据库
    artifact, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // 写入缓存
    s.cache.Set(cacheKey, artifact.Header, 10*time.Minute)
    
    return artifact.Header, nil
}
```

**对象存储集成 (可选MinIO)**

```go
// 初始化: cmd/server/main.go
func setupStorage(config *config.Config) (storage.Storage, error) {
    if config.MinIOEndpoint != "" {
        // 使用MinIO
        return storage.NewMinIOStorage(storage.MinIOConfig{
            Endpoint:   config.MinIOEndpoint,
            AccessKey:  config.MinIOAccessKey,
            SecretKey:  config.MinIOSecretKey,
            BucketName: config.MinIOBucket,
        })
    }
    
    // 使用本地文件系统
    return storage.NewLocalStorage(config.DataDir + "/uploads"), nil
}

// 使用: internal/service/user_service.go
func (s *UserService) UploadAvatar(ctx context.Context, userID uint, avatar io.Reader) error {
    key := fmt.Sprintf("avatars/%d.jpg", userID)
    
    if err := s.storage.Upload(ctx, key, avatar); err != nil {
        return fmt.Errorf("upload avatar failed: %w", err)
    }
    
    // 更新用户记录
    return s.userRepo.UpdateAvatar(ctx, userID, key)
}
```

---

### 文件组织模式

#### 配置文件组织

```
项目根目录/
├── .env                     # 实际环境变量 (git忽略)
├── .env.example             # 环境变量示例 (提交到git)
├── configs/
│   ├── config.yaml          # 默认配置 (提交到git,无敏感信息)
│   └── config.example.yaml  # 配置示例 (提交到git)
└── internal/config/
    └── config.go            # 配置加载逻辑
```

**加载优先级**: 环境变量 > config.yaml > 默认值

```go
// internal/config/config.go
type Config struct {
    Port         int    `yaml:"port" envconfig:"PORT" default:"3005"`
    DatabaseURL  string `yaml:"database_url" envconfig:"DATABASE_URL" default:"./data/happy.db"`
    MasterSecret string `yaml:"-" envconfig:"HANDY_MASTER_SECRET" required:"true"`
    // ...
}

func Load() (*Config, error) {
    var cfg Config
    
    // 1. 加载YAML文件
    if data, err := os.ReadFile("configs/config.yaml"); err == nil {
        yaml.Unmarshal(data, &cfg)
    }
    
    // 2. 环境变量覆盖
    envconfig.Process("", &cfg)
    
    // 3. 验证
    if cfg.MasterSecret == "" {
        return nil, errors.New("HANDY_MASTER_SECRET is required")
    }
    
    return &cfg, nil
}
```

#### 源代码组织

**按层次分离**
```
internal/
├── handlers/    # API层,处理HTTP请求/响应
├── service/     # 业务逻辑层
├── repository/  # 数据访问层
├── models/      # 数据模型层
└── domain/      # 领域定义(错误、常量等)
```

**按功能分组文件**
```
每个功能域对应一组文件:
- user_handler.go, user_service.go, user_repo.go, models/user.go
- session_handler.go, session_service.go, session_repo.go, models/session.go
```

**接口集中定义**
```
internal/repository/interfaces.go  # 所有Repository接口
internal/service/interfaces.go     # 所有Service接口(可选)
```

#### 测试组织

**单元测试 (co-located)**
```
internal/service/
├── user_service.go
├── user_service_test.go      # 单元测试,同目录
├── session_service.go
└── session_service_test.go
```

**集成测试 (独立目录)**
```
tests/integration/
├── user_test.go              # 测试完整用户流程
├── session_test.go           # 测试会话管理流程
├── websocket_test.go         # 测试WebSocket连接
└── api_compatibility_test.go # 测试API兼容性
```

**测试工具与Fixtures**
```
tests/
├── fixtures/
│   └── test_data.go          # 测试数据生成器
└── mocks/
    ├── repositories.go       # Repository mock
    └── services.go           # Service mock (可选)
```

#### 资产组织

```
data/                         # 运行时数据目录
├── happy.db                  # SQLite数据库
└── uploads/                  # 本地上传文件(如不用MinIO)
    ├── avatars/
    └── attachments/

migrations/                   # 数据库迁移脚本
├── 000001_init_schema.up.sql
├── 000001_init_schema.down.sql
├── 000002_add_indexes.up.sql
└── 000002_add_indexes.down.sql
```

---

### 开发工作流集成

#### 开发服务器

```bash
# scripts/dev.sh
#!/bin/bash
set -e

# 1. 加载环境变量
if [ -f .env ]; then
    export $(cat .env | xargs)
fi

# 2. 运行数据库迁移
./scripts/migrate.sh up

# 3. 启动开发服务器 (支持热重载,如使用air)
if command -v air &> /dev/null; then
    air
else
    go run cmd/server/main.go
fi
```

**热重载配置 (可选air)**
```toml
# .air.toml
root = "."
tmp_dir = "tmp"

[build]
  cmd = "go build -o ./tmp/main cmd/server/main.go"
  bin = "tmp/main"
  include_ext = ["go", "yaml"]
  exclude_dir = ["data", "tmp", "vendor", "tests"]
```

#### 构建流程

```bash
# scripts/build.sh
#!/bin/bash
set -e

# 清理旧构建
rm -rf bin/
mkdir -p bin/

# 下载依赖
go mod download

# 构建二进制
CGO_ENABLED=1 go build \
    -ldflags="-s -w" \
    -o bin/happy-server \
    cmd/server/main.go

# 复制配置示例
cp configs/config.example.yaml bin/configs/

echo "Build complete: bin/happy-server"
```

**构建产物结构**
```
bin/
├── happy-server              # 可执行文件
└── configs/
    └── config.example.yaml   # 配置示例
```

#### 部署结构

**二进制部署**
```
/opt/happy-server/
├── happy-server              # 可执行文件
├── configs/
│   └── config.yaml           # 生产配置
├── data/
│   └── happy.db              # 数据库文件
├── logs/
│   └── app.log               # 应用日志
└── .env                      # 环境变量
```

**Docker部署**
```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

# 安装构建依赖 (libvips for bimg)
RUN apk add --no-cache gcc musl-dev vips-dev

WORKDIR /app
COPY . .

# 构建
RUN go mod download
RUN CGO_ENABLED=1 go build -o happy-server cmd/server/main.go

# 运行时镜像
FROM alpine:latest

# 安装运行时依赖
RUN apk add --no-cache ca-certificates vips

WORKDIR /app
COPY --from=builder /app/happy-server .
COPY --from=builder /app/configs/config.example.yaml configs/

# 创建数据目录
RUN mkdir -p data logs

EXPOSE 3005 9090

CMD ["./happy-server"]
```

**Docker Compose部署**
```yaml
# docker-compose.yml
version: '3.8'

services:
  happy-server:
    build: .
    ports:
      - "3005:3005"
      - "9090:9090"
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
      - ./configs/config.yaml:/app/configs/config.yaml
    environment:
      - PORT=3005
      - HANDY_MASTER_SECRET=${HANDY_MASTER_SECRET}
    restart: unless-stopped
```

---

### 数据流架构总结

**完整请求生命周期**

```
1. 客户端发起HTTP请求
   ↓
2. Gin接收请求 (server/router.go)
   ↓
3. 中间件链处理
   - Recovery: panic恢复
   - Logging: 记录请求开始
   - CORS: 跨域头设置
   - RateLimit: 速率检查
   - Auth: Token验证 → 设置userID到context
   ↓
4. 路由到Handler (handlers/*.go)
   - 解析路径参数
   - 验证请求体格式 (gin binding)
   - 创建5秒超时context
   ↓
5. Handler调用Service (service/*.go)
   - 从context获取userID
   - 业务规则验证
   - 可能调用其他Service
   - 调用Repository
   ↓
6. Service调用Repository (repository/*.go)
   - 检查缓存 (可选)
   - 执行SQL查询
   - GORM自动处理事务
   ↓
7. 数据持久层
   - SQLite WAL模式写入
   - BigCache缓存更新
   ↓
8. 响应返回路径
   - Repository返回models
   - Service转换为DTO
   - Handler序列化JSON
   - Logging中间件记录响应
   ↓
9. 客户端接收响应
```

**WebSocket事件流**

```
1. 客户端建立WebSocket连接
   ↓
2. Hub.register ← 新连接加入
   - 按userID分组存储
   - 按连接类型标记
   现有加密数据可被Go解密
  - Go加密数据可被客户端解密
  - 性能: 1MB数据加解密↓
3. 业务事件触发 (Service层)
   - session.created
   - device.updated
   - kv.changed
   ↓
4. Service调用Hub.BroadcastToUser
   - 查找用户的所有连接
   - 根据连接类型过滤
   - 避免发送者回显
   ↓
5. Hub路由到Connection
   - 写入conn.Send channel
   - 非阻塞发送
   ↓
6. Connection goroutine
   - 从Send channel读取
   - 序列化JSON
   - WebSocket.WriteJSON发送
   ↓
7. 客户端接收事件消息
```


---

## 架构验证结果

### 一致性验证 ✅

**决策兼容性:**
所有技术选择相互兼容且协同工作:
- Go 1.21+ 支持所有选定的库和框架
- Gin + GORM + zap 是成熟的技术组合,广泛应用于生产环境
- SQLite WAL模式与GORM完美集成
- coder/websocket是标准库兼容的现代WebSocket实现
- BigCache作为纯Go缓存无外部依赖
- bimg需要CGo和libvips,已在文档中明确标注

无版本冲突或不兼容问题。

**模式一致性:**
实现模式与架构决策完全对齐:
- 命名模式遵循Go标准 (PascalCase导出, camelCase私有) + 数据库标准 (snake_case)
- JSON API使用camelCase保持与原Node.js项目100%兼容
- 层次架构 (Handler→Service→Repository→Models) 清晰且单向依赖
- 错误处理采用Go惯用的分层模式,禁止panic
- Context传递模式符合Go并发最佳实践

无矛盾或冲突的模式定义。

**结构对齐:**
项目结构完美支持所有架构决策:
- 60+文件和目录按功能域和层次清晰组织
- 10个业务域映射到对应的handler/service/repository文件
- 横切关注点 (encryption, cache, storage, websocket) 独立模块化
- 测试结构 (co-located单元测试 + 独立集成测试) 支持TDD开发
- 配置、迁移、脚本、文档组织符合Go项目标准

结构与决策、模式完全一致。

---

### 需求覆盖验证 ✅

**功能需求覆盖:**
PRD中的所有10大功能域都有完整的架构支持:

| 功能域 | 架构组件 | 覆盖状态 |
|-------|---------|---------|
| 零知识加密 | `internal/encryption/` (TokenService + KeyTree) | ✅ 100% |
| REST API (50+端点) | 10个handler模块 + 完整路由定义 | ✅ 100% |
| WebSocket实时同步 | `websocket/hub.go` + 三种连接类型 | ✅ 100% |
| 会话与设备管理 | `session/` + `device/` 模块 | ✅ 100% |
| Artifacts存储 | `artifact/` + header/body分离 + 缓存 | ✅ 100% |
| KV存储 | `kv/` + CAS操作支持 | ✅ 100% |
| 用户与账户 | `user/` + `account/` + 头像上传 | ✅ 100% |
| 社交功能 | `friend/` + `feed/` 模块 | ✅ 100% |
| 推送通知 | `push/` + 订阅管理 | ✅ 100% |
| 对象存储 | `storage/` (本地/MinIO) + 图片处理 | ✅ 100% |

**非功能需求覆盖:**

| NFR | 目标 | 架构方案 | 验证 |
|-----|------|---------|------|
| 内存占用 | < 500MB | 内存预算: BigCache(128MB) + TokenCache(<10MB) + GORM(<20MB) + WebSocket(~200MB) + App(<100MB) ≈ 458MB | ✅ 满足 |
| API响应时间 | P95 < 200ms | Gin高性能框架 + SQLite WAL + BigCache缓存 + 无外部网络调用 | ✅ 可达成 |
| API兼容性 | 100% | JSON字段camelCase + 路由完全映射 + 错误格式一致 + HTTP状态码标准 | ✅ 保证 |
| 零外部依赖 | 必须 | SQLite嵌入式 + BigCache内存 + MinIO可选 | ✅ 满足 |
| 加密兼容 | 100% | privacy-kit三阶段实现与验证策略 | ✅ 有保障 |
| 可靠性 | 7x24 | Graceful shutdown(30s) + 健康检查 + SQLite WAL持久化 | ✅ 支持 |

**跨切关注点覆盖:**
- **认证授权**: JWT Token生成验证 + 中间件拦截 + Context传递userID
- **日志记录**: zap结构化JSON日志 + 敏感信息脱敏规则 + 级别标准
- **错误处理**: 分层错误处理 (Repository→Service→Handler) + 领域错误 + HTTP映射
- **配置管理**: envconfig + YAML + 明确的优先级 (环境变量 > YAML > 默认值)
- **监控指标**: Prometheus metrics + 独立端口9090 + 健康检查端点

所有需求都有明确且可实现的架构支持。

---

### 实现准备度验证 ✅

**决策完整性:**
- ✅ 所有关键技术决策都有详细文档和代码示例
- ✅ 技术栈版本明确 (Go 1.21+, Gin v1.10+, GORM v1.25+等)
- ✅ 数据库优化配置 (PRAGMA设置, 连接池参数) 具体明确
- ✅ WebSocket Hub模式有完整的实现示例
- ✅ 加密服务接口和实现策略清晰定义
- ✅ 缓存策略和键格式标准明确

AI代理可以直接参考文档进行实现,无需猜测或假设。

**结构完整性:**
- ✅ 完整项目目录树 (60+文件) 详细定义
- ✅ 每个文件的职责和内容明确
- ✅ 所有集成点有代码示例 (Handler↔Service, Service↔Repository, Service↔Service)
- ✅ 依赖注入模式清晰展示
- ✅ 边界定义完整 (API边界、组件边界、服务边界、数据边界)
- ✅ 数据流架构图完整描述请求生命周期

AI代理知道在哪里创建什么文件,如何组织代码。

**模式完整性:**
- ✅ 识别52个潜在冲突点,全部有明确规则
- ✅ 命名模式 (15条规则) 覆盖数据库、API、Go代码所有层次
- ✅ 结构模式 (12条规则) 定义测试位置、文件组织、配置管理
- ✅ 格式模式 (10条规则) 规范JSON、HTTP、时间格式
- ✅ 通信模式 (8条规则) 统一WebSocket事件、日志格式
- ✅ 流程模式 (7条规则) 规范错误处理、Context、事务、验证
- ✅ 每个模式都有良好示例和反模式警示

AI代理可以保持跨整个项目的一致性实现。

---

### Gap分析结果

**Critical Gaps (阻塞实现):** 
- 无

**Important Gaps (建议补充,不阻塞初始开发):**
1. **数据库Schema迁移文件**: 当前有models定义,建议添加完整的SQL DDL示例 (18个表)
2. **privacy-kit详细测试计划**: 三阶段策略已定义,可在Phase 1时补充具体测试用例

**Nice-to-Have Gaps (可选优化):**
1. API文档生成策略 (OpenAPI/Swagger)
2. 性能基准测试计划
3. 灾难恢复与备份策略

**评估结论**: 无阻塞性缺陷,可以立即进入实现阶段。Important Gaps可在实现过程中逐步补充。

---

### 验证问题处理

**初步识别问题:**
Privacy-kit兼容性验证细节可进一步完善

**用户反馈:**
用户确认当前高级策略已足够,详细测试矩阵暂不需要

**最终决策:**
采用现有三阶段策略,在Phase 1独立验证项目中根据实际情况补充测试用例

---

### 架构完整性检查清单

**✅ 需求分析**
- [x] 项目上下文深度分析 (8,700行代码, 18个模型, 50+端点)
- [x] 规模与复杂度评估 (零知识加密, 100% API兼容)
- [x] 技术约束识别 (<500MB内存, privacy-kit兼容)
- [x] 跨切关注点映射 (认证、日志、错误、配置、监控)

**✅ 架构决策**
- [x] 关键决策文档化并包含版本 (GORM, Gin, SQLite WAL, WebSocket)
- [x] 技术栈完全指定 (Go 1.21+, 所有依赖库版本范围)
- [x] 集成模式定义 (依赖注入, 接口抽象, 分层架构)
- [x] 性能考虑 (连接池, 缓存策略, SQLite优化, 内存预算)

**✅ 实现模式**
- [x] 命名规范建立 (数据库snake_case, Go PascalCase, JSON camelCase)
- [x] 结构模式定义 (co-located测试, 层次分离, 接口集中)
- [x] 通信模式规范 (WebSocket事件格式, 日志级别, 敏感信息脱敏)
- [x] 流程模式文档化 (错误处理, Context传递, 事务管理, 资源清理)

**✅ 项目结构**
- [x] 完整目录结构定义 (cmd, internal, pkg, configs, migrations, tests等60+文件)
- [x] 组件边界建立 (Handler→Service→Repository→Models单向依赖)
- [x] 集成点映射 (Handler↔Service, Service↔Repository, Service↔Service)
- [x] 需求到结构映射完整 (10个功能域 → handler/service/repository文件)

---

### 架构准备度评估

**整体状态:** ✅ **准备就绪,可进入实现阶段**

**信心等级:** **高** - 架构文档全面、具体、可操作

**关键优势:**
1. **详尽的决策文档**: 所有关键技术决策都有代码示例和配置细节
2. **完整的模式规范**: 52个潜在冲突点全部有明确规则,防止AI代理实现差异
3. **清晰的项目结构**: 60+文件详细定义,每个文件职责明确
4. **强一致性保障**: 命名、格式、通信、流程模式全面规范
5. **100% API兼容设计**: JSON字段、路由、错误格式与原Node.js项目完全对齐
6. **实战代码示例**: Handler/Service/Repository每层都有完整实现示例

**未来增强领域:**
1. 在实际开发中补充完整的数据库迁移SQL (基于GORM models生成)
2. Phase 1时根据privacy-kit实际行为完善测试用例
3. 可选: 添加OpenAPI规范自动生成工具
4. 可选: 建立性能基准测试套件

---

### 实现交接

**AI代理实施指南:**

1. **严格遵循架构决策**: 
   - 所有技术选择 (Gin, GORM, SQLite, coder/websocket) 不可更改
   - 配置参数 (连接池大小, PRAGMA设置, 缓存大小) 精确使用文档值

2. **一致应用实现模式**:
   - 每个命名决策都参考命名模式章节
   - 所有错误处理遵循分层模式 (Repository→Service→Handler)
   - 每个handler/service/repository方法第一参数必须是context.Context

3. **尊重项目结构与边界**:
   - 按照定义的目录树创建文件
   - Handler层不得跨过Service直接调用Repository
   - Service层不得依赖gin.Context,只使用context.Context

4. **参考架构文档解决所有问题**:
   - 遇到命名疑问 → 查看命名模式章节
   - 遇到集成疑问 → 查看集成点定义章节
   - 遇到错误处理疑问 → 查看流程模式章节

**首要实现优先级:**

根据决策影响分析,建议实现顺序:

**P0 (阻塞所有):**
1. 项目初始化 (go.mod, 目录结构)
2. 配置管理 (`internal/config/`)
3. 数据库连接与Models (`internal/models/`, SQLite + GORM)
4. 核心加密服务 (`internal/encryption/` TokenService - Phase 1验证后)

**P1 (阻塞业务逻辑):**
5. Repository层实现 (所有数据访问)
6. 中间件 (`middleware/auth.go` 等)
7. 基础Service实现 (user, session, device)

**P2 (核心业务功能):**
8. 所有Handler实现
9. WebSocket Hub (`websocket/`)
10. 缓存集成 (`cache/`)

**P3 (增强功能):**
11. 存储服务 (`storage/`)
12. 图片处理 (bimg)
13. 推送通知


---

## 架构完成总结

### 工作流完成

**架构决策工作流:** 已完成 ✅  
**完成步骤总数:** 8  
**完成日期:** 2025-12-10  
**文档位置:** `docs/architecture.md`

---

### 最终架构交付物

**📋 完整架构文档**

- 所有架构决策记录,包含具体版本号
- 实现模式规范,确保AI代理一致性
- 完整项目结构,包含所有文件和目录
- 需求到架构的映射关系
- 全面验证确认一致性与完整性

**🏗️ 实现就绪基础**

- **架构决策**: 4大类(数据架构、加密安全、WebSocket实现、基础设施部署)
- **实现模式**: 52条规则(命名15条、结构12条、格式10条、通信8条、流程7条)
- **架构组件**: 10个业务域 + 5个横切关注点(encryption、cache、storage、websocket、middleware)
- **需求支持**: 100%功能需求覆盖 + 6项NFR全部满足

**📚 AI代理实施指南**

- 技术栈及验证版本 (Go 1.21+, Gin v1.10+, GORM v1.25+等)
- 一致性规则防止实现冲突
- 清晰的项目结构与边界定义
- 集成模式与通信标准

---

### 实现交接

**致AI代理:**

本架构文档是实现 **happy-server-go** 的完整指南。严格遵循所有决策、模式和结构。

**首要实现优先级:**

**P0 (阻塞所有):**
1. 项目初始化 (`go mod init`, 创建目录结构)
2. 配置管理 (`internal/config/config.go`)
3. 数据库连接与Models (`internal/models/`, GORM + SQLite)
4. 核心加密服务 (`internal/encryption/` - 需先完成Phase 1兼容性验证)

**P1 (阻塞业务逻辑):**
5. Repository层实现 (所有数据访问接口)
6. 中间件 (`middleware/auth.go`等)
7. 基础Service (`user_service.go`, `session_service.go`, `device_service.go`)

**P2 (核心业务功能):**
8. 所有Handler实现 (10个handler文件)
9. WebSocket Hub (`websocket/hub.go`)
10. 缓存集成 (`cache/bigcache.go`, `cache/token_cache.go`)

**P3 (增强功能):**
11. 存储服务 (`storage/local.go`, 可选`storage/minio.go`)
12. 图片处理 (`storage/image.go` with bimg)
13. 推送通知 (`push_service.go`)

**开发流程:**

1. 初始化项目使用标准Go项目布局
2. 按照P0→P1→P2→P3顺序实现
3. 每个组件实现后编写单元测试(co-located)
4. 关键流程编写集成测试(`tests/integration/`)
5. 遵循架构文档中的所有命名、格式、流程模式

---

### 质量保证检查清单

**✅ 架构一致性**

- [x] 所有决策协同工作无冲突
- [x] 技术选择相互兼容
- [x] 模式支持架构决策
- [x] 结构与所有决策对齐

**✅ 需求覆盖**

- [x] 所有功能需求得到支持
- [x] 所有非功能需求得到解决
- [x] 横切关注点得到处理
- [x] 集成点已定义

**✅ 实现准备度**

- [x] 决策具体且可操作
- [x] 模式防止代理冲突
- [x] 结构完整且明确
- [x] 提供示例以便理解

---

### 项目成功要素

**🎯 清晰的决策框架**  
每个技术选择都经过协作式讨论并有明确理由,确保所有利益相关者理解架构方向。

**🔧 一致性保证**  
实现模式和规则确保多个AI代理生成的代码兼容且一致,无缝协同工作。

**📋 全面覆盖**  
所有项目需求都有架构支持,从业务需求到技术实现有清晰的映射关系。

**🏗️ 坚实基础**  
选定的架构模式和技术栈提供了遵循当前最佳实践的生产就绪基础。

---

**架构状态:** ✅ **准备就绪,可进入实现阶段**

**下一阶段:** 使用本文档中的架构决策和模式开始实现。

**文档维护:** 在实现过程中做出重大技术决策时更新本架构文档。

