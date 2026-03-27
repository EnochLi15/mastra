# Agent 能力审计（2026-03-27）

本报告基于仓库源码静态扫描（不含 `examples/`）给出能力判断。

判定说明：

- ✅ 明确支持
- 🟡 部分支持 / 需组合实现
- ❌ 未发现明确实现

| 能力               | 判定 | 依据（源码要点）                                                                                                         |
| ------------------ | ---- | ------------------------------------------------------------------------------------------------------------------------ |
| Skill 支持         | ✅   | WorkspaceSkills 提供技能发现、检索、解析与去歧义；支持技能列表/读取。                                                    |
| Subagent 支持      | ✅   | Agent 类型定义包含 `agents?: ...` 子代理；`network()` 明确“路由代理委派给子代理”。                                       |
| MCP 支持           | ✅   | `@mastra/mcp` 同时提供 `MCPClient` 与 `MCPServer`；服务端支持 MCP 路由与传输。                                           |
| 多 Agent 支持      | ✅   | `network()` 支持多代理协作循环；Mastra 配置可注册多个 agents。                                                           |
| 多用户支持         | 🟡   | Memory 体系以 `resourceId` + `threadId` 做隔离；请求上下文可强制覆盖避免越权，但未看到完整租户系统。                     |
| 状态持久化         | ✅   | `storage` 说明用于持久化会话历史与 workflow state；memory 线程可保存/更新。                                              |
| 会话管理           | ✅   | Memory 提供线程创建/更新/删除 API；MCP 传输支持 session 选项与自定义 sessionId。                                         |
| 长期记忆           | ✅   | 支持 working memory、semantic recall、向量检索/更新。                                                                    |
| 沙箱机制           | ✅   | WorkspaceSandbox 抽象用于隔离执行，支持 E2B/Modal/Docker/Local 等 provider。                                             |
| Web 服务机制       | ✅   | Server adapter 汇总大量 REST/MCP 路由（agents/tools/memory/workspace/mcp/observability 等）。                            |
| 可扩展性           | ✅   | Mastra Config 支持 tools/processors/memory/workspace/gateways/mcpServers/events 等注入式扩展。                           |
| 分布式文件存储支持 | 🟡   | Workspace 支持 mount 多文件系统并给出 S3Filesystem 示例；sandbox mount 描述含 s3fs/gcsfuse，但具体 provider 多在外部包。 |
| 多模型支持         | ✅   | README 明确 model routing 多 provider；核心 router/gateway/provider registry 已实现。                                    |
| 内置工具           | ✅   | workspace tools 暴露读写文件、搜索、命令执行、进程管理等内置工具。                                                       |
| MCP 管理           | ✅   | 提供 `/stored/mcp-clients` 的 CRUD/版本化 API；并有 MCP registry/tool routes。                                           |
| Skill 管理         | ✅   | 提供 `/stored/skills` CRUD + publish API；workspace skills 运行态接口。                                                  |
| Subagent 管理      | 🟡   | `stored-agents` schema 支持 `agents` 配置（可静态/条件化）；但无独立“subagent 资源域”管理 API。                          |
| 可观测             | ✅   | Observability 配置入口 + server 侧 traces/logs/metrics/scores/feedback 路由。                                            |

## 总结

1. 该仓库在 **Agent 编排（多 Agent/Subagent）**、**记忆体系**、**MCP 协议**、**可观测**、**工具与工作区执行** 方面能力较完整。
2. “多用户”“分布式文件存储”“Subagent 管理”更偏 **架构可支持**，通常需结合上层身份系统、外部 filesystem provider 与业务管理面板落地。
3. 如果你要做“企业级 Agent 平台”，当前代码最值得优先复用的是：
   - `packages/core` 的 Agent/Memory/Workspace/Router 扩展点
   - `packages/server` 的 API 路由与存储化管理（stored-\*）
   - `packages/mcp` 的标准协议接入层

## 追加问题答复

### 1) 是否支持通过 SDK 提供服务？

**结论：支持。**

- 仓库提供 `@mastra/client-js`，README 明确其用于与 Mastra API 交互，覆盖 agents/vectors/memory/tools/workflows 等。
- `MastraClient` 在实现中直接封装了这些资源接口（如 Agent/Workflow/Tool/Vector/Observability/Stored\* 等）。
- Server 端已提供统一路由聚合（含 agents/tools/memory/workspace/mcp/observability/stored-\*），可作为 SDK 对接的服务面。

### 2) 是否支持隐私安全、敏感信息不上云？

**结论：支持“本地优先与脱敏能力”，但是否“绝不上云”取决于你的部署与模型/存储选型。**

- Server 流式输出支持敏感请求体脱敏（默认 redaction 开启），会移除系统提示词、工具定义、API key 等请求数据后再下发给客户端。
- Workspace/Sandbox 可本地运行，LocalSandbox 支持本地执行与可选 OS 级隔离（seatbelt/bwrap）。
- Workspace 支持本地文件系统与可插拔 provider；也支持挂载云文件系统，意味着“是否上云”是可配置策略。
- 因此：若要满足“敏感信息不上云”，建议组合使用本地部署 + 本地模型网关/私有模型 + 本地存储 + redaction，并禁用外部云 provider。
