
这里是在学习《Hello-Agent》教程时，记录的一些笔记。重点在于学习 `HelloAgents` 框架的实现。部分内容摘自教程，部分内容为个人理解，也有部分内容为大语言模型生成。

该框架源代码仓库：https://github.com/jjyaoao/HelloAgents

## 1. 整体架构设计

除Agent类外一切皆为Tools，Memory/RAG/RL/MCP统一抽象为工具，消除不必要的抽象层。

### 1.1 Message类

Message 类是 HelloAgents 框架中用于规范管理智能体与大语言模型之间交互信息的核心数据结构。它的作用是：确保用户指令、系统设定、AI 回复、工具结果这四类信息都以统一的格式在框架中流转和存储，同时能无缝对接 OpenAI API 的消息规范。

|类的属性|字段名|类型|
|---|---|---|
|消息内容|content|str|
|角色|role|MessageRole|
|时间戳|timestamp|datetime|
|元数据|metadata|Optional[Dict[str, Any]]|

其中，`MessageRole` 是一个枚举类型，定义了消息的四种角色：`user`，`assistant`，`system`，`tool`。

### 1.2 Config类

这是配置类，从配置文件中读取配置信息（避免硬编码），并设置默认值。

类的属性分成三组：LLM配置、系统配置、其他配置。
LLM配置：模型名称、提供商、温度、最大词元数。
系统配置：是否开启调试模式、日志级别。
其他配置：最大历史记录长度。

|类的属性|字段名|类型|
|---|---|---|
|模型名称|default_model|str|
|提供商|default_provider|str|
|温度|temperature|float|
|最大词元数|max_tokens|Optional[int]|
|是否开启调试模式|debug|bool|
|日志级别|log_level|str|
|最大历史记录长度|max_history_length|int|

### 1.3 Agent抽象基类

它通过继承 ABC 被定义为一个不能直接实例化的抽象类。其构造函数 `__init__` 清晰地定义了 Agent 的核心依赖：名称、LLM 实例、系统提示词和配置。最重要的部分是使用 @abstractmethod 装饰的 run 方法，它强制所有子类必须实现此方法，从而保证了所有智能体都有统一的执行入口。此外，基类还提供了通用的历史记录管理方法，这些方法与 Message 类协同工作，体现了组件间的联系。

|类的属性|字段名|类型|
|---|---|---|
|Agent名称|name|str|
|LLM客户端|llm|HelloAgentsLLM|
|系统提示词|system_prompt|Optional[str]|
|配置|config|Optional[Config]|
|历史消息|_history|list[Message]|

目前有这几个实现类：`SimpleAgent`、`ReActAgent`、`ReflexionAgent`、`PlanAndSolveAgent`、`FunctionCallAgent`。

### 1.4 HelloAgentsLLM类

这个类是 HelloAgents 框架中用于与 LLM（大型语言模型）进行交互的接口。它封装了 OpenAI API 的调用，并提供了多种方法来发送消息并获取回复。这个类的主要目的是简化与 LLM 的交互，使得用户可以更方便地使用 LLM 来实现各种功能。

|类的属性|字段名|类型|
|---|---|---|
|模型名称|model|Optional[str]|
|API密钥|api_key|Optional[str]|
|请求URL|base_url|Optional[str]|
|提供商|provider|Optional[SUPPORTED_PROVIDERS]|
|温度|temperature|float|
|最大词元数|max_tokens|Optional[int]|
|超时时间|timeout|Optional[int]|
|客户端|_client|OpenAI|

## 2. 工具系统

### 2.1 工具抽象基类Tool

这是所有工具类的抽象基类，所有工具类都需要实现这个类。它提供了抽象的 `run` 方法和 `get_parameters` 方法，分别用于执行工具和获取工具的参数。

> 注意：这个类里有一个 `to_openai_schema` 方法，用于将工具的参数转换为 OpenAI API 的 schema 格式。为了让 LLM 不仅仅知道有什么工具存在，还需要知道每个工具需要什么参数，这个方法就是用来描述工具参数的。
> 正常情况下，这个方法解析当前工具的参数 list[ToolParameters]，返回 OpenAI 原生 function calling 格式，会随着系统提示词一并被发送给 LLM。

### 2.2 工具参数类ToolParameters

ToolParameter 是 HelloAgents 工具系统中用来精确描述一个工具"需要什么参数"​的数据结构。它告诉调用者（Agent 或 LLM）：这个工具接受几个参数、每个参数叫什么名字、是什么类型、是否必填、有没有默认值。

### 2.3 工具注册类ToolRegistry

ToolRegistry 是 HelloAgents 工具系统中用来管理所有工具的注册表。它提供了注册工具、获取工具、获取工具参数、获取工具描述、执行工具、注销工具等功能。

这里的工具注册有两种方式：一种是注册 Tool 的实现类对象，另一种是注册一个函数作为工具。

> 编写工具：编写一个类，继承 `Tool` 类，并实现其中的抽象方法。
> 使用工具：获取 `ToolRegistry` 实例，调用 `register_tool` 方法注册工具，调用 `execute_tool` 方法执行工具。

工具系统里还提供了**链式执行**和**异步执行**的功能。

## 3. 记忆与检索系统

设计思想：`memory_tool` 与 `rag_tool` 作为两个独立工具，解耦记忆维护与知识检索。

### 3.1 记忆系统

#### 3.1.1 记忆系统四层架构

```
HelloAgents记忆系统
├── 基础设施层 (Infrastructure Layer)
│   ├── MemoryManager - 记忆管理器（统一调度和协调）
│   ├── MemoryItem - 记忆数据结构（标准化记忆项）
│   ├── MemoryConfig - 配置管理（系统参数设置）
│   └── BaseMemory - 记忆基类（通用接口定义）
├── 记忆类型层 (Memory Types Layer)
│   ├── WorkingMemory - 工作记忆（临时信息，TTL管理）
│   ├── EpisodicMemory - 情景记忆（具体事件，时间序列）
│   ├── SemanticMemory - 语义记忆（抽象知识，图谱关系）
│   └── PerceptualMemory - 感知记忆（多模态数据）
├── 存储后端层 (Storage Backend Layer)
│   ├── QdrantVectorStore - 向量存储（高性能语义检索）
│   ├── Neo4jGraphStore - 图存储（知识图谱管理）
│   └── SQLiteDocumentStore - 文档存储（结构化持久化）
└── 嵌入服务层 (Embedding Service Layer)
    ├── DashScopeEmbedding - 通义千问嵌入（云端API）
    ├── LocalTransformerEmbedding - 本地嵌入（离线部署）
    └── TFIDFEmbedding - TFIDF嵌入（轻量级兜底）
```

基础设施层：记忆系统的地基，定义了所有记忆模块都需要的公共组件。
记忆类型层：记忆系统的核心，提供了不同类型的记忆实现。
存储后端层：真正存数据的地方，决定数据用什么形式、存在什么数据库里
嵌入服务层：记忆系统的辅助层，提供了不同类型的嵌入服务，如通义千问嵌入、本地嵌入等。

#### 3.1.2 项目结构

memory/base.py 和 memory/manager.py 定义了基础设施层。
memory/types/ 目录下定义了记忆类型层。
memory/storage/  目录下定义了存储后端层。
memory/embedding.py 定义了嵌入服务层。

#### 3.1.3 四种记忆类型实现

> 实现一种记忆类型，就是要解决如何存储、如何（高效）检索、如何更新、如何删除、如何统计等问题。

（1）工作记忆 WorkingMemory

纯内存存储 + TTL自动清理
默认容量50条，默认TTL 60分钟
混合检索：TF-IDF向量化（0.7权重）+ 关键词匹配（0.3权重）
⭐评分公式：(相似度 × 时间衰减) × (0.8 + 重要性 × 0.4)

（2）情景记忆 EpisodicMemory

SQLite + Qdrant混合存储架构
会话索引管理，结构化过滤 + 语义向量检索
⭐评分公式：(向量相似度 × 0.8 + 时间近因性 × 0.2) × (0.8 + 重要性 × 0.4)
设计逻辑：强调时间近因性，适合事件回溯

（3）语义记忆 SemanticMemory

Qdrant向量数据库 + Neo4j图数据库混合架构
自动提取实体和关系，构建知识图谱
混合检索：向量检索 + 图检索
⭐评分公式：(向量相似度 × 0.7 + 图相似度 × 0.3) × (0.8 + 重要性 × 0.4)
设计逻辑：图检索补充关系推理，发现概念间隐含关联
支持中英文NLP处理（spaCy模型）

（4）感知记忆 PerceptualMemory

模态分离存储策略（text/image/audio独立向量集合）
跨模态语义对齐（CLIP图像编码 / CLAP音频编码）
指数衰减时间模型：recency_score = exp(-0.1 × age_hours/24)，最低0.1
⭐评分公式：(向量相似度 × 0.8 + 时间近因性 × 0.2) × (0.8 + 重要性 × 0.4)

#### 3.1.4 MemoryTool

在 HelloAgents框架中，记忆系统也被定义成一个内置工具，在 tools/builtin/memory_tool.py 文件中定义。通过工具系统的统一的 execute 方法来调用记忆系统的功能。

### 3.2 检索系统

在 HelloAgents 框架中 RAG 的工作流程如下。
- 数据处理流程：处理和存储知识文档，采取工具Markitdown，设计思路是将传入的一切外部知识源统一转化为Markdown格式进行处理。
- 查询与生成流程：根据查询检索相关信息并生成回答。

#### 3.2.1 检索系统四层架构

```
HelloAgents RAG系统
├── 文档处理层 (Document Processing Layer)
│   ├── DocumentProcessor - 文档处理器（多格式解析）
│   ├── Document - 文档对象（元数据管理）
│   └── Pipeline - RAG管道（端到端处理）
├── 嵌入表示层 (Embedding Layer)
│   └── 统一嵌入接口 - 复用记忆系统的嵌入服务
├── 向量存储层 (Vector Storage Layer)
│   └── QdrantVectorStore - 向量数据库（命名空间隔离）
└── 智能问答层 (Intelligent Q&A Layer)
    ├── 多策略检索 - 向量检索 + MQE + HyDE
    ├── 上下文构建 - 智能片段合并与截断
    └── LLM增强生成 - 基于上下文的准确问答
```

#### 3.2.2 数据载入

多模态文档载入：MarkItDown统一转换，PDF增强处理，多格式支持。

针对Markdown格式的智能分块策略：
Markdown标题层次解析 → 段落语义分割 → Token计算分块 → 重叠策略优化
中英文混合Token估算（CJK字符1 token + 空白分词）

统一嵌入与向量存储：云端嵌入模型API / 本地嵌入模型 / TF-IDF算法（备选）

#### 3.2.3 高级检索策略

- 多查询扩展MQE：用LLM生成语义等价的多样化查询。
- 假设文档嵌入HyDE：用LLM生成假设性答案，再用答案检索真实文档。
- 统一扩展检索框架：MQE + HyDE + 向量检索，多策略融合。

## 4. 上下文工程

上下文工程是在推理阶段，策划与维护"最优信息集合（tokens）"的工程方法。从持续增长的候选信息中，甄别哪些内容应当进入有限的上下文窗口。

> 区分“上下文工程”与“提示词工程”
> 提示工程：关注如何编写与组织 LLM 指令
> 上下文工程：管理整个上下文状态（系统指令、工具、MCP、外部数据、消息历史）

### 4.1 上下文工程的重要性

随着上下文窗口 tokens 增加，模型准确回忆信息的能力反而下降。上下文是有限资源，具有边际收益递减特性。

### 4.2 有效的上下文

- 系统提示词：语言清晰、直白，信息层级把握在“刚刚好”的高度。不能过于空泛，也不能过于硬编码。
- 工具描述：职责单一、对错误鲁棒、入参无歧义。
- 给LLM提供的示例：精挑细选一组多样且典型的示例，直接对“期望行为”进行画像。好的示例胜过千言万语。

> 工程实践正在从“推理前一次性检索”逐步过渡到“及时（Just-in-time, JIT）上下文”。后者不再预先加载所有相关数据，而是维护轻量化引用（文件路径、存储查询、URL 等），在运行时通过工具动态加载所需数据。
> 混合策略：前置加载高价值上下文 + 允许按需自主探索

### 4.3 三大核心策略

(1) 压缩整合（Compaction）

- 定义：当对话接近上下文上限时，对其进行高保真总结，并用该摘要重启一个新的上下文窗口，以维持长程连贯性。
- 实践：让模型压缩并保留架构性决策、未解决缺陷、实现细节，丢弃重复的工具输出与噪声；新窗口携带压缩摘要 + 最近少量高相关工件（如“最近访问的若干文件”）。
- 调参建议：先优化召回（确保不遗漏关键信息），再优化精确度（剔除冗余内容）；一种安全的“轻触式”压缩是对“深历史中的工具调用与结果”进行清理。

(2) 结构化笔记（Structured note-taking）

- 定义：也称“智能体记忆”。智能体以固定频率将关键信息写入上下文外的持久化存储，在后续阶段按需拉回。
- 价值：以极低的上下文开销维持持久状态与依赖关系。例如维护 TODO 列表、项目 NOTES.md、关键结论/依赖/阻塞项的索引，跨数十次工具调用与多轮上下文重置仍能保持进度与一致性。
- 说明：在非编码场景中同样有效（如长期策略性任务、游戏/仿真中的目标管理与统计计数）。结合前面的 MemoryTool，可轻松实现文件式/向量式的外部记忆并在运行时检索。

(3) 子代理架构（Sub-agent architectures）

- 思想：由主代理负责高层规划与综合，多个专长子代理在“干净的上下文窗口”中各自深挖、调用工具并探索，最后仅回传凝练摘要（常见 1,000–2,000 tokens）。
- 好处：实现关注点分离。庞杂的搜索上下文留在子代理内部，主代理专注于整合与推理；适合需要并行探索的复杂研究/分析任务。
- 经验：公开的多智能体研究系统显示，该模式在复杂研究任务上相较单代理基线具有显著优势。

### 4.4 工程实践

#### 4.4.1 ContextBuilder

GSSC 流水线解析如下。

- Gather（多源信息汇集）

五大来源：系统指令 → 记忆检索 → RAG 检索 → 对话历史 → 自定义信息包
容错设计：每个源 try-except 包裹；优先级处理：系统指令始终保留

- Select（智能信息选择）

综合评分 = 相关性权重 × 相关性 + 新近性权重 × 新近性
相关性：Jaccard 相似度（可替换为向量相似度）
新近性：指数衰减模型（24h 内保持高分）
贪心选择：按分数降序填充至 token 上限

- Structure（结构化输出）

按类型分组 → 构建六分区模板
优势：可读性、可调试性、可扩展性

- Compress（兜底压缩）

分区压缩，保持结构完整性
至少保留 50 tokens，生产环境可用 LLM 摘要替代截断

#### 4.4.2 NoteTool结构化笔记

七个操作：create / read / update / search / list / summary / delete

笔记类型体系：task_state / conclusion / blocker / action / reference / general

可以与 ContextBuilder 集成使用。

#### 4.4.3 TerminalTool即时文件系统访问

TerminalTool 为智能体提供了安全的命令行执行能力，支持常用的文件系统和文本处理命令，同时通过多层安全机制确保系统安全。

四层安全机制：
- 命令白名单：仅允许只读命令（ls/cat/grep/find/awk 等）
- 工作目录限制（沙箱）：禁止访问工作目录外路径
- 超时控制：默认 30 秒
- 输出大小限制：防止内存溢出

## 5. 智能体通信协议

### 5.1 三种通信协议

(1) MCP（Model Context Protocol）​——智能体与工具的桥梁

提出方：Anthropic团队
设计哲学：上下文共享（不仅是RPC，更关注智能体与工具间的丰富上下文交换）

(2) A2A（Agent-to-Agent Protocol）​——智能体间的对话

提出方：Google团队
设计哲学：对等通信（每个智能体既是服务提供者也是消费者，去中心化）

(3) ANP（Agent Network Protocol）​——智能体网络的基础设施

提出方：开源社区
设计哲学：去中心化服务发现（大规模网络中动态发现和连接智能体）

> 协议选择原则：访问外部服务选MCP，多智能体协作选A2A，大规模生态系统选ANP
> 重要结论：当前协议处于发展早期，MCP生态相对成熟，推荐优先选择大公司背书的工具

### 5.2 工程实践

三层架构设计：

- 协议实现层：MCP基于FastMCP库、A2A基于a2a-sdk、ANP为轻量级自研实现
- 工具封装层：MCPTool/A2ATool/ANPTool统一继承BaseTool，提供一致run()方法
- 智能体集成层：所有智能体通过Tool System使用协议工具，屏蔽底层
