# 知识库维护

本模块描述 `knowledge-repo` 的 **入库与维护**：目录约定、增量同步、启动初始化、完全重建、Redis 分布式锁与维度一致性。

## 文档

| 文档 | 说明 |
|------|------|
| [知识库维护](知识库维护.md) | 目录/高级配置、分片、同步、协调器、exclusive 门禁、维度一致性 |
| [静态文件展示机制](../静态文件展示机制.md) | 图片等静态资源 URL 改写与 HTTP 直链（不入向量库） |

## 代码入口

| 模块 | 路径 |
|------|------|
| 维护协调器 | `.../service/rag/knowledge/repo/KnowledgeRepoMaintenanceCoordinator.java` |
| Redis 维护锁 | `.../service/rag/knowledge/repo/KnowledgeRepoMaintenanceLockService.java` |
| 同步逻辑 | `.../service/rag/knowledge/repo/KnowledgeRepoSyncService.java` |
| 向量库初始化 | `.../config/VectorDatabaseInit.java` |
| Embedding 变更委托 | `.../service/embedding/EmbeddingChangeOrchestrator.java` |
| Milvus 写入 | `.../service/rag/knowledge/MilvusKnowledgeWriteService.java` |
| 远程知识库仓库配置 | `.../service/rag/knowledge/repository/KnowledgeRepositoryService.java` |

## 知识库列表高级配置

知识库列表支持为本地文件知识库与远程 Git 知识库统一设置高级配置；目录内文件配置不再参与知识库元数据解析。根目录下尚未登记的一级目录会自动补登记为本地文件知识库并使用默认高级配置。

| 项 | 规格 |
|----|------|
| 配置范围 | `KnowledgeRepositoryConstants.TYPE_LOCAL_FILE` 与 `TYPE_REMOTE` |
| 前端入口 | 知识库列表页新增/编辑配置时填写“高级配置” |
| 落库位置 | `knowledge_repository` 表的独立列 |
| DTO 字段 | `displayName`、`collectionName`、`partitionNames`、`minHeadingLevel`、`filenameAsTitle` |
| 默认规则 | collection 为空时生成 `kb_{repoCode}`；分区为空表示默认分区；`minHeadingLevel=3`；`filenameAsTitle=true` |
| 展示规则 | `displayName` 为空时以 `repoCode` 作为展示回退；检索请求仍使用真实 collection id |

展示名称只影响前端显示，不改变 Milvus collection 名称，也不改变聊天请求中的 `knowledgeCollections` 真实 id。

## 与检索模块的关系

查询侧混合检索、超长 query 融合、维护期间检索降级见 [检索模块](../检索/README.md)。
