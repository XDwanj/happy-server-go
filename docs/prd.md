---
stepsCompleted: [1, 2, 3, 4, 7]
inputDocuments:
  - 'docs/analysis/product-brief-happy-server-go-2025-12-09.md'
  - 'docs/analysis/research/technical-happy-server-go-rewrite-research-2025-12-09.md'
documentCounts:
  briefs: 1
  research: 1
  brainstorming: 0
  projectDocs: 0
workflowType: 'prd'
lastStep: 7
project_name: 'happy-server-go'
user_name: 'XDwanj'
date: '2025-12-09'
---

# Product Requirements Document - happy-server-go

**Author:** XDwanj
**Date:** 2025-12-09

## Executive Summary

Happy Server Go 是 Happy Server 的轻量级 Go 重写版本，专为资源受限环境下的自部署场景设计。通过 Go 单体架构重写，它将原本需要多个外部依赖（PostgreSQL、Redis、MinIO）的部署简化为单个二进制文件或单容器部署，总内存占用从 300MB+ 外部依赖降低至 500MB 以内。

该项目旨在解决当前 Happy Server (Node.js) 自部署门槛过高的问题：复杂的依赖管理、较高的资源占用（对 1-2GB VPS 负担过重）、以及多组件协调配置的部署复杂度。这些障碍导致许多潜在的自部署用户望而却步，限制了 Happy Server 在需要自部署的细分市场（出于隐私、合规或特殊网络环境）的采用。

Happy Server Go 保持与原版 100% 的 API 兼容性和功能一致性，确保现有客户端无需任何修改即可切换。作为并行维护的自部署友好版本，它为那些希望自建基础设施但资源有限的用户提供了理想选择，同时保持了 Happy Server 核心的零知识架构和隐私优先原则。

**目标用户：** 小团队技术负责人和全栈开发者（2-10 人规模），他们运行预算有限的 VPS 环境（1-2GB RAM），需要自建基础设施（出于隐私、合规或成本考虑），具备基本的 Docker 和命令行操作能力。次要用户是使用 Claude Code 的团队成员，他们是最终受益者——简化的部署意味着团队能够更快地开始使用多设备同步功能。

### What Makes This Special

Happy Server Go 的核心差异化优势体现在五个关键方面：

1. **零依赖部署** - 无需配置 PostgreSQL、Redis、MinIO 三个独立服务，从"需要运维多个服务"到"下载即运行"，降低部署失败率和运维复杂度。

2. **资源友好** - 总内存占用 < 500MB（相比 Node.js 版本的 300MB + 外部依赖），适配 1GB VPS 环境，降低自部署成本。

3. **完全兼容** - API 接口 100% 一致，功能特性完全对齐，现有客户端零改动即可切换使用，确保平滑迁移。

4. **快速部署** - 单二进制文件部署（下载 → 运行）或 Docker 单容器部署（一条命令完成），从"需要 30 分钟配置多个服务"到"5 分钟完成部署"。

5. **保持开源透明** - 继承 Happy Server 的开源精神，MIT 协议，可审计、可修改，社区驱动的自部署友好版本。

这些差异化因素共同创造了一个独特的价值主张：在保持完整功能和零知识安全架构的前提下，让小团队和个人开发者能够在资源受限的环境中轻松部署和维护端到端加密同步服务。

## Project Classification

**Technical Type:** API Backend
**Domain:** General
**Complexity:** Low
**Project Context:** Greenfield - new project

Happy Server Go 是一个 API 后端服务项目，提供 RESTful API 端点和 WebSocket 实时通信功能，支持端到端加密的多设备同步。作为通用软件项目，它专注于标准的安全性和性能要求，无特殊行业监管约束。

项目采用 Go 单体架构，内置所有依赖（BigCache + 事件总线替代 Redis，SQLite 替代 PostgreSQL，本地文件存储替代 MinIO），实现零外部依赖的部署模式。核心技术栈包括：
- **Web 框架：** Gin（高性能 HTTP 框架）
- **数据库：** SQLite + GORM ORM + golang-migrate
- **缓存：** BigCache（GC 优化的内存缓存）
- **WebSocket：** coder/websocket（现代化 WebSocket 库）
- **存储：** 本地文件系统
- **日志：** zap（高性能结构化日志）

作为绿地项目，Happy Server Go 从零开始构建，但必须保持与现有 Happy Server (Node.js) 的 API 完全兼容，确保现有客户端（Claude Code）无需修改即可切换使用。

## Success Criteria

### User Success

Happy Server Go 的用户成功体现在部署体验和运行稳定性两个关键维度：

**部署体验成功：**
- **部署时间**：从下载到运行成功 < 5 分钟（无论是二进制文件还是 Docker 部署）
- **部署成功率**：首次部署成功率 > 90%，用户无需额外配置或故障排除即可完成部署
- **零配置启动**：开箱即用，无需手动创建配置文件或初始化数据库
- **清晰的错误提示**：部署失败时提供明确的错误信息和解决建议

**运行稳定性成功：**
- **资源占用**：总内存占用 < 500MB（包括所有内置服务），在 1-2GB RAM VPS 上稳定运行
- **功能完整性**：100% API 兼容，现有 Claude Code 客户端零改动即可切换使用
- **多设备同步**：支持与 Node.js 版本相同的并发设备数量，实时同步无延迟
- **长期稳定性**：7×24 小时持续运行无内存泄漏，无需定期重启

**用户"值得"的时刻：**
用户成功的标志是当他们完成部署后，第一次看到客户端成功连接并同步数据，同时发现服务器内存占用确实低于 500MB，意识到"终于可以在我的小 VPS 上运行了"。

### Business Success

Happy Server Go 的业务成功聚焦于成为自部署场景的首选方案和建立健康的开源社区。

**3 个月成功指标（验证方向）：**
- **社区认可**：GitHub Stars > 100，表明项目获得初步关注
- **实际部署**：至少 50 个活跃部署实例（通过可选的匿名统计或社区反馈推算）
- **首批反馈**：收集到 10+ 条真实用户反馈（GitHub Issues/Discussions），用于验证产品方向
- **文档完善**：部署文档完整，90% 的常见问题可通过文档自助解决

**12 个月成功指标（市场定位）：**
- **市场定位确立**：被社区明确定位为"Happy Server 的自部署友好版本"，在相关讨论中被推荐
- **用户增长**：GitHub Stars > 500，活跃部署实例 > 200
- **社区健康**：至少 5 位外部贡献者，Issue 平均响应时间 < 48 小时
- **用户留存**：持续运行超过 3 个月的实例占比 > 70%（表明稳定可靠）

**核心业务指标：**
- **采用率**：每月新增部署实例数量持续增长
- **社区活跃度**：GitHub Issues、Discussions、PR 的数量和质量
- **用户自助成功率**：新用户通过文档成功部署的比例 > 85%
- **问题解决效率**：Critical bugs 平均修复时间 < 7 天

### Technical Success

Happy Server Go 的技术成功建立在完全兼容、性能达标和架构简洁三个支柱之上。

**功能完整性标准：**
- **API 完全兼容**：所有 Happy Server Node.js 版本的 RESTful API 端点在 Go 版本中完全实现且行为一致
- **WebSocket 实时同步**：实时双向通信功能完整，支持多设备并发连接
- **零知识架构**：端到端加密存储和传输机制完整实现，服务器无法解密用户数据
- **客户端兼容性**：现有 Claude Code 客户端无需任何代码修改即可切换使用 Go 版本
- **功能对等性**：加密推送通知、多设备会话管理、加密签名认证等所有原版功能完整实现

**性能基线要求：**
- **内存占用**：峰值内存 < 500MB（在 10 个并发设备连接的场景下测试）
- **运行环境**：在 1GB RAM VPS 上稳定运行，系统可用内存保持 > 200MB
- **并发能力**：支持至少 50 个并发 WebSocket 连接（与 Node.js 版本对等）
- **响应时间**：API 端点 95 百分位响应时间 < 200ms（正常负载）
- **启动时间**：服务启动到就绪 < 10 秒

**架构与部署标准：**
- **零外部依赖**：无需安装或配置 PostgreSQL、Redis、MinIO
- **单二进制部署**：编译后生成单个可执行文件，支持 Linux/macOS/Windows
- **容器化支持**：提供官方 Docker 镜像，镜像大小 < 100MB
- **数据持久化**：SQLite 数据库支持热备份，数据迁移机制清晰
- **配置管理**：支持环境变量和配置文件两种配置方式

**质量与可维护性：**
- **代码质量**：关键业务逻辑单元测试覆盖率 > 80%
- **可观测性**：结构化日志（zap），支持不同日志级别，便于问题排查
- **错误处理**：所有错误场景有明确的错误码和错误信息
- **文档完整性**：API 文档、部署文档、故障排查文档完整

**安全基线：**
- **零知识架构**：服务器无法访问用户明文数据
- **加密传输**：所有通信使用 TLS 1.3+
- **依赖安全**：定期更新依赖，及时修复已知安全漏洞
- **安全配置**：默认配置遵循安全最佳实践

### Measurable Outcomes

**MVP 验证指标（发布后 1 个月内）：**
- ✅ 至少 10 个独立用户成功完成部署（通过 GitHub Issues/Discussions 验证）
- ✅ 首次部署成功率 > 90%（通过用户反馈统计）
- ✅ 至少 1 个用户报告从 Node.js 版本切换成功，客户端无需修改
- ✅ 无 Critical 级别的 bug 未修复超过 7 天
- ✅ 内存占用实测数据 < 500MB（在典型使用场景下）

**技术里程碑：**
- ✅ 所有原版 API 端点通过集成测试（100% 覆盖）
- ✅ 性能基准测试通过：内存 < 500MB，响应时间 < 200ms（P95）
- ✅ 在 1GB VPS 上连续运行 7 天无崩溃
- ✅ 支持 Linux/macOS/Windows 三个平台的二进制构建
- ✅ Docker 镜像可在主流平台（amd64/arm64）运行

**社区健康指标：**
- ✅ 文档完整度：部署指南、API 参考、故障排查至少覆盖 90% 常见场景
- ✅ Issue 响应时间中位数 < 48 小时
- ✅ 至少 20 个 GitHub Stars（表明社区初步关注）

## Product Scope

### MVP - Minimum Viable Product

MVP 的核心目标是**功能完整且部署简单**，实现与 Node.js 版本 100% 功能对等，同时验证零依赖部署的可行性。

**核心功能（必须实现）：**

1. **零知识加密存储**
   - 服务器存储加密数据但无法解密
   - 支持用户端加密/解密
   - 加密密钥由客户端管理

2. **RESTful API 完整实现**
   - 所有原版 API 端点（用户认证、数据同步、会话管理等）
   - API 行为与 Node.js 版本完全一致
   - 错误响应格式保持兼容

3. **WebSocket 实时同步**
   - 跨设备实时数据同步
   - 支持多设备并发连接
   - 连接断开重连机制

4. **多设备会话管理**
   - 设备注册和认证
   - 设备列表管理
   - 设备间数据同步

5. **加密推送通知**
   - 任务完成通知
   - 权限请求通知
   - 通知内容加密

6. **加密签名认证**
   - 基于公钥签名的认证系统
   - 安全的会话管理

**技术架构（必须完成）：**

1. **内置依赖替代**
   - BigCache + 自实现事件总线替代 Redis
   - SQLite + GORM 替代 PostgreSQL
   - 本地文件存储替代 MinIO
   - 所有替代方案功能对等

2. **部署支持**
   - 单二进制文件编译和分发
   - Docker 镜像构建和发布
   - 基本的环境变量配置
   - 数据目录配置

3. **基础可观测性**
   - 结构化日志（zap）
   - 基本的健康检查端点
   - 启动/关闭日志清晰

**文档与测试（必须完成）：**
- 快速开始指南（5 分钟部署）
- Docker 部署文档
- 基本的 API 文档
- 单元测试覆盖核心业务逻辑
- 集成测试验证 API 兼容性

**MVP 明确不包含：**
- ❌ 数据迁移工具（从 Node.js 版本迁移）
- ❌ Web 管理界面
- ❌ 集群部署支持（多实例负载均衡）
- ❌ 高级监控和指标采集
- ❌ 性能调优工具

**MVP 成功标准：**
- 现有客户端零改动即可连接使用
- 在 1GB RAM VPS 上稳定运行
- 部署时间 < 5 分钟
- 所有核心功能通过测试

### Growth Features (Post-MVP)

MVP 验证成功后，这些功能将提升用户体验和运维便利性。

**Phase 1：运维增强（MVP 后 1-3 个月）**
- 数据迁移工具（从 Node.js 版本迁移数据）
- 数据库备份/恢复命令行工具
- 配置文件热重载
- 更详细的性能指标输出（内存、CPU、连接数）
- 日志轮转和归档配置

**Phase 2：可观测性增强（3-6 个月）**
- Prometheus 指标暴露
- 更丰富的健康检查端点（数据库、缓存、存储）
- 性能分析工具集成（pprof）
- 告警机制（关键错误通知）

**Phase 3：管理便利性（6-12 个月）**
- Web 管理界面（可选）
  - 基本系统状态展示
  - 连接设备列表
  - 简单的配置管理
- CLI 管理工具
  - 用户管理命令
  - 数据库维护命令
  - 诊断工具

**Phase 4：高级部署（12+ 个月）**
- 集群部署支持（多实例 + 负载均衡）
- 分布式缓存（多实例间缓存同步）
- 数据库读写分离（SQLite 限制下的优化）

### Vision (Future)

长期愿景是让 Happy Server Go 成为 Happy Server 生态的标准自部署选项，并扩展到更多部署场景。

**生态定位愿景：**
- 成为 Claude Code 官方推荐的自部署方案
- 在社区中被认可为"资源受限环境下的最佳选择"
- 与 Node.js 版本形成互补，而非竞争关系

**技术演进愿景：**
- 支持更多嵌入式和边缘设备部署场景（如 Raspberry Pi）
- 探索更轻量的部署选项（如 WebAssembly）
- 社区驱动的插件系统（可选功能模块）

**社区驱动愿景：**
- 活跃的贡献者社区
- 丰富的第三方工具生态（监控、部署脚本等）
- 多语言文档和本地化支持

**潜在创新方向：**
- 自动化的性能调优建议
- 智能的资源使用监控和预警
- 更简单的多实例部署工具

## User Journeys

### Journey 1: 李明 - 技术负责人的突破时刻

李明是一家 8 人远程团队的技术负责人，团队刚开始使用 Claude Code。公司出于数据隐私考虑，要求所有工具必须自部署，但李明只有一台 2GB RAM 的 VPS。周末他尝试部署 Happy Server (Node.js 版本)，配置 PostgreSQL、Redis、MinIO 后，服务器几乎崩溃。他沮丧地关闭了配置文档，不知道该如何向团队交代。

周一早上，李明在 GitHub 上偶然发现 Happy Server Go，文档承诺"5 分钟部署、内存 < 500MB"。抱着最后试试的心态，他下载了二进制文件，运行 `./happy-server-go`。终端显示 "Server started on port 8080"，他惊讶地发现一切已经就绪——无需配置数据库、无需启动缓存服务、无需设置对象存储。

他快速配置好客户端，Claude Code 立即连接成功，数据开始同步。李明打开 `htop`，内存占用显示 420MB。他简直不敢相信——整个服务运行在半 GB 内存以内。

两周后，团队的 8 名开发者都在使用这套系统，多设备同步流畅无比。当 CEO 询问部署进度时，李明自豪地说："早就完成了，而且非常稳定。"这一刻，他意识到技术选择的简洁性有时比复杂性更有价值。

### Journey Requirements Summary

李明的旅程揭示了 Happy Server Go 需要的核心能力：

**部署体验能力：**
- **零配置启动**：单二进制文件运行，无需手动配置外部依赖
- **快速验证**：从下载到服务就绪 < 5 分钟，启动日志清晰明确
- **开箱即用**：默认配置即可支持生产使用，无需调优

**运行时能力：**
- **资源可见性**：用户能够通过系统工具（如 htop）验证内存占用
- **多用户并发**：支持团队（8+ 人）同时使用，设备数量无限制
- **长期稳定性**：持续运行数周无需重启或人工干预

**集成能力：**
- **客户端兼容性**：Claude Code 客户端零配置即可连接
- **实时同步**：多设备间数据同步流畅，延迟低
- **会话管理**：支持多设备注册和管理

**信心建立：**
- **性能承诺兑现**：实际内存占用与文档承诺一致（< 500MB）
- **简单可靠**：技术负责人能够自信地向管理层和团队报告部署状态

## API Backend Specific Requirements

### Project-Type Overview

Happy Server Go 是一个 API 后端服务，提供完整的 RESTful API 和 WebSocket 实时通信功能，支持端到端加密的多设备同步。作为 Happy Server (Node.js) 的 Go 重写版本，它必须保持 100% API 兼容性，确保现有 Claude Code 客户端无需修改即可切换使用。

### Technical Architecture Considerations

**API 架构设计：**
- **单体架构**：所有 API 端点和 WebSocket 服务运行在单个 Go 进程中
- **RESTful API**：遵循 REST 设计原则，使用标准 HTTP 方法
- **WebSocket 实时通信**：独立的 WebSocket 服务用于实时数据推送和双向通信
- **版本控制策略**：保持与 Node.js 版本的 API 兼容性，不引入新的版本控制机制

### API 端点规范

Happy Server Go 必须实现以下 API 端点群组（基于 Node.js 版本分析）：

**1. 认证端点（Authentication）：**
- `POST /v1/auth` - 公钥签名认证
  - 请求：publicKey (string), challenge (string), signature (string)
  - 响应：success (boolean), token (string)
  - 使用 tweetnacl 验证签名

- `POST /v1/auth/request` - 终端认证请求
  - 请求：publicKey (string), supportsV2 (boolean)
  - 响应：state ('requested' | 'authorized'), token (optional)

- `GET /v1/auth/request/status` - 认证请求状态查询
  - 查询参数：publicKey (string)
  - 响应：status ('not_found' | 'pending' | 'authorized'), supportsV2 (boolean)

**2. 会话管理端点（Sessions）：**
- `GET /v1/sessions` - 获取用户所有会话列表
  - 认证：Bearer Token
  - 响应：sessions 数组，包含会话元数据、状态、加密密钥等

- `GET /v2/sessions/active` - 获取活跃会话列表（V2）
  - 认证：Bearer Token
  - 查询参数：limit (number, 1-500, 默认 150)
  - 响应：仅返回 15 分钟内活跃的会话

- `POST /v1/sessions` - 创建新会话
- `PUT /v1/sessions/:id` - 更新会话元数据
- `DELETE /v1/sessions/:id` - 删除会话

**3. 设备管理端点（Machines）：**
- `GET /v1/machines` - 获取用户设备列表
- `POST /v1/machines` - 注册新设备
- `PUT /v1/machines/:id` - 更新设备信息
- `DELETE /v1/machines/:id` - 删除设备

**4. 数据同步端点（KV Store & Artifacts）：**
- `GET /v1/kv/:key` - 获取键值对数据
- `PUT /v1/kv/:key` - 设置键值对数据
- `DELETE /v1/kv/:key` - 删除键值对数据
- `GET /v1/artifacts/:id` - 获取工件数据
- `POST /v1/artifacts` - 创建工件
- `PUT /v1/artifacts/:id` - 更新工件

**5. 推送通知端点（Push Notifications）：**
- `POST /v1/push/register` - 注册推送令牌
- `DELETE /v1/push/unregister` - 注销推送令牌

**6. 用户管理端点（Account & User）：**
- `GET /v1/account` - 获取当前用户信息
- `PUT /v1/account` - 更新用户信息
- `GET /v1/user/:username` - 获取用户公开信息

**7. 其他端点：**
- `GET /v1/version` - 获取服务器版本信息
- `GET /v1/feed` - 获取用户动态流
- `GET /v1/connect` - 连接状态检查
- `POST /v1/voice` - 语音相关功能
- `GET /v1/accessKeys` - 访问密钥管理
- `/v1/dev/*` - 开发者端点（可选）

### WebSocket 实时通信

**WebSocket 服务规范：**
- **路径**：`/v1/updates`
- **传输协议**：WebSocket 和 Polling（降级支持）
- **认证机制**：握手时通过 `auth.token` 进行认证

**客户端类型支持：**
1. **session-scoped** - 会话级客户端（需要 sessionId）
2. **user-scoped** - 用户级客户端（默认）
3. **machine-scoped** - 设备级客户端（需要 machineId）

**WebSocket 事件处理器：**
- `ping` - 心跳检测
- `sessionUpdate` - 会话更新通知
- `machineUpdate` - 设备更新通知
- `artifactUpdate` - 工件更新通知
- `accessKeyUpdate` - 访问密钥更新通知
- `rpc` - 远程过程调用
- `usage` - 使用情况上报

**连接参数：**
- `pingTimeout`: 45秒
- `pingInterval`: 15秒
- `upgradeTimeout`: 10秒
- `connectTimeout`: 20秒

### 认证与授权模型

**认证机制：**
- **公钥签名认证（Primary）**：
  - 使用 Ed25519 签名算法（tweetnacl）
  - 客户端使用私钥签名挑战（challenge）
  - 服务器使用公钥验证签名
  - 验证成功后颁发 JWT token

- **Bearer Token 认证（Secondary）**：
  - 所有需要认证的 API 使用 Bearer Token
  - Token 在请求头 `Authorization: Bearer <token>` 中传递
  - WebSocket 握手时在 `auth.token` 中传递

**授权模型：**
- **用户级授权**：所有 API 端点基于用户身份进行授权
- **会话级授权**：会话相关操作验证会话所有权
- **设备级授权**：设备相关操作验证设备所有权
- **无速率限制**：根据用户确认，不实现速率限制策略

### 数据格式与序列化

**请求/响应格式：**
- **Content-Type**: `application/json`
- **字符编码**: UTF-8
- **日期时间格式**: Unix 时间戳（毫秒）或 ISO 8601

**数据验证：**
- 使用 Zod 或等效的 Go 验证库进行请求数据验证
- 所有 API 端点定义清晰的输入 schema
- 验证失败返回 400 Bad Request

**二进制数据编码：**
- 公钥、签名、挑战等二进制数据使用 Base64 编码
- 加密数据使用 Base64 编码传输

**错误响应格式：**
与 Node.js 版本完全一致：
```json
{
  "error": "错误描述信息"
}
```

**HTTP 状态码：**
- `200 OK` - 成功
- `400 Bad Request` - 请求参数错误
- `401 Unauthorized` - 认证失败
- `403 Forbidden` - 权限不足
- `404 Not Found` - 资源不存在
- `500 Internal Server Error` - 服务器内部错误

### 数据模型映射

**Prisma (PostgreSQL) 到 GORM (SQLite) 映射：**

Happy Server Go 必须将以下 Prisma 数据模型映射为 GORM 模型：

**核心模型：**
- `Account` - 用户账户（publicKey, settings, profile）
- `Session` - 会话（metadata, agentState, dataEncryptionKey）
- `Machine` - 设备（metadata, lastActiveAt）
- `Artifact` - 工件（加密数据存储）
- `UserKVStore` - 键值存储
- `AccessKey` - 访问密钥
- `AccountPushToken` - 推送令牌
- `TerminalAuthRequest` - 终端认证请求
- `AccountAuthRequest` - 账户认证请求
- `UserFeedItem` - 用户动态
- `UploadedFile` - 上传文件元数据

**关键字段注意事项：**
- `id`: 使用 CUID 生成（需要 Go 等效库）
- `createdAt/updatedAt`: 自动时间戳
- `seq`: 序列号字段（用于增量同步）
- JSON 字段：使用 SQLite 的 JSON 存储或序列化为 TEXT

### 文件存储规范

**替代 MinIO 的本地存储：**
- **存储路径结构**：`{data_dir}/uploads/{account_id}/{file_id}`
- **元数据存储**：在 SQLite 的 `UploadedFile` 表中存储文件元数据
- **支持操作**：
  - 上传文件（流式写入）
  - 下载文件（流式读取）
  - 删除文件
  - 获取文件元数据

**文件存储要求：**
- 支持大文件（流式处理）
- 原子性写入（临时文件 + 重命名）
- 磁盘空间检查
- 文件权限管理（仅所有者可访问）

### 缓存策略

**替代 Redis 的 BigCache + 事件总线：**

**BigCache 用途：**
- 用户会话缓存（token → userId 映射）
- 活跃连接跟踪
- 临时数据缓存（如认证挑战）
- 频繁访问数据的缓存

**事件总线用途：**
- WebSocket 事件分发（替代 Redis Pub/Sub）
- 会话更新通知
- 设备状态变更通知
- 工件更新通知
- 跨 WebSocket 连接的消息广播

**事件总线实现要求：**
- 支持订阅/发布模式
- 支持 topic/channel 概念
- 内存高效（轻量级）
- 并发安全

### API 兼容性保证

**兼容性验证方法：**
- **集成测试套件**：针对所有 API 端点的集成测试，与 Node.js 版本对比响应
- **契约测试**：使用 Pact 或类似工具验证 API 契约
- **端到端测试**：使用真实的 Claude Code 客户端进行端到端测试

**兼容性检查点：**
- ✅ 所有 API 端点路径完全一致
- ✅ 请求/响应 JSON schema 完全一致
- ✅ HTTP 状态码使用一致
- ✅ 错误消息格式一致
- ✅ WebSocket 事件类型和数据格式一致
- ✅ 认证流程行为一致
- ✅ 数据加密/解密行为一致

### 非功能性要求

**性能要求：**
- API 响应时间 P95 < 200ms（正常负载）
- WebSocket 消息延迟 < 100ms
- 支持至少 50 个并发 WebSocket 连接
- 内存占用 < 500MB（包括所有内置服务）

**可靠性要求：**
- 数据库事务保证 ACID 特性
- WebSocket 自动重连机制
- 优雅关闭（等待请求完成）
- 错误日志记录完整

**安全要求：**
- 所有 API 端点强制 HTTPS（生产环境）
- 敏感数据加密存储
- 防止 SQL 注入（使用参数化查询）
- 防止 XSS 和 CSRF（API 层面）

### Implementation Considerations

**从 Node.js/TypeScript 到 Go 的迁移策略：**

1. **分阶段实现**：
   - Phase 1: 核心 API（auth, sessions, machines）
   - Phase 2: 数据同步 API（kv, artifacts）
   - Phase 3: WebSocket 实时通信
   - Phase 4: 推送通知和高级功能

2. **代码组织**：
   - 模仿 Node.js 版本的模块结构（modules, routes, handlers）
   - 使用清晰的包划分（api, services, storage, utils）

3. **测试策略**：
   - 单元测试：业务逻辑层
   - 集成测试：API 端点（与 Node.js 版本对比）
   - 端到端测试：完整流程验证

4. **监控和日志**：
   - 结构化日志（zap）
   - 请求/响应日志
   - 性能指标（内存、CPU、延迟）
   - WebSocket 连接统计

5. **部署考虑**：
   - 单二进制文件编译
   - 配置管理（环境变量 + 配置文件）
   - 数据目录初始化
   - 数据库迁移（golang-migrate）
