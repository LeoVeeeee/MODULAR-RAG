建议采用 **3 天、约 20 小时净学习时间**的计划：Day 1 理解数据如何入库，Day 2 理解查询如何变成检索结果并通过 MCP 返回，Day 3 理解扩展、排障和质量验证。

已完成 `project-learner` Skill 全文阅读、规格相关章节研读、`src/` 和 `tests/` 文件扫描，以及入口和主干实现核对。以下计划依据当前源码；本轮没有执行问答、修改业务代码或写入学习评分。

# 一、整体学习策略

1. **先修正项目入口认知。** [main.py](/E:/文档/Program/Github/MODULAR-RAG/main.py:15) 目前只加载配置、初始化日志，没有启动 MCP。实际服务入口是 `src/mcp_server/server.py`。因此，学习从 `scripts/ingest.py`、`scripts/query.py` 两条业务路径开始，再进入 MCP。

2. **学习主线按数据依赖展开。**  
   配置与数据契约 → 摄取 → 编码与存储 → Dense/Sparse 召回 → RRF → Rerank → Response → MCP → Trace/Evaluation → 生命周期与测试。D6 配置和抽象贯穿全过程，第三天再集中讨论扩展。

3. **Day 1 建立数据模型。** 重点追踪 `Document → Chunk → dense_vectors/sparse_stats → 存储记录`。当前实现中，`VectorUpserter` 会生成存储 ID，Pipeline 再把 BM25 的 `chunk_id` 对齐到这些 ID；这是第二天理解稀疏检索回查正文的前提。

4. **Day 2 建立查询模型。** 核心结果类型是 `RetrievalResult`。CLI 查询主要打印结果；MCP 查询再经过 `ResponseBuilder` 组织片段、引用和图片。**这里的 Response 构建没有调用 LLM 生成最终问答答案**，不要把它画成服务器内的答案生成链。

5. **Day 3 建立工程判断能力。** 学会回答“换 Provider 改哪里”“某一路失败会怎样”“指标为何为空”“如何定位召回与重排问题”。这些比读完全部 Provider 更适合第一轮求职准备。

6. **理论紧贴刚读过的代码。** 每个阅读块约按“70% 代码、15% 理论、15% 闭卷输出”分配。`DEV_SPEC.md` 只在实现与设计产生疑问时回查对应小节，不重新通读。

7. **限制跳转深度。** 从主流程最多向下追两层。遇到工具函数，先记录“输入、输出、职责”，把调用位置留作返回点。每个块结束必须回到入口复述，不能停在某个 Provider 内部。

8. **用 Skill 的 45 个知识点作索引，用 8 个知识点作阶段抽测。** 当前 [学习进度文件](/E:/文档/Program/Github/MODULAR-RAG/.agents/skills/project-learner/references/LEARNING_PROGRESS.md) 的两个进度表均未记录已学习项目，历史为空。这表示尚无正式验收记录，不否定你已经完成的规格阅读。

第一轮的取舍如下：

| 知识域 | 本轮处理方式 |
|---|---|
| D1、D2、D3 | 深入主调用链和数据契约 |
| D4 | 深入集成、当前 LLM 后端和降级；CrossEncoder 理解接口与原理 |
| D5 | 深入工具注册、请求分发、返回边界和生命周期 |
| D6 | 深入配置、Factory、Base 与实现关系；不逐个读完厂商代码 |
| D7 | 贯穿摄取和 Response；Vision 当前关闭，先理解图文关联 |
| D8 | 深入 Trace、实际指标和 EvalRunner；Dashboard 页面实现略读 |
| D9 | 选择代表性测试，理解验证边界 |
| D10 | 深入去重、更新边界和跨存储删除，不把幂等等同于事务 |

# 二、3 天时间表

下面的 **B01—B17 是阅读块，CP 是验收块**。表中的文件名用于快速定位，后面的块说明提供完整路径及阅读顺序。所有学习块为 30–75 分钟；休息单独安排。

每块统一填写一张“模块卡”，不适用的项明确写“不适用”：

> 输入 / 输出｜调用者 / 被调用者｜关键数据和状态｜Factory 或直接构造｜配置读取点｜异常处理位置

## Day 1：从文件到可检索数据

**15:30–22:15，净学习 5 小时 30 分钟。**

| 时间 | Knowledge Domain | 学习主题 | 核心代码 | 理论 | 验收 |
|---|---|---|---|---|---|
| 15:30–16:30 | D1.2–1.5、D6.2 | B01：入口、配置、数据契约 | main、ingest、settings、types、YAML | 分层、契约、配置驱动 | 画出入口和数据类型关系 |
| 16:30–16:45 | — | 休息 | — | — | 离开屏幕 |
| 16:45–18:00 | D2.1–2.2、D7.1 | B02：加载与切分 | pipeline、pdf_loader、document_chunker、两份 splitter 文件 | Chunk 大小与重叠 | 解释一个 PDF 如何成为 Chunks |
| 18:00–18:45 | — | 晚餐 | — | — | — |
| 18:45–19:45 | D2.3、D7.2–7.3 | B03：Transform 链 | pipeline、base_transform、三个 transform | 增强收益、成本与降级 | 画出文本和 metadata 的变化 |
| 19:45–20:00 | — | 休息 | — | — | — |
| 20:00–21:15 | D2.4、D6.4 | B04：双编码和批处理 | batch_processor、dense_encoder、sparse_encoder、embedding_factory、ollama_embedding | 向量表示、词项统计、批处理 | 解释编码结果的顺序和数量契约 |
| 21:15–21:45 | D2.5、D10.1–10.2 | B05：存储和去重边界 | pipeline、vector_upserter、bm25_indexer、file_integrity | 稳定 ID、upsert、幂等 | 画出 Chunk ID 到存储 ID 的映射 |
| 21:45–22:15 | D2.1、D2.5 | CP1：摄取与存储验收 | 回查 pipeline、vector_upserter、bm25_indexer | 数据一致性 | 闭卷讲完摄取链并记录薄弱点 |

### B01：知道程序从哪里进入，数据如何传递

**5 个文件，顺序：**

`main.py` → `scripts/ingest.py` → `config/settings.yaml` → `src/core/settings.py` → `src/core/types.py`，最后返回 `ingest.main()`。

- **目标：** 能解释 CLI 参数、Settings 和 Pipeline 的责任边界。
- **重点：** `load_settings()` 如何读 YAML、构建 dataclass 和校验；`--collection`、`--force` 如何传入 Pipeline；配置错误在哪一层转换为退出码。
- **数据契约：** 先看 `Document`、`Chunk`、`ProcessedQuery`、`RetrievalResult` 的字段。`ChunkRecord` 只了解用途，暂不假设所有存储路径都经过它。
- **验收：** 不看代码说明“改 YAML”和“改 CLI 参数”分别影响哪里，并指出 `main.py` 尚未承担的职责。

### B02：从完整 Document 追到 Chunk 列表

**5 个文件，顺序：**

`src/ingestion/pipeline.py`  
→ `src/libs/loader/pdf_loader.py`  
→ 回到 Pipeline  
→ `src/ingestion/chunking/document_chunker.py`  
→ `src/libs/splitter/splitter_factory.py`  
→ `src/libs/splitter/recursive_splitter.py`  
→ 回到 Pipeline。

- **目标：** 解释文件路径如何转成 `Document`，再变成 `List[Chunk]`。
- **重点：** Loader 直接实例化，Splitter 经 Factory 创建；正文、`source_ref`、`chunk_index`、图片引用如何传递；空文本和切分失败在哪处理。
- **理论：** `chunk_size` 与 `chunk_overlap` 如何影响上下文完整性、重复召回和索引规模；不要把字符切分参数直接当成 token 数。
- **验收：** 手画一个包含两张图片的文档切分过程，指出每个 Chunk 为什么只应关联对应图片。

### B03：明确增强发生在编码之前

**5 个文件，顺序：**

`src/ingestion/pipeline.py`  
→ `src/ingestion/transform/base_transform.py`  
→ `src/ingestion/transform/chunk_refiner.py`  
→ `src/ingestion/transform/metadata_enricher.py`  
→ `src/ingestion/transform/image_captioner.py`  
→ 返回 Pipeline 的 Encoding 阶段。

- **目标：** 解释 `Refiner → Enricher → Captioner` 的执行顺序及输入输出。
- **重点：** 都处理 Chunk 列表，但正文和 metadata 的变化不同；LLM 通过 Factory 获取；区分禁用增强、Provider 初始化失败和单次增强失败。
- **当前配置：** Refiner、Enricher 的 `use_llm` 为 true，Vision 为 false。先沿实际启用路径阅读，再看关闭时的分支。
- **理论：** 内容增强改善召回的可能性，以及改写失真、额外耗时和模型费用的代价。
- **验收：** 写出“LLM 不可用时还剩下哪些能力”，避免笼统回答“整个 Pipeline 失败”。

### B04：理解批次、顺序和双编码输出

**5 个文件，顺序：**

`src/ingestion/embedding/batch_processor.py`  
→ `src/ingestion/embedding/dense_encoder.py`  
→ `src/libs/embedding/embedding_factory.py`  
→ `src/libs/embedding/ollama_embedding.py`  
→ 返回 BatchProcessor  
→ `src/ingestion/embedding/sparse_encoder.py`。

- **目标：** 解释 `List[Chunk]` 如何得到 `BatchResult` 中的向量和词项统计。
- **重点：** 当前 Embedding Provider 为 Ollama；DenseEncoder 检查返回数量和维度；BatchProcessor 如何计数成功、失败批次。
- **重要实现事实：** 当前 BatchProcessor 在每批内先执行 Dense，再执行 Sparse；不能仅凭“双路编码”称它为并行执行。
- **理论：** 文档与查询必须使用兼容的向量空间；稀疏统计与 Dense 向量不是同一种表示。
- **验收：** 推演“第二批失败”时返回列表与原始 chunks 是否仍对齐，指出下游需要验证的契约。

### B05：连接第一天与第二天的关键块

**4 个文件，顺序：**

`src/ingestion/pipeline.py`  
→ `src/ingestion/storage/vector_upserter.py`  
→ 返回 Pipeline 的 ID 对齐代码  
→ `src/ingestion/storage/bm25_indexer.py`  
→ `src/libs/loader/file_integrity.py`  
→ 返回 Pipeline 的成功与失败分支。

- **目标：** 解释文件 hash、切分时 Chunk ID、最终向量 ID 的区别。
- **重点：** VectorUpserter 经 Factory 获取向量库；Pipeline 将 `sparse_stats` 的 ID 改为写入向量库的 ID；全部存储步骤之后才标记摄取成功。
- **理论：** 重复执行可跳过、重复写入可覆盖、跨存储原子性是三个不同问题。
- **验收：** 回答“Chroma 成功、BM25 失败后重试会怎样”；指出当前流程没有跨三个存储的统一事务。

## Day 2：从查询到 MCP 返回

**09:00–17:45 学习，随后晚餐；净学习 7 小时。**

| 时间 | Knowledge Domain | 学习主题 | 核心代码 | 理论 | 验收 |
|---|---|---|---|---|---|
| 09:00–10:15 | D1.5、D3.1 | B06：查询组装与 Dense | query、hybrid_search、dense_retriever、embedding_factory、chroma_store | 向量空间、cosine | 画出 query 到候选结果链 |
| 10:15–10:30 | — | 休息 | — | — | — |
| 10:30–11:45 | D3.2、D3.4 | B07：预处理、BM25 和正文回查 | query_processor、sparse_encoder、bm25_indexer、sparse_retriever、chroma_store | 分词、TF/IDF、长度归一化 | 解释“BM25 命中但正文缺失” |
| 11:45–12:15 | D3.3 | B08：融合和单路降级 | hybrid_search、fusion、test_fusion_rrf | RRF | 手算融合并解释降级 |
| 12:15–13:15 | — | 午餐、休息 | — | — | — |
| 13:15–14:30 | D4.1–4.4 | B09：重排和回退 | core reranker、base、factory、llm、cross_encoder | Bi-Encoder / Cross-Encoder、候选预算 | 解释正常、关闭、失败三条路径 |
| 14:30–14:45 | — | 休息 | — | — | — |
| 14:45–15:15 | D3.3、D4.4 | CP2：检索与重排验收 | 回查 hybrid_search、fusion、core reranker | 召回与排序的分工 | 能定位相关片段在哪一阶段丢失 |
| 15:15–16:15 | D3.5、D7.4 | B10：结果、引用和图片 | query tool、response_builder、citation_generator、multimodal_assembler、image_storage | 溯源、图文关联 | 区分检索返回与答案生成 |
| 16:15–16:30 | — | 休息 | — | — | — |
| 16:30–17:45 | D5.1–5.4 | B11：MCP 服务边界 | server、protocol_handler、三个 tools | JSON-RPC、stdio、异步边界 | 画出 tools/call 到返回的时序 |
| 17:45–18:45 | — | 晚餐，结束当天学习 | — | — | — |

### B06：沿真实 CLI 查询入口进入 Dense

**5 个文件，顺序：**

`scripts/query.py` 的 `_build_components()`  
→ `src/libs/embedding/embedding_factory.py`  
→ 返回 `_run_query()`  
→ `src/core/query_engine/hybrid_search.py`  
→ `src/core/query_engine/dense_retriever.py`  
→ `src/libs/vector_store/chroma_store.py`  
→ 返回 CLI 打印结果的位置。

- **目标：** 解释字符串 query 如何变成 `List[RetrievalResult]`。
- **重点：** 组件在哪里组装；Embedding 与 VectorStore 如何注入 Retriever；query embedding、数据库查询、结果转换分别在哪。
- **配置：** 追踪 `dense_top_k`、collection 与显式 `top_k`；初始化失败和检索失败由不同层处理。
- **理论：** cosine distance 与输出 score 的转换；不要把结果分数自动解释成置信概率。
- **验收：** 解释为什么换 Embedding 模型后不能直接假设旧索引仍可正常检索。

### B07：理解 Sparse 为什么仍依赖向量库

**5 个文件，顺序：**

`src/core/query_engine/query_processor.py`  
→ `src/ingestion/embedding/sparse_encoder.py`  
→ `src/ingestion/storage/bm25_indexer.py`  
→ `src/core/query_engine/sparse_retriever.py`  
→ `src/libs/vector_store/chroma_store.py` 的 `get_by_ids()`  
→ 返回 SparseRetriever。

- **目标：** 解释 `query → keywords → BM25 命中 ID → 正文和 metadata → RetrievalResult`。
- **重点：** 摄取与查询的分词关系；索引 load/query；`default_collection`；缺失索引与回查缺失记录的行为。
- **实现边界：** QueryProcessor 当前主要做规范化、分词、停用词处理和过滤语法解析，不能把规格里的同义词扩展直接算作已实现能力。
- **理论：** BM25 的 TF 饱和、IDF、文档长度归一化；SparseEncoder 提供词项统计，BM25Indexer 才结合语料统计计算检索分数。
- **验收：** 给出“BM25 命中但没有最终正文”的排查顺序：集合 → 索引 → ID 对齐 → `get_by_ids()`。

### B08：理解排名融合，不混加原始分数

**3 个文件，顺序：**

`src/core/query_engine/hybrid_search.py`  
→ `src/core/query_engine/fusion.py`  
→ `tests/unit/test_fusion_rrf.py`  
→ 返回 HybridSearch 的降级和过滤分支。

- **目标：** 解释两路结果如何按 `chunk_id` 合并，并形成最终候选。
- **重点：** `rrf_k` 与返回 `top_k` 的不同；同分排序；单路失败、双路失败、双路都为空的差别。
- **理论：** \(RRF(d)=\sum_i 1/(k+rank_i(d))\)。
- **验收：** 对 Dense `[A,B,C]`、Sparse `[B,D,A]` 手算排序，并解释为什么不直接加 BM25 分数与 Dense 分数。

### B09：先读 Core 编排，再进入当前后端

**5 个文件，顺序：**

`src/core/query_engine/reranker.py`  
→ `src/libs/reranker/base_reranker.py`  
→ `src/libs/reranker/reranker_factory.py`  
→ `src/libs/reranker/llm_reranker.py`  
→ 返回 CoreReranker  
→ 最后略读 `src/libs/reranker/cross_encoder_reranker.py` 的输入输出。

- **目标：** 解释 `RetrievalResult → candidates → 后端排序 → RerankResult`。
- **重点：** 当前配置启用 LLM 重排；LLMReranker 经 `LLMFactory` 获得模型；解析失败、后端异常和禁用重排的处理。
- **配置：** 比较 YAML 默认 `top_k` 与调用时显式 `top_k`；沿调用处确认实际候选数，不能只背“20→10→5”。
- **理论：** 召回负责覆盖，重排负责候选排序；CrossEncoder 与 LLM Rerank 的延迟、成本和稳定性取舍。
- **验收：** 回答“正确片段没进入候选集时，重排能否救回来”；说明为何代码中存在 `timeout` 字段不等于已经实现超时取消。

### B10：把 Response 的职责说准确

**5 个文件，顺序：**

`src/mcp_server/tools/query_knowledge_hub.py`  
→ `src/core/response/response_builder.py`  
→ `src/core/response/citation_generator.py`  
→ `src/core/response/multimodal_assembler.py`  
→ `src/ingestion/storage/image_storage.py`  
→ 返回 `to_mcp_content()` 的调用处。

- **目标：** 解释检索结果如何变成 MCP 可消费的文本、引用和图片内容块。
- **重点：** `RetrievalResult → MCPToolResponse → TextContent/ImageContent`；来源字段、图片引用与实际文件的关联；空结果处理。
- **构造方式：** ResponseBuilder 可以直接构造或注入，并非所有组件都经过 Factory。
- **理论：** 引用提供可追溯性，但不自动证明内容正确；图片 Caption 用于文本检索，原图用于返回展示。
- **验收：** 用两句话区分“本服务提供了什么”和“客户端如何据此生成回答”。

### B11：业务链已经懂，再理解协议外壳

**5 个文件，顺序：**

`src/mcp_server/server.py`  
→ `src/mcp_server/protocol_handler.py`  
→ `src/mcp_server/tools/query_knowledge_hub.py`  
→ 对照 `src/mcp_server/tools/list_collections.py`  
→ 对照 `src/mcp_server/tools/get_document_summary.py`  
→ 返回 Server 生命周期。

- **目标：** 解释工具 Schema、注册、分发和业务调用之间的关系。
- **重点：** 官方 SDK 承担的协议工作；`register_tool()` 和 `execute_tool()` 的职责；错误如何转换成 `CallToolResult`。
- **状态：** 查询工具缓存部分组件，并重建检索相关组件以读取新数据；观察 `_initialized`、collection 和组件缓存。
- **理论：** stdout 是协议通道，stderr 是日志通道；阻塞检索为什么通过 `asyncio.to_thread()` 调用。
- **验收：** 画出 `tools/list` 与 `tools/call` 两条路径，并指出工具异常由哪些层处理。

## Day 3：扩展、排障、评估和工程表达

**09:00–19:15，净学习 7 小时 30 分钟。**

| 时间 | Knowledge Domain | 学习主题 | 核心代码 | 理论 | 验收 |
|---|---|---|---|---|---|
| 09:00–10:00 | D6.1、6.3、6.5 | B12：用一条扩展链理解 Factory | llm_factory、base_llm、deepseek_llm、embedding_factory、test_llm_factory | 依赖倒置、注册、注入 | 设计新增 Provider 的改动清单 |
| 10:00–10:30 | D5.2、D6.1 | CP3：协议与可插拔架构验收 | 回查 protocol_handler、query tool、llm_factory | 扩展边界 | 分清新增工具与新增 Provider |
| 10:30–10:45 | — | 休息 | — | — | — |
| 10:45–12:00 | D8.1、8.2、8.5 | B13：Trace 和排障 | query、trace_context、trace_collector、trace_service、logger | 请求关联、阶段耗时 | 从慢查询构造排障路径 |
| 12:00–13:00 | — | 午餐、休息 | — | — | — |
| 13:00–14:15 | D8.3–8.4 | B14：评估链与指标边界 | evaluate、eval_runner、evaluator_factory、custom_evaluator、golden_test_set | Hit Rate、MRR、有效评估前提 | 解释空报告与真实零分 |
| 14:15–14:30 | — | 休息 | — | — | — |
| 14:30–15:00 | D8.1、D8.3 | CP4：可观测性与评估验收 | 回查 trace_collector、eval_runner、custom_evaluator | 性能与质量的区分 | 给出排障证据及评估方案 |
| 15:00–16:00 | D10.1–10.4 | B15：生命周期和一致性 | document_manager、file_integrity、vector_upserter、test_document_manager | 幂等、部分失败、补偿 | 分析重复、更新、删除三类操作 |
| 16:00–16:15 | — | 休息 | — | — | — |
| 16:15–17:15 | D9.1–9.4 | B16：测试作为工程证据 | pyproject、conftest、reranker_fallback、hybrid_search、mcp_client | Unit / Integration / E2E | 为三个风险选择正确测试层级 |
| 17:15–18:15 | — | 晚餐、休息 | — | — | — |
| 18:15–19:15 | D1 综合 | B17：闭卷重建与面试表达 | ingest、query、query tool、eval_runner | What / How / Why / Trade-off | 完成最终架构图和口述录音 |

### B12：深入一个 Factory，迁移理解其他 Factory

**5 个文件，顺序：**

`src/libs/llm/llm_factory.py`  
→ `src/libs/llm/base_llm.py`  
→ `src/libs/llm/deepseek_llm.py`  
→ 返回 Factory  
→ 对照 `src/libs/embedding/embedding_factory.py`  
→ `tests/unit/test_llm_factory.py`。

- **目标：** 从“会切换配置”提升到“会设计扩展”。
- **重点：** 注册表、内置注册、Base 契约、`create()`、不支持的 Provider 和构造异常。
- **理论：** Factory 决定实例创建；Base 定义契约；依赖注入让调用者可替换实现与测试替身。
- **验收：** 列出新增一个 LLM Provider 的实现、注册、配置和测试步骤，并解释为什么通常不用改 Ingestion 主流程。
- **边界：** 配置注释列出的选项不保证都有已注册实现；以 Factory 和实际文件为准。

### B13：把 Trace 当作数据流读

**5 个文件，顺序：**

`scripts/query.py`  
→ `src/core/trace/trace_context.py`  
→ `src/core/trace/trace_collector.py`  
→ `src/observability/dashboard/services/trace_service.py`  
→ 对照 `src/observability/logger.py`  
→ 返回查询入口。

- **目标：** 解释一条 Trace 从创建、记录到落盘和读取的全过程。
- **重点：** `trace_id`、`trace_type`、stages、metadata；`finish()` 只结束计时，真正写 JSONL 的是 Collector。
- **配置与异常：** 沿调用点确认是否传入配置路径、开关是否实际被消费；检查提前 return 是否绕过 collect。
- **理论：** 单次请求总耗时与并行阶段耗时不能简单相加；缺少记录也可能是未打点，不能直接判断阶段没有执行。
- **验收：** 写一份“查询变慢”的排查流程，区分初始化、Embedding、数据库、重排和返回组装。

### B14：先证明评估链有效，再讨论分数

**5 个文件，顺序：**

`scripts/evaluate.py`  
→ `src/libs/evaluator/evaluator_factory.py`  
→ `src/observability/evaluation/eval_runner.py`  
→ `tests/fixtures/golden_test_set.json`  
→ `src/libs/evaluator/custom_evaluator.py`  
→ 返回报告输出。

- **目标：** 解释测试样本、检索结果、ground truth 如何形成单题指标和聚合指标。
- **重点：** `GoldenTestCase → QueryResult → EvalReport`；Factory 创建何种 Evaluator；单题异常为什么可能变成空 metrics。
- **理论：** Hit Rate@K 判断是否至少命中一个正确项；MRR 关注首个正确项的名次。Recall@K 则关注找回了多少相关项，不能混为一谈。
- **验收：** 手算三条查询的 Hit Rate 和 MRR，再解释“没有有效评估”和“有效评估后得零分”的区别。

这里安排三个**源码核对题**：

1. 当前 `evaluation.enabled: false` 会得到 `NoneEvaluator`；若改为 true，`custom` 的 metrics 应与其支持集合一致，当前其中的 `faithfulness` 不受支持。
2. `scripts/evaluate.py` 没有向 EvalRunner 注入 reranker，因此不能把该脚本的结果直接称为重排后的质量评估；未注入答案生成器时，默认生成内容是片段拼接。
3. 当前 `CustomEvaluator` 的对象 ID 提取分支检查 `.id`，检索结果类型使用 `.chunk_id`。把它作为接口兼容性的复现题，追踪异常是否被转为空 metrics；不要只看脚本退出码判断成功。

### B15：能描述幂等性的实际边界

**4 个文件，顺序：**

`src/libs/loader/file_integrity.py`  
→ `src/ingestion/storage/vector_upserter.py`  
→ `src/ingestion/document_manager.py`  
→ `tests/unit/test_document_manager.py`。

- **目标：** 区分重复摄取、内容更新、删除和重新摄取。
- **重点：** `should_skip()` 基于文件 hash；向量 ID 包含内容 hash；删除分别协调 Chroma、BM25、图片、摄取历史，并收集部分失败。
- **构造方式：** DocumentManager 接收存储依赖；关注传入的 collection 和已有客户端，而非寻找一个并不存在的统一管理 Factory。
- **理论：** 可重试不意味着自动回滚；稳定 ID 不意味着自动清除旧版本；检查 hash 去重是否符合跨 collection 使用需求。
- **验收：** 给“文档内容修改后旧 Chunk 是否残留”画状态变化，并说明还需要哪些验证证据。

### B16：用测试确认边界，不追求跑完全部测试

**5 个文件，顺序：**

`pyproject.toml`  
→ `tests/conftest.py`  
→ `tests/unit/test_reranker_fallback.py`  
→ `tests/integration/test_hybrid_search.py`  
→ `tests/e2e/test_mcp_client.py`。

- **目标：** 解释不同测试层到底证明了什么。
- **重点：** Fixture 如何准备数据；Fake/Mock 替代哪些依赖；集成测试是否使用真实外部服务；MCP E2E 如何启动子进程与处理 stdio。
- **理论：** 算法正确性、组件协作、协议可用性和实际检索质量需要不同证据。
- **验收：** 为“RRF 算错”“重排失败没有回退”“stdout 混入日志”分别选择测试层级。

在已安装项目开发依赖的 Python 环境中，可运行以下聚焦练习：

```powershell
python -m pytest tests/unit/test_fusion_rrf.py tests/unit/test_reranker_fallback.py tests/integration/test_hybrid_search.py -q
```

运行前先预测两个关键断言；运行后用一句话说明每组测试证明了什么。本轮规划没有执行这些测试。

### B17：闭卷重建，而不是再次浏览全部代码

**4 个回查文件：**

`scripts/ingest.py`、`scripts/query.py`、`src/mcp_server/tools/query_knowledge_hub.py`、`src/observability/evaluation/eval_runner.py`。

- **前 20 分钟：** 关掉代码，画架构、数据流和调用链。
- **中间 20 分钟：** 对照这四个入口，只修正画错或漏掉的部分。
- **最后 20 分钟：** 录制项目介绍，覆盖 What、How、Why、Trade-off，并列出仍不确定的三件事。
- **验收：** 每个主要箭头都能说明传递什么数据、失败在哪里处理。

# 三、代码阅读主线

标记规则：

- **[主线]** 当场读控制流和数据变化。
- **[二级]** 先理解接口，进入指定实现，再返回调用者。
- **[跳过]** 第一轮只记职责，不继续递归追踪。

## 1. 离线摄取链

入口依据：[scripts/ingest.py](/E:/文档/Program/Github/MODULAR-RAG/scripts/ingest.py:183)、[IngestionPipeline.run](/E:/文档/Program/Github/MODULAR-RAG/src/ingestion/pipeline.py:197)。

```text
[主线] scripts/ingest.py::main
├─ load_settings + discover_files + CLI 参数
├─ IngestionPipeline(settings, collection, force)
│  ├─ [二级] 直接创建 SQLiteIntegrityChecker、PdfLoader
│  ├─ [二级] DocumentChunker → SplitterFactory → RecursiveSplitter
│  ├─ [二级] Refiner / Enricher → LLMFactory
│  ├─ [二级] Captioner → 启用时创建 Vision LLM
│  ├─ [二级] EmbeddingFactory → 当前 OllamaEmbedding
│  └─ [二级] VectorUpserter → VectorStoreFactory → ChromaStore
│
└─ 每个文件创建 TraceContext → pipeline.run
   ├─ [主线] hash → should_skip；命中成功记录则提前返回
   ├─ [主线] PdfLoader.load → Document
   ├─ [主线] DocumentChunker.split_document → Chunks
   ├─ [主线] Refiner → Enricher → Captioner
   ├─ [主线] BatchProcessor.process → BatchResult
   │  └─ 每批 DenseEncoder.encode → SparseEncoder.encode
   ├─ [主线] VectorUpserter.upsert → vector_ids
   ├─ [主线] sparse_stats.chunk_id 对齐 vector_ids
   ├─ [主线] BM25Indexer.add_documents
   ├─ [二级] ImageStorage.register_image
   ├─ [主线] mark_success / 失败处理 → PipelineResult
   └─ CLI 调用 TraceCollector.collect
```

**第一轮跳过：** PDF 第三方解析器内部、正则清洗的所有分支、厂商 SDK 网络传输内部、图片压缩细节。

特别保留这个数据关系：

```text
文件 hash
  ├─ 摄取历史去重
  └─ 文档关联

Chunk.id
  └─ 切分阶段产生

VectorUpserter 生成的存储 ID
  ├─ Chroma 中的记录 ID
  └─ BM25 命中后回查正文所使用的 chunk_id
```

## 2. CLI 查询链

入口依据：[组件组装](/E:/文档/Program/Github/MODULAR-RAG/scripts/query.py:135)、[查询执行](/E:/文档/Program/Github/MODULAR-RAG/scripts/query.py:170)。

```text
[主线] scripts/query.py::main
├─ load_settings
├─ _build_components
│  ├─ [二级] VectorStoreFactory、EmbeddingFactory
│  ├─ [主线] DenseRetriever、SparseRetriever
│  ├─ [主线] QueryProcessor、HybridSearch
│  └─ [主线] CoreReranker
└─ _run_query
   ├─ [主线] HybridSearch.search
   │  ├─ QueryProcessor.process → ProcessedQuery
   │  ├─ DenseRetriever.retrieve
   │  │  └─ embed(query) → VectorStore.query → RetrievalResult[]
   │  ├─ SparseRetriever.retrieve
   │  │  └─ BM25 load/query → IDs → get_by_ids → RetrievalResult[]
   │  ├─ 双路正常：RRFFusion.fuse
   │  ├─ 单路失败：按配置走降级分支
   │  └─ 后置过滤、数量截断
   ├─ [主线] 可选 CoreReranker.rerank
   │  └─ RerankerFactory → LLM / CrossEncoder / None
   ├─ [主线] _print_results
   └─ TraceCollector.collect
```

**第一轮跳过：** HNSW 内部实现、模型推理框架内部、所有 Provider 的完整对比。

## 3. MCP 查询链

业务入口依据：[QueryKnowledgeHubTool.execute](/E:/文档/Program/Github/MODULAR-RAG/src/mcp_server/tools/query_knowledge_hub.py:219)。

```text
[主线] python -m src.mcp_server.server
└─ run_stdio_server_async
   ├─ 日志转 stderr、预加载依赖
   ├─ create_mcp_server
   │  └─ ProtocolHandler 注册三个工具
   └─ 官方 SDK 的 stdio_server + server.run
      ├─ tools/list → 返回工具 Schema
      └─ tools/call → ProtocolHandler.execute_tool
         └─ query_knowledge_hub_handler
            └─ QueryKnowledgeHubTool.execute
               ├─ [主线] 参数、collection、top_k
               ├─ [主线] 初始化 / 重建检索组件
               ├─ [主线] _perform_search → HybridSearch
               ├─ [主线] _apply_rerank → CoreReranker
               ├─ [主线] ResponseBuilder.build
               │  ├─ CitationGenerator
               │  └─ [二级] MultimodalAssembler → ImageStorage
               ├─ TraceCollector.collect
               └─ MCPToolResponse.to_mcp_content
                  → CallToolResult → MCP Client
```

`list_collections`、`get_document_summary` 是平行工具，理解各自输入输出即可，不要把它们串入查询主链。

**第一轮跳过：** MCP SDK 内部协议实现、JSON 序列化库内部、所有协议错误码的逐条背诵。

## 4. 观察、评估与管理分支

```text
[主线] TraceContext.record_stage
  → TraceCollector.collect
  → logs/traces.jsonl
  → [二级] TraceService
  → [跳过细节] Dashboard 各页面渲染

[主线] scripts/evaluate.py
  → EvaluatorFactory
  → 组装 HybridSearch
  → EvalRunner.run
     → 加载 GoldenTestCase
     → 检索
     → 生成内容 / 默认片段拼接
     → evaluator.evaluate
     → QueryResult
     → 聚合为 EvalReport

[二级，Day 3 深入] DocumentManager.delete_document
  → Chroma 删除
  → BM25 删除
  → 图片删除
  → 摄取历史删除
  → 汇总 DeleteResult.errors
```

注意：评估 CLI 与 MCP 查询没有完全相同的组装路径；比较效果前必须确认是否使用相同的 collection、候选预算和重排组件。

# 四、理论复习清单

## 必须掌握

| 理论 | 对应代码 | 达标要求 |
|---|---|---|
| 离线摄取与在线查询分离 | `pipeline.py`、`scripts/query.py` | 能解释为什么解析、增强、文档编码通常不在每次查询时执行 |
| 数据契约与模块职责 | `core/types.py`、各入口 | 能说清主要输入输出，而非只记类名 |
| Chunk 大小与 overlap | `document_chunker.py`、`recursive_splitter.py` | 能分析上下文、重复、成本和召回的取舍 |
| Embedding 与 cosine | `dense_encoder.py`、`dense_retriever.py`、`chroma_store.py` | 能解释向量空间兼容性和距离到分数的映射 |
| BM25 | `sparse_encoder.py`、`bm25_indexer.py` | 能解释 TF、IDF、长度归一化及分词影响 |
| RRF | `fusion.py` | 会手算，能区分 `rrf_k` 与 `top_k` |
| 召回与重排分工 | `hybrid_search.py`、`reranker.py` | 能解释召回上限和重排成本 |
| Factory、抽象、依赖注入 | `embedding_factory.py`、`llm_factory.py`、Base 类 | 能提出新增实现的最小改动范围 |
| MCP 服务边界 | `server.py`、`protocol_handler.py` | 能解释 stdio、工具注册、SDK 和业务层分工 |
| 幂等与一致性 | `file_integrity.py`、`vector_upserter.py`、`document_manager.py` | 能区分跳过、覆盖、旧数据清理、部分失败 |
| Trace 与质量评估 | `trace_collector.py`、`eval_runner.py` | 能区分运行性能问题和检索质量问题 |
| Hit Rate、MRR、Recall 的差别 | `custom_evaluator.py` | 会计算前两者，知道 Recall 不是当前 CustomEvaluator 已实现指标 |

## 理解即可

| 内容 | 对应代码 | 本轮边界 |
|---|---|---|
| 图转文的多模态方案 | `image_captioner.py`、`multimodal_assembler.py` | 知道 Caption 用于召回、图片引用用于返回 |
| CrossEncoder 内部模型机制 | `cross_encoder_reranker.py` | 理解联合编码 query-document，比双塔更贵但更适合精排 |
| RAGAS 与组合评估 | `ragas_evaluator.py`、`composite_evaluator.py` | 知道输入契约、额外依赖、指标含义；扩展时再深入 |
| Dashboard 服务层 | `dashboard/services/trace_service.py` | 理解数据从哪里来，不逐页读 UI |
| SQLite、Chroma、BM25 文件的分工 | 对应 storage/loader 实现 | 会说明存什么、用什么 ID 关联 |
| 测试替身和分层 | 三层测试目录 | 会判断测试证明的范围 |

## 第一轮暂时跳过

- 所有 LLM/Embedding 厂商适配的逐行比较。
- Transformer、Embedding 模型的训练与微调细节。
- Chroma/HNSW、PDF 解析器和 MCP SDK 的内部源码。
- 图片压缩、Base64 和 Dashboard 样式的实现细节。
- 规格中的未来方向，如完整 Agentic RAG、分布式部署、多租户和图搜图。

这些内容不阻塞你建立本项目第一轮 Call Graph、Data Flow 和架构模型。

# 五、Project Learner 验收节点

设置 **4 个 Checkpoint，共 8 个正式抽测知识点**。每次 30 分钟，建议：

- 5 分钟：闭卷画图或复述。
- 15 分钟：两个知识点各一个主问题，必要时各追问一次。
- 10 分钟：评分、定向 Learning Guide、记录进度。

不要求每题完成四轮追问。以下是后续验收题目方向，本轮不开始作答。

| Checkpoint | 选取知识点 | 选择原因 | How / Why / Debug / Extend 问题方向 |
|---|---|---|---|
| CP1，Day 1 末 | **D2.1 Pipeline 整体流程、D2.5 存储层协同** | 能同时暴露主调用链与数据契约问题 | 为什么 BM25 ID 要改成 vector ID？Chroma 成功、BM25 失败时，重试的依据和风险是什么？ |
| CP2，Day 2 下午 | **D3.3 Hybrid Search 融合、D4.4 Rerank 集成** | 区分“没召回”和“排得差” | 正确片段在哪一阶段丢失？为什么用 RRF？重排失败后保留什么顺序？ |
| CP3，Day 3 上午 | **D5.2 Tool 注册、D6.1 工厂模式** | 验证能否迁移和扩展架构 | 新增一个工具与新增一个 Provider 分别改哪里？哪些业务模块可以保持不动，为什么？ |
| CP4，Day 3 下午 | **D8.1 Trace、D8.3 评估指标** | 验证能否用证据排障 | 查询变慢与质量下降分别看什么？评估脚本完成却没有指标，按什么顺序检查？ |

**评分沿用 Skill：** 准确性、深度、代码关联、设计思维各 10 分，综合分按四维平均并取最近的 0.5 分。

**通过标准：** 综合分至少 7，且没有核心链路错误。未通过时，Learning Guide 只指定本块的 3–5 个文件中的关键段落，补读后换一个角度复测。

进度处理：

- 实际完成问答后，才更新 `.agents/skills/project-learner/references/LEARNING_PROGRESS.md`。
- 更新被考查知识点的最近分、最高分、状态，并追加 Detailed History。
- 不把“安排了阅读”记为“已掌握”；通过 8 个抽测点也不等于掌握全部 45 点。

后续可以直接这样启动：

> 按三天计划执行 CP2，只考 D3.3 和 D4.4。每个知识点一个主问题、最多一次必要追问；结束后评分，给出对应源码补读位置，并记录真实进度。

# 六、3 天结束后的验收标准

| 能力 | 可检验的标准 |
|---|---|
| 架构模型 | 10 分钟内闭卷画出入口、Ingestion、Query、libs、MCP、Trace/Evaluation 和主要存储 |
| 摄取调用链 | 5 分钟讲完路径、数据类型变化、去重、编码、存储 ID 对齐及失败边界 |
| 查询调用链 | 5 分钟讲完预处理、Dense、BM25、正文回查、RRF、Rerank；区分 CLI 与 MCP 返回 |
| 抽象与扩展 | 用一个具体 Provider 说明 Base、Factory、implementation、调用者的关系 |
| 核心算法 | 手算一组 RRF、一组 Hit Rate/MRR，解释 BM25 和重排的适用问题 |
| MCP | 画出 tools/list、tools/call；解释 stdout/stderr 和阻塞调用的处理 |
| 配置驱动 | 说明 Provider、chunk 参数、四类 top-k、Vision/Evaluation 开关的实际读取位置 |
| 可观测性与评估 | 根据 Trace 提出排障步骤；识别禁用评估、无检索、指标不支持、接口不匹配等情况 |
| 工程边界 | 清楚说明“支持什么、没有保证什么”，尤其是跨存储事务、更新清理、最终答案生成 |
| 面试表达 | 用 8–10 分钟介绍项目，每个核心设计至少说明一个 Why 和一个 Trade-off |

最终保留四份学习产物：**一张架构图、一张带数据类型的调用链图、一份配置与故障边界表、一段项目介绍录音**。第一轮是否完成，以这些产物和阶段验收为准。