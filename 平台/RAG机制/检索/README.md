# 检索

本模块描述 RAG **查询侧**：向量化、混合检索、超长 query 融合，以及知识库维护期间的检索降级。知识库检索和 SimpleRag 检索都复用 `VectorDatabaseService.hybridRetrieval`，但入口服务分开。

## 文档

| 文档 | 说明 |
|------|------|
| [Query 预处理](Query预处理.md) | Advanced RAG：Multimodal / Compression / Rewrite 链 |
| [融合检索](融合检索.md) | Milvus 稠密+稀疏混合检索、QueryChunker 多段融合、配置与验证 |
| [SimpleRag](../SimpleRag.md) | Agent 随附 Markdown 资料的检索入口与生命周期 |

## 代码入口

| 模块 | 路径 |
|------|------|
| 检索引擎 | `j2agent/j2agent-server/.../service/rag/retrieval/Retriever.java` |
| Query 预处理 | `.../rag/query/DefaultQueryTransformers.java`、`MultimodalQueryTransformer.java` |
| 查询切分 | `.../service/rag/retrieval/QueryChunker.java` |
| Milvus 混合检索 | `.../service/rag/vdb/milvus/MilvusService.java` |
| 对话 RAG | `.../service/rag/inf/AbstractCollectionKbRetriever.java` |
| SimpleRag | `.../service/rag/inf/AbstractSimpleRagRetriever.java`、`.../service/rag/SimpleRagStoreSyncService.java` |
| Markdown grep/read | `j2agent-plugins-agents/common-tools/.../plugins/tool/KnowledgeRepoGrepTools.java` |

## 仓库授权与范围传递

知识库的授权单位是仓库 ID，collection 是物理存储位置。普通用户的可读仓库由 SQL 条件“公开 OR 本人创建 OR 有效等级 1/2 授权”决定；管理员省略授权过滤，但不跳过维护可用性检查。

通用知识库助手提交 `knowledgeRepositoryIds`。`ChatService` 在启动检索与 LLM 前逐项校验显式选择；任一仓库不存在或无权，整轮以 `KNOWLEDGE_ACCESS_DENIED` 拒绝，不能静默丢掉该项。校验通过后转换为含 repoCode 与 collection 的内部选择值。

业务 Agent 的自动检索范围为“Agent 配置范围 ∩ 用户可读仓库”；存在显式选择时还需与选择范围相交。自动范围为空时不检索，不能降级为无条件搜索。`Retriever` 使用运行期间绑定的原始用户身份（`AgentAccessContext`）解析范围；动态检索保留仓库选择值，不仅传 collection 名。

`KnowledgeReadScope` 在同步检索调用中传递仓库范围：

1. Milvus 稠密、稀疏与混合分支附加 `source_file` 的仓库前缀过滤，限制共享 collection 内的候选。
2. 命中结果再次按来源前缀校验；PostgreSQL 正文回填以块 ID 和仓库范围联合查询，不凭块 ID 单独取正文。
3. 正文列表也附加 collection 与仓库条件；grep 工具按原用户身份重新解析已选范围。文件入口另行校验来源仓库与真实路径。

该 ThreadLocal 范围只覆盖当前同步调用，不能作为异步任务自动继承身份的依据。SimpleRAG 是 Agent 随附资料，沿用 Agent 使用权限，不要求另有仓库授权。

### grep/read 工具权限边界

插件通用工具 [`KnowledgeRepoGrepTools`](../../../../j2agent-plugins-agents/common-tools/src/main/java/io/github/jerryt92/j2agent/plugins/tool/KnowledgeRepoGrepTools.java) 按绑定的知识库子目录工作。例如 `kbRelativeSubPath=j2agent-docs` 时，只代表 `j2agent-docs` 这个仓库代码，不是可以任意扫描的文件系统目录。

- `grep_knowledge_repo` 和 `read_knowledge_repo_file` 都先从 `ToolContext` 解析当前轮原始用户，再调用 `ResourceAccessService.requireRepository(user, repoCode, 2)` 校验读权限；未注入权限服务、没有当前用户、用户无权或仓库标识非法时直接返回“无权访问该知识库”，不会扫描或读取磁盘。
- `read_knowledge_repo_file` 在仓库校验后还会按来源路径调用 `requireSource`，仅允许仓库内的 `.md` 文件；绝对路径、`..` 路径和越出绑定仓库的路径均拒绝。
- grep 只匹配 Markdown 正文行和文件名，忽略大小写；单次最多扫描 500 个文件、返回 40 处命中，单文件上限 512 KiB，并保留命中行前后各 2 行上下文。较长中文关键词无直接命中时会按中文片段拆词回退。
- read 默认最多返回 32,000 字符，单文件同样受 512 KiB 限制；可选图片改写和命中文件来源发布不改变权限校验。

文件预览、下载及历史引用的撤权行为见 [静态文件展示机制](../静态文件展示机制.md)。共享 collection 不是授权单位；所有检索分支、正文回填、来源文件和 grep/read 都必须回到仓库权限边界。

## 与知识库维护的关系

- 入库、增量同步、完全重建见 [知识库维护](../知识库维护/README.md)。
- Agent 随附 Markdown 资料的入库、刷新与清理见 [SimpleRag](../SimpleRag.md)。
- `KnowledgeRepoMaintenanceCoordinator.isExclusiveSyncActive()` 为 true 时，检索返回降级/空结果，避免用新维度 query vector 查询旧 collection schema。
- Milvus 检索前会校验 query 向量维度与 `lastDimension`、collection schema 一致。

- 单库状态为 `REBUILDING`、`REBUILD_FAILED` 或 `DELETING` 时，自动范围排除该库；显式选择返回 `409 KNOWLEDGE_UNAVAILABLE`。其他仓库保持可检索；全局 exclusive 仍按上述规则影响全局检索。
