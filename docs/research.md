# AI 记忆系统 — 项目调研(第一阶段)

> 状态:第一阶段交付物之一(项目调研)
> 关联仓库:<https://github.com/fantasylincen/ai-memory>
> 说明:本文只做技术调研与结论,需求与行为定义见同目录 `PRD.md`。

## 1. 调研目标与范围

本次调研服务于一个近期目标:**让本地 AI Agent 在 128KB 上下文的限制下持续不间断地工作**,不因上下文丢失而中断,不忘记核心开发工作与当前正在处理的事务。

调研需要回答四个问题:

| 编号 | 调研问题 | 影响 |
|---|---|---|
| Q1 | 记忆系统以什么形态接入 Trae Agent? | 决定整体架构与用户操作方式 |
| Q2 | 用什么存储与检索? | 决定召回能力与部署成本 |
| Q3 | 用什么语义模型? | 决定中英双语/代码场景的检索质量 |
| Q4 | 现有开源方案能借鉴什么、差异在哪? | 避免重复造轮子、明确自身定位 |

范围限定:仅近期目标(MVP),不考虑多用户、云端多租户、跨团队协作等远期场景。

## 2. 问题定义

本地 Agent 在长周期开发任务中的三个可观察失效模式:

1. **上下文溢出中断**:上下文达到上限后被裁剪/压缩,早期已确认的目标与约束丢失。
2. **进行中事务遗忘**:重新进入会话时,Agent 不记得"上次做到哪一步、下一步是什么"。
3. **终极目标漂移**:在多轮迭代后逐渐偏离项目最初的终态,产出与目标不一致。

这三者本质是同一个问题:**Agent 缺少一个跨会话、可检索、可压缩的持久记忆层**,并且缺少对"任务结构"的显式建模。

## 3. Q1:接入形态调研 — 记忆系统如何接入 Agent

### 3.1 候选形态对比

| 形态 | 机制 | 优点 | 局限 |
|---|---|---|---|
| MCP Server(stdio) | 以本地进程形式注册,Agent 通过工具调用读写记忆 | Trae 原生支持;工具契约清晰;可移植到其他支持 MCP 的客户端 | 需本地安装运行时(npx/uvx 或 Python/Node) |
| 项目 Rules / AGENTS.md | 用文件约定 + 脚本读写仓库内记忆 | 零依赖、人类可读 | 无工具契约,靠 Agent "自觉"读取,检索能力弱 |
| 插件/自定义脚本 | 独立 CLI 或脚本 | 灵活 | 需 Agent 手动执行命令,交互重 |

### 3.2 Trae MCP 关键约束(已核实)

Trae 支持通过 JSON 配置本地(stdio)与远程(HTTP)MCP Server,并支持项目级 MCP:

- 项目级 MCP 配置放在项目根目录 `.trae/mcp.json`,启用后 Trae 在需要时自动加载。
- stdio 类型字段:`command`(必填,不能含空格)、`args`(字符串数组)、`env`(字符串键值)。
- 支持变量引用 `${workspaceFolder}`,启动时替换为项目真实路径。
- 支持超时配置:`START_MCP_TIMEOUT_MS`、`RUN_MCP_TIMEOUT_MS`(经 `env` 传入)。

### 3.3 结论

**选择 MCP Server(stdio,项目级)。** 理由:最贴合 Trae Agent 的调用模型,能够把"记忆读写"变成显式、可审计、可测试的工具调用,而不是依赖 Agent 的记忆自觉;同时 stdio 本地进程便于本地运行语义模型,契合隐私要求。

> 备注:第一种形态是推荐且可在 Trae 复用的;`${workspaceFolder}` 与超时项将在需求文档的 MCP 工具契约中作为运行约束固定下来。

## 4. Q2:存储与检索调研 — Qdrant

### 4.1 Qdrant 能力

- **原生混合检索(Dense + Sparse)**:单次查询可融合关键词与向量,支持 BM25、SPLADE++、miniCOIL 等稀疏表示。
- **高效过滤**:支持 `keyword` / `integer` / `float` 等 payload 索引,可在单阶段内完成"向量检索 + 结构化过滤"。
- **Multivector**:支持一个对象多个向量(ColBERT 式晚交互),提升相关性表达力。
- **新版本(1.19.x)**:keyword 前缀匹配、slice 过滤等,并在过滤与可视化上增强。

### 4.2 与候选对比

| 方案 | 混合检索 | 结构化过滤 | 本地部署 | 结论 |
|---|---|---|---|---|
| Qdrant | 原生支持 | 强(payload 索引) | 支持(Docker/本地) | **选定** |
| SQLite(+FTS5) | 关键词 + 语义需自建 | 一般 | 极轻 | 备份方案 |
| pgvector | 需外挂全文检索 | 强(借 SQL) | 较重 | 备选 |
| Chroma | 弱 | 一般 | 轻 | 备选 |

### 4.3 结论

**选择 Qdrant。** 需求同时要求"语义召回"与"按任务状态/父子关系过滤",Qdrant 的原生混合检索 + payload 过滤正好覆盖这两点,且可本地部署。这直接满足用户"主流语义模型 + Qdrant"的技术约束。

## 5. Q3:语义模型调研

### 5.1 候选对比

| 模型 | 特点 | 适用 | 局限 |
|---|---|---|---|
| BGE-M3(BAAI, MIT) | 568M,单模型同时产出 dense/sparse/ColBERT 多向量;100+ 语言;8192 token 上下文;HF 上下载量最高 | 自托管、中英双语、混合检索 | 需本地算力,首启下载权重 |
| Qwen3-Embedding-8B | MTEB 多语与代码任务领先(8B 约 75% 均值,多语 70.58 / 代码 80.68) | 纯中文占比高、追求质量上限 | 8B 较重,本地运行成本高 |
| 云端 Embedding API | 免本地算力 | 快速起步 | 需 Key、联网、费用与隐私顾虑 |

### 5.2 结论

**选择本地自托管 BGE-M3(默认)。**
理由:(1) 中英双语强,契合中文为主、夹带代码/英文标识符的开发场景;(2) 单模型即支持 dense + sparse,与 Qdrant 混合检索天然配套;(3) MIT 许可、可离线、数据不出本机,契合隐私要求;(4) 相比 8B 模型部署负担更轻,适合 MVP。

设计上保留 Embedding Provider 抽象,但不作为 MVP 必交付项(见 PRD 开放问题)。

## 6. Q4:现有开源方案调研(可借鉴点)

| 方案 | 核心机制 | 可借鉴 | 与我们的差异 |
|---|---|---|---|
| 官方 `memory` MCP 参考服务 | 本地知识图谱:实体/关系/观察 | 图结构持久化思路 | 无向量语义召回,无任务状态机 |
| `mcp-ai-memory` | 本地部署,多 LLM,多向量库(推荐 Qdrant),完整 CRUD | Qdrant 接入与 CRUD 设计 | 面向通用记忆,无"任务思维导图 + 状态" |
| Qdrant Memory System v3 | Qdrant + MCP,双向同步 | Qdrant 落地经验 | 无分层上下文装配与遗忘策略 |
| mnemostack | 向量 + BM25 + 时序 + RRF 融合 | **混合检索 + RRF 排序** | 无任务结构建模 |
| Dragon Brain | 知识图谱(FalkorDB) + 向量(Qdrant) | 图谱 + 向量结合 | 组件重,偏通用助手 |
| Trae Memory 系列(SQLite + FTS5 + 语义) | 轻量本地记忆 | 轻量部署参考 | 无思维导图与上下文预算控制 |

**可复用的共性做法**:Qdrant 作为向量后端 + MCP 暴露工具 + 混合检索 + RRF 融合。
**本项目的差异化**:把记忆组织成**任务思维导图(节点=任务、边=父子)**,并引入**按状态分层装配上下文**与**完成即折叠的选择性遗忘**——这正是"不忘记进行中事务、同时释放已完成细节"的核心。

## 7. 关键约束

| 约束 | 说明 | 来源 |
|---|---|---|
| 上下文 128KB | 单次装配进 Agent 上下文的记忆量受限,必须分层 + 预算裁剪 | 用户明确要求 |
| 本地运行 | 语义模型与向量库均本地,数据不出机 | 技术约束 + 隐私 |
| Trae Agent 为测试环境 | 最终以 Trae Agent 表现验证有效性 | 用户明确要求 |
| 近期目标优先 | 先做能跑的 MVP,不做远期规划 | 用户明确要求 |

## 8. 调研结论

1. 形态:**本地 stdio MCP Server**,项目级配置 `.trae/mcp.json`。
2. 存储:**Qdrant**(原生混合检索 + payload 过滤)。
3. 模型:**本地 BGE-M3**(dense + sparse,8192 token,中英双语)。
4. 核心创新:**任务思维导图建模 + 状态分层上下文装配 + 完成即折叠的选择性遗忘**。
5. 排序:**混合检索 + RRF 融合 + 结构化状态过滤**。

以上结论作为 `PRD.md` 的技术与产品前提。检索排序、遗忘阶梯、上下文预算等规则的具体定义见需求文档。

## 9. 来源

- Trae 官方文档 · 添加 MCP Server:<https://docs.trae.ai/ide/add-mcp-servers>
- Qdrant 官网(混合检索、Multivector):<https://qdrant.tech/>
- Qdrant 索引与 payload 索引:<https://qdrant.tech/documentation/manage-data/indexing/>
- Qdrant 1.19 发布说明(过滤增强):<https://qdrant.tech/blog/qdrant-1.19.x/>
- Qdrant Fundamentals(能力边界):<https://qdrant.tech/documentation/faq/qdrant-fundamentals/>
- Qwen3 Embedding 技术报告:<https://arxiv.org/pdf/2506.05176>
- 开放权重 Embedding 模型综述(2026):<https://presenc.ai/research/best-open-weight-embedding-models-2026>
- 中英双语 RAG 生产栈(2026):<https://www.syncsoft.ai/en/blog/bilingual-rag-production-stack-chinese-chuhai-multilingual-retrieval-2026>
- MCP 架构总览(stdio / Streamable HTTP):<https://modelcontextprotocol.io/docs/draft/learn/architecture>
- mcp-ai-memory(Qdrant 版 MCP 记忆服务):<https://lobehub.com/mcp/morsewayne-mcp-ai-memory>
- Qdrant Memory System v3:<https://lobehub.com/mcp/hailanlan0577-mcp-qdrant-memory>
- mnemostack(向量 + BM25 + RRF):<https://himcp.ai/server/mnemostack>
- Dragon Brain(知识图谱 + Qdrant):<https://lobehub.com/de/mcp/iikarus-dragon-brain>