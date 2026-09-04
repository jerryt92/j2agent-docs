# 知识库维护

本模块描述 `knowledge-repo` 的 **入库与维护**：目录约定、本地与远程知识库仓库、增量同步、启动初始化、**管理员全局完全重建**、中断恢复、Redisson 分布式锁与维度一致性。

## 文档

| 文档 | 说明 |
|------|------|
| [知识库维护](知识库维护.md) | 完整机制：目录/高级配置、远程 Git、分片、同步、**完全重建**、协调器、exclusive 门禁、中断恢复、列表状态展示 |
| [静态文件展示机制](../静态文件展示机制.md) | 图片等静态资源 URL 改写与 HTTP 直链（不入向量库） |

## 完全重建速览

| 项 | 说明 |
|----|------|
| 谁能触发 | **仅管理员**（`POST .../knowledge/sync?fullRebuild=true`），或启动探测 / Embedding 配置变更自动触发 |
| 单库能否重建 | **否**；单库只有增量 `SYNC`，不会 drop collection |
| 准备顺序 | 停任务 → 清 `knowledge_source_file_hash` + `knowledge_text_chunk` → drop 知识库 collection → `resetClient()` → probe → 全量 re-embed |
| 列表展示 | 所有有库权限的用户见 `GLOBAL_REBUILDING` 状态标签，**无文件进度** |
| 详细进度 | 管理员在「知识列表 → 同步状态」查看 `GET .../knowledge/sync/status` |
| 中断恢复 | `SYNCING` 哈希残留清理、孤儿 collection drop、协调器/按库任务状态回收（见主文档 §9.5、§10.5） |

## 代码入口

| 模块 | 路径 |
|------|------|
| 单库任务与锁 | `.../service/rag/knowledge/repository/RepositoryMaintenanceService.java` |
| 全局维护协调器 | `.../service/rag/knowledge/repo/KnowledgeRepoMaintenanceCoordinator.java` |
| Redis 维护锁 | `.../service/rag/knowledge/repo/KnowledgeRepoMaintenanceLockService.java` |
| 同步与完全重建 | `.../service/rag/knowledge/repo/KnowledgeRepoSyncService.java` |
| 哈希树（含 SYNCING） | `.../service/rag/knowledge/repo/KnowledgeRepoHashTreeService.java` |
| 向量库初始化 | `.../config/rag/VectorDatabaseInit.java` |
| Embedding 变更委托 | `.../service/embedding/EmbeddingChangeOrchestrator.java` |
| Milvus 写入 | `.../service/rag/knowledge/MilvusKnowledgeWriteService.java` |
| 知识库列表与状态展示 | `.../service/rag/knowledge/repository/KnowledgeRepositoryService.java` |

## 知识库列表与远程知识库

知识库列表支持为本地文件知识库与远程 Git 知识库统一设置高级配置；目录内文件配置不再参与知识库元数据解析。根目录下尚未登记的一级目录会自动补登记为本地文件知识库并使用默认高级配置；远程 Git 知识库需在列表中显式创建配置。

| 项 | 规格 |
|----|------|
| 配置范围 | `KnowledgeRepositoryConstants.TYPE_LOCAL_FILE` 与 `TYPE_REMOTE` |
| 前端入口 | 知识库列表页新增/编辑配置时填写“高级配置” |
| 落库位置 | `knowledge_repository`；高级配置在 `metadata_config` JSON 中，名称与权限属性使用独立列 |
| DTO 字段 | `displayName`、`collectionName`、`partitionNames`、`minHeadingLevel`、`filenameAsTitle` |
| 默认规则 | collection 为空时生成 `kb_{repoCode}`；分区为空表示默认分区；`minHeadingLevel=3`；`filenameAsTitle=true` |
| 展示规则 | `displayName` 为空时以 `repoCode` 作为展示回退；聊天选择使用 repositoryId，内部解析真实 collection |
| 状态展示 | 见 [§2.2 知识库列表状态展示](知识库维护.md#22-知识库列表状态展示) |

展示名称只影响前端显示，不改变 Milvus collection 名称。聊天请求使用 `knowledgeRepositoryIds`；仓库授权互相独立，不因共享 collection 合并。当前知识库管理接口要求 `KNOWLEDGE_ADMIN`，普通用户不能创建、修改、同步或删除知识库；管理、授权、删除和用户删除后的所有权规则见 [用户权限](../../安全与用户/用户权限.md)。

远程知识库机制已合并到 [知识库维护 §2.1](知识库维护.md#21-远程知识库机制)：远端内容通过 JGit 同步到 `knowledge-repo/{repoCode}`，随后复用本地知识库的扫描、分片、增量入库与静态资源直链流程。

## 与检索模块的关系

查询侧混合检索、超长 query 融合、维护期间检索降级见 [检索模块](../检索/README.md)。
